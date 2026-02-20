# Auto-Caching API & Manual Cache Control

## How Auto-Caching Works

The Anthropic API automatically caches prompt prefixes that meet minimum token thresholds. No explicit `cache_control` breakpoints are required.

**Minimum token thresholds for auto-caching:**
- Claude Opus 4.6 / Sonnet 4.6: 1024 tokens
- Claude Haiku 4.5: 2048 tokens

When a request is sent, the system identifies the longest prefix that matches a previously cached request and serves those tokens from cache. The remaining tokens are processed normally.

**Cache TTL**: ~5 minutes, refreshed on each cache hit. In agentic loops with rapid back-and-forth turns, the cache stays warm indefinitely throughout the session.

## Manual Cache Control: `cache_control` Parameter

For explicit control, add `cache_control: {"type": "ephemeral"}` to content blocks:

```json
{
  "model": "claude-opus-4-6-20260220",
  "max_tokens": 1024,
  "system": [
    {
      "type": "text",
      "text": "You are a helpful AI assistant with deep knowledge...",
      "cache_control": {"type": "ephemeral"}
    }
  ],
  "messages": [...]
}
```

You can place up to **4 explicit cache breakpoints** per request. Each breakpoint tells the API: "cache everything up to and including this block."

**Where to place breakpoints (in order of stability):**
1. End of system prompt
2. End of tool definitions
3. End of long, reusable context (e.g., a large document)
4. Before the final user message (for multi-turn)

## Pricing: Why Caching Is the #1 Optimization

| Token Type | Cost Relative to Base Input |
|-----------|---------------------------|
| Cache write (first time) | 125% of base input price |
| Cache read (subsequent) | **10% of base input price** |
| Normal input (uncached) | 100% of base input price |

**Example — 10-turn agentic session with 50k token system prompt:**

Without caching:
```
10 turns × 50k tokens = 500k input tokens billed at full price
```

With caching:
```
Turn 1: 50k tokens cache write (125% cost) = 62.5k equivalent
Turns 2-10: 50k tokens × 9 cache reads (10% cost) = 45k equivalent
Total: 107.5k equivalent vs 500k — ~78% savings
```

In practice, savings are even higher because tool definitions and growing conversation history also benefit from prefix caching.

## Agentic Loop Optimization

In agentic applications like Claude Code and Manus, the prompt structure is:

```
[System Prompt] → [Tool Definitions] → [Conversation History] → [Latest Message]
```

Each turn appends new messages but the prefix remains identical. This means:

- **Turn 1**: Full cache write (one-time cost)
- **Turn 2+**: Entire prefix is a cache read
- **Turn N**: Only the new message tokens are uncached

**Cache hit rate is the #1 metric** for agentic cost optimization. Claude Code achieves >90% cache read rates in typical sessions, translating to ~90% cost reduction.

### Why Caching Is Critical for Agents

Agents make **many sequential API calls** — often 20-50+ per task. Without caching:
- Each call re-processes the entire context from scratch
- Cost scales linearly: O(turns × context_size)

With caching:
- Each call pays full price only for new tokens
- Cost scales much better: O(context_size + turns × new_tokens_per_turn)

This is the difference between an agent costing $5 per task vs $0.50 per task.

### Best Practices for Agentic Caching

1. **Never modify the system prompt mid-session** — this invalidates the entire prefix cache
2. **Keep tool definitions stable** — adding/removing tools breaks the cache
3. **Use subagents for model switching** — different models have separate cache pools
4. **Monitor cache_read_input_tokens** in API responses — this should be >80% of total input
5. **Trigger compaction before context limit** — summarize old messages while preserving the prefix

### Response Headers to Monitor

Every API response includes cache performance data:

```json
{
  "usage": {
    "input_tokens": 100,
    "cache_creation_input_tokens": 50000,
    "cache_read_input_tokens": 45000
  }
}
```

**Healthy agentic session (after turn 1):**
- `cache_read_input_tokens` >> `input_tokens`
- `cache_creation_input_tokens` ≈ 0 (only new content)

**Unhealthy session (cache broken):**
- `cache_creation_input_tokens` ≈ total context size (full rewrite every turn)
- `cache_read_input_tokens` ≈ 0

## Auto-Caching vs Manual: Decision Guide

**Use auto-caching (no `cache_control`)** when:
- Your prompts naturally exceed the minimum threshold (1024+ tokens)
- You're in an agentic loop with stable prefix
- You want zero-config caching

**Add manual `cache_control`** when:
- Your system prompt is short (<1024 tokens) but you want it cached
- You need guaranteed breakpoints at specific content boundaries
- You're injecting large documents mid-conversation and want them cached
- You need fine-grained control over exactly what gets cached
