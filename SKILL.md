---
name: prompt-caching
description: >
  Guide for optimizing prompt caching in agentic AI applications.
  Use when building Claude agents, optimizing API costs, designing
  system prompts, managing tool definitions, implementing compaction,
  or troubleshooting cache miss rates. Triggers: "prompt caching",
  "cache hit rate", "reduce API costs", "system prompt ordering",
  "tool definitions caching", "compaction", "context window management",
  "agent cost optimization", "prefix caching", "cache-safe forking",
  "agentic loop cost", "token cost reduction", "cache miss debugging".
---

# Prompt Caching for Agentic AI Applications

Prompt caching is the single most impactful optimization for agentic AI systems. In Claude Code, prompt caching reduced costs by ~90% by ensuring the vast majority of tokens are cache reads rather than cache writes or uncached input.

## Core Principle: Prefix Matching

Prompt caching works on **exact prefix matching**. The API caches the longest prefix of your request that matches a previous request. Any change at position N invalidates the cache for everything after position N.

**Request layout (order matters):**
```
[System Prompt] → [Tool Definitions] → [System Reminders] → [Conversation Messages]
```

The system prompt and tool definitions form the **stable prefix**. Conversation messages are the **dynamic suffix** that grows with each turn. This layout maximizes cache hits because the stable prefix is identical across every request in a session.

> **Key insight**: You are NOT paying for the full context on every turn. With caching, you pay full price once (cache write), then ~10% for every subsequent read. The longer your stable prefix, the more you save.

## 1. System Prompt Ordering: Static First, Dynamic Last

Structure your system prompt with the most stable content at the top:

```
1. Core identity and capabilities (never changes)
2. Tool usage instructions (rarely changes)
3. Code style guidelines (rarely changes)
4. Project-specific context from CLAUDE.md (changes per project)
5. Environment info, git status (changes per session)
6. Memory files, auto-loaded skills (changes per session)
```

**Anti-pattern**: Putting timestamps, random session IDs, or frequently changing data at the top of the system prompt. This invalidates the entire cache on every request.

**Pattern**: If you must include dynamic data (current date, git status), append it at the very end of the system prompt or use system reminder messages between conversation turns.

## 2. System Messages for Mid-Conversation Updates

When you need to inject dynamic context mid-conversation (new file contents, updated state, skill content), use **system-type messages** inserted between conversation turns rather than modifying the system prompt.

```
System Prompt (cached) → Tools (cached) → [turn 1] → <system-reminder> → [turn 2] → ...
```

This preserves the stable prefix cache while still delivering fresh context. The system reminder only adds new tokens at the end of the message sequence.

**Use cases:**
- Injecting skill content when a skill is triggered
- Updating git status or environment state
- Adding hook feedback or validation results
- Loading file contents referenced by the user

## 3. Model Switching: The Subagent Pattern

Switching models mid-session (e.g., from Opus to Haiku for a simple task) destroys your cache because different models have separate cache pools.

**Anti-pattern**: Switching the primary model mid-conversation.

**Pattern**: Use **subagents** — spawn a separate, short-lived agent on the cheaper model. The subagent has its own conversation context (and its own cache), while the parent agent's cache remains intact.

```
Main agent (Opus, cached context preserved)
  └── Subagent (Haiku, fresh context, cheap)
        └── Returns result to main agent
```

Benefits:
- Main agent cache stays warm
- Subagent context is minimal (only the specific task)
- Subagent results are summarized back, keeping main context lean

## 4. Tool Stability: Never Mutate the Tool Set

Tool definitions are part of the cached prefix. Adding, removing, or reordering tools mid-session invalidates the cache for the entire tool block and everything after it.

**Anti-pattern**: Dynamically adding/removing tools based on conversation state.

**Pattern**: Define ALL tools upfront. Use **state transition tools** to control which actions are valid:

```
# Instead of removing "execute" tool when not in execute mode:
# Define a "request_execution_permission" tool that transitions state

Tools (always present):
  - read_file
  - write_file
  - request_execution_permission  ← gates access
  - execute_command               ← always defined, validated at runtime
```

Runtime validation happens in your application logic, not in the tool definitions sent to the API.

## 5. Plan Mode Pattern: Tools as State Machines

The EnterPlanMode / ExitPlanMode pattern demonstrates cache-safe state transitions:

```
Tools defined (constant):
  - EnterPlanMode    (available when: not in plan mode)
  - ExitPlanMode     (available when: in plan mode)
  - Read, Glob, Grep (available in both modes)
  - Edit, Write      (available when: not in plan mode)
```

All tools are always present in the API request. The application layer enforces which tools are valid based on current state. When the model calls a tool that's not valid in the current state, the application returns an error message — it does NOT remove the tool from the next request.

## 6. Tool Search / Deferred Loading

For large tool sets (e.g., many MCP servers), loading all tools upfront can bloat the prefix. Use a **lightweight stub + deferred loading** pattern:

```
Initial tool set:
  - core_tools (read, write, search, etc.)
  - tool_search(query: string)  ← meta-tool

When model calls tool_search("database"):
  → System finds matching MCP tools
  → Returns tool descriptions as TEXT in the tool result
  → Model uses the discovered tool on the next turn
```

The key insight: tool *descriptions* returned as text in a tool result don't affect the cached prefix. Only the formal tool *definitions* in the API request affect caching.

For tools that support `defer_loading: true`, the tool is not included in the initial request but can be loaded on demand without invalidating the prefix of other tools — because it's appended at the end of the tool list.

## 7. Cache-Safe Compaction

When conversation context approaches the context window limit, you must compact (summarize) older messages. Naive compaction destroys the cache. Cache-safe compaction preserves it.

**Cache-safe compaction flow:**

```
Before compaction:
  [System Prompt] [Tools] [msg1] [msg2] ... [msg50]
                   ↑ cached prefix

After compaction (fork the conversation):
  [System Prompt] [Tools] [summary of msg1..msg45] [msg46] ... [msg50]
                   ↑ same cached prefix preserved!
```

**Critical rules:**
1. **Never modify the system prompt or tools during compaction** — this preserves the prefix cache
2. **Keep recent messages verbatim** — the model needs exact recent context for coherent continuation
3. **Summarize older messages** — replace early messages with a concise summary
4. **Use a compaction buffer** — trigger compaction before hitting the limit, not at the limit. Leave room for the summary + a few more turns

**Compaction buffer sizing:**
```
context_window = 200k tokens
compaction_trigger = 160k tokens (80%)
compaction_target = 100k tokens (50%)
preserved_recent = last 20-30 messages
```

## 8. Cache-Safe Forking

When you need to explore multiple approaches (e.g., trying different fixes), fork the conversation:

```
Main context: [System] [Tools] [msg1..msg20]

Fork A: [System] [Tools] [msg1..msg20] [try approach A]
Fork B: [System] [Tools] [msg1..msg20] [try approach B]
```

Both forks share the same prefix cache. This is dramatically cheaper than starting fresh conversations for each approach.

## 9. Monitoring and Debugging Cache Performance

Track these metrics for every API request:

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| Cache read rate | >90% of input tokens | <80% |
| Cache write rate | <10% of input tokens | >20% |
| Uncached tokens | <5% of input tokens | >10% |

**Common cache miss causes:**
1. Dynamic content at the start of system prompt (timestamps, random IDs)
2. Tool definitions changed between requests
3. Model switched mid-session
4. System prompt modified mid-session
5. Tool order shuffled between requests

**Debugging steps:**
1. Compare the system prompt between two consecutive requests — any diff?
2. Compare tool definitions — any added/removed/reordered?
3. Check if model changed between requests
4. Look for any prefix modification

## 10. Auto-Caching vs Manual Cache Control

The Anthropic API supports **automatic caching** for prompts ≥1024 tokens (2048 for Claude Haiku 4.5). No explicit cache breakpoints needed — the system automatically caches the longest matching prefix.

**Manual cache control** uses the `cache_control` parameter for explicit breakpoints:

```json
{
  "system": [
    {
      "type": "text",
      "text": "Your system prompt...",
      "cache_control": {"type": "ephemeral"}
    }
  ]
}
```

**When to use manual vs auto:**

| Scenario | Recommendation |
|----------|---------------|
| Agentic loops (Claude Code, Manus) | Auto-caching sufficient — prefix grows naturally |
| Short prompts (<1024 tokens) | Manual `cache_control` to force caching |
| Multi-turn with stable tools | Auto-caching handles this well |
| Critical breakpoints you must guarantee | Manual `cache_control` for precision |

**Cost impact**: Cached token reads cost **~10% of base input price**. In agentic loops where the same prefix is sent dozens of times, this reduces costs by up to **90%**. Cache hit rate is the #1 cost metric to track.

> *For detailed auto-caching implementation, pricing breakdown, and agentic loop optimization, see `references/auto-caching-api.md`.*

---

## Quick Reference: Is It Cache-Safe?

| Action | Cache-Safe? | Alternative |
|--------|-------------|-------------|
| Add dynamic data to system prompt top | No | Append at end or use system reminders |
| Remove a tool mid-session | No | Keep tool, validate at runtime |
| Add a new tool mid-session | Partial | Append at end of tool list |
| Switch model mid-conversation | No | Use subagent on different model |
| Reorder tools between requests | No | Keep consistent tool ordering |
| Modify system prompt | No | Use system reminder messages |
| Summarize old messages | Yes | Fork with same prefix |
| Add new user/assistant turns | Yes | Normal conversation flow |
| Inject system reminder between turns | Yes | Appended after cached prefix |

---

*For detailed implementation patterns, pseudocode examples, and anti-pattern catalog, see `references/caching-patterns.md`.*
