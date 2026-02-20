# Prompt Caching Implementation Patterns

Detailed implementation patterns, pseudocode, and anti-pattern catalog for prompt caching in agentic AI applications.

## Request Layout Diagram

```
┌─────────────────────────────────────────────────────┐
│                   API Request                        │
├─────────────────────────────────────────────────────┤
│  1. System Prompt                                    │
│     ├── Core identity (static)                       │
│     ├── Tool instructions (static)                   │
│     ├── Code style rules (static)                    │
│     ├── Project context / CLAUDE.md (per-project)    │
│     └── Environment / git status (per-session)       │
│                                                      │
│  2. Tool Definitions                                 │
│     ├── Core tools (static)                          │
│     ├── MCP tools (static per session)               │
│     └── Deferred tools (appended on demand)          │
│                                                      │
│  3. Conversation Messages                            │
│     ├── [system-reminder] (injected context)         │
│     ├── User message 1                               │
│     ├── Assistant message 1 + tool calls             │
│     ├── Tool results 1                               │
│     ├── ...                                          │
│     └── User message N (latest)                      │
│                                                      │
│  ════════════════════════════════════════════════     │
│  ↑ CACHED PREFIX (stable across requests)            │
│  ↓ NEW TOKENS (only latest turn is uncached)         │
└─────────────────────────────────────────────────────┘
```

## Pattern 1: System Prompt Construction

### Correct ordering

```python
def build_system_prompt(config, project, session):
    parts = []

    # Layer 1: Static (identical across all sessions)
    parts.append(CORE_IDENTITY)           # "You are Claude Code..."
    parts.append(TOOL_USAGE_RULES)        # "Use Read instead of cat..."
    parts.append(CODE_STYLE_GUIDELINES)   # "kebab-case files..."

    # Layer 2: Per-project (identical within a project)
    parts.append(project.claude_md)       # CLAUDE.md contents
    parts.append(project.memory_files)    # Auto-loaded memory

    # Layer 3: Per-session (set once at start, stable within session)
    parts.append(session.environment)     # OS, shell, platform
    parts.append(session.git_status)      # Initial git status snapshot

    # Layer 4: Dynamic (changes occasionally — LAST)
    parts.append(session.loaded_skills)   # Skills activated this session

    return "\n\n".join(parts)
```

### Anti-pattern: Timestamp at top

```python
# BAD — invalidates entire cache every request
def build_system_prompt():
    parts = []
    parts.append(f"Current time: {datetime.now()}")  # WRONG: changes every request
    parts.append(CORE_IDENTITY)
    parts.append(TOOL_USAGE_RULES)
    return "\n\n".join(parts)
```

### Fix: Timestamp at end or in system reminder

```python
# GOOD — timestamp at end of system prompt
def build_system_prompt():
    parts = []
    parts.append(CORE_IDENTITY)
    parts.append(TOOL_USAGE_RULES)
    parts.append(f"Current date: {date.today()}")  # Only date, not time; at end
    return "\n\n".join(parts)

# BETTER — timestamp as system reminder between turns
def build_messages(conversation):
    messages = []
    for turn in conversation:
        messages.append(turn)
    # Inject current context as system reminder before latest turn
    messages.insert(-1, {
        "role": "system",
        "content": f"<system-reminder>Current time: {datetime.now()}</system-reminder>"
    })
    return messages
```

## Pattern 2: Tool Definitions Stability

### State machine approach

```python
# Define ALL tools upfront — never add or remove
TOOLS = [
    {"name": "read_file", "description": "..."},
    {"name": "write_file", "description": "..."},
    {"name": "execute_command", "description": "..."},
    {"name": "enter_plan_mode", "description": "..."},
    {"name": "exit_plan_mode", "description": "..."},
    {"name": "enter_worktree", "description": "..."},
]

# Validate at runtime, not by modifying tool list
def handle_tool_call(tool_name, args, state):
    if tool_name == "write_file" and state.mode == "plan":
        return {"error": "Cannot write files in plan mode. Use exit_plan_mode first."}
    if tool_name == "exit_plan_mode" and state.mode != "plan":
        return {"error": "Not in plan mode."}
    # Execute the tool normally
    return execute_tool(tool_name, args)
```

### Anti-pattern: Dynamic tool removal

```python
# BAD — removing tools invalidates cache
def get_tools(state):
    tools = [READ, WRITE, EXECUTE]
    if state.mode == "plan":
        tools.remove(WRITE)     # WRONG: changes tool set
        tools.remove(EXECUTE)   # WRONG: changes tool set
    return tools
```

## Pattern 3: Subagent Model Switching

```python
# Main loop runs on Opus with warm cache
def main_agent_loop(conversation):
    while True:
        response = call_api(
            model="claude-opus-4-6",
            system=SYSTEM_PROMPT,       # Cached
            tools=TOOLS,                 # Cached
            messages=conversation,       # Prefix cached
        )

        if needs_simple_task(response):
            # Spawn subagent on Haiku — separate cache pool
            result = run_subagent(
                model="claude-haiku-4-5",
                task=extract_task(response),
                # Minimal context — only what's needed
                context=extract_relevant_context(response),
            )
            # Inject result back into main conversation
            conversation.append(tool_result(result))
        else:
            conversation.append(response)
```

### Why subagents preserve cache

```
Main agent cache pool (Opus):
  Request 1: [System][Tools][msg1]                    → cache write
  Request 2: [System][Tools][msg1][msg2]              → cache READ + small write
  Request 3: [System][Tools][msg1][msg2][msg3]        → cache READ + small write
                                                         ↑ warm cache preserved

Subagent cache pool (Haiku):
  Request 1: [Mini system][Few tools][task]           → separate, small cache write
  (completes and returns result)

Main agent continues with warm cache:
  Request 4: [System][Tools][msg1][msg2][msg3][result] → cache READ + small write
```

## Pattern 4: Cache-Safe Compaction

```python
def compact_conversation(system_prompt, tools, messages):
    """
    Compact conversation while preserving the system prompt + tools cache.
    """
    total_tokens = count_tokens(system_prompt + tools + messages)

    if total_tokens < COMPACTION_TRIGGER:
        return messages  # No compaction needed

    # Split messages into old (to summarize) and recent (to preserve)
    split_point = len(messages) - PRESERVE_RECENT_COUNT
    old_messages = messages[:split_point]
    recent_messages = messages[split_point:]

    # Summarize old messages using a SEPARATE API call (subagent)
    summary = summarize_with_subagent(old_messages)

    # Build compacted conversation
    # CRITICAL: system_prompt and tools are UNCHANGED
    compacted = [
        {"role": "user", "content": f"<conversation-summary>{summary}</conversation-summary>"},
        {"role": "assistant", "content": "I understand the context from the summary. Continuing..."},
        *recent_messages,
    ]

    return compacted

# Configuration
CONTEXT_WINDOW = 200_000
COMPACTION_TRIGGER = 160_000    # 80% — start compacting before it's too late
COMPACTION_TARGET = 100_000     # 50% — leave room for growth
PRESERVE_RECENT_COUNT = 25      # Keep last 25 messages verbatim
```

### Compaction flow diagram

```
Before (approaching limit):
  [System: 10k] [Tools: 5k] [Messages: 145k]  = 160k tokens
                              ├── msg 1-80 (old, 120k)
                              └── msg 81-100 (recent, 25k)

Compaction step (separate subagent call):
  Summarize msg 1-80 → summary (3k tokens)

After compaction:
  [System: 10k] [Tools: 5k] [Summary: 3k] [msg 81-100: 25k] = 43k tokens
   ↑ SAME         ↑ SAME
   Cache prefix preserved!
```

## Pattern 5: Deferred Tool Loading

```python
# Initial tool set (small, stable)
CORE_TOOLS = [
    {"name": "read_file", ...},
    {"name": "write_file", ...},
    {"name": "bash", ...},
    {"name": "tool_search", "description": "Search for additional tools by capability"},
]

# When model calls tool_search
def handle_tool_search(query):
    matching_tools = search_mcp_tools(query)
    # Return descriptions as TEXT, not as tool definitions
    return {
        "result": format_tool_descriptions(matching_tools),
        "hint": "You can now use these tools by calling them directly."
    }

# On next request, append discovered tools at END of tool list
def get_tools(session):
    tools = list(CORE_TOOLS)  # Always start with same core tools
    # Append discovered tools at the end — preserves prefix cache
    tools.extend(session.discovered_tools)
    return tools
```

## Anti-Pattern Catalog

### 1. Shuffled tool order

```python
# BAD — random order invalidates cache
import random
tools = list(ALL_TOOLS)
random.shuffle(tools)  # NEVER DO THIS
```

**Fix**: Always use a deterministic tool order. Sort alphabetically or use a fixed list.

### 2. Conditional system prompt sections

```python
# BAD — system prompt varies based on state
def build_system(state):
    prompt = CORE
    if state.has_errors:
        prompt += ERROR_HANDLING_SECTION  # Changes prefix length
    return prompt
```

**Fix**: Always include all sections. Use placeholders or empty sections if needed.

### 3. Tool description updates

```python
# BAD — updating tool descriptions mid-session
def update_tool_help(tool, new_description):
    tool["description"] = new_description  # Invalidates cache
```

**Fix**: Tool descriptions should be immutable for the session duration.

### 4. Mid-session model switching

```python
# BAD — switching model in the same conversation
response = call_api(model="opus", messages=conv)     # Opus cache
response = call_api(model="haiku", messages=conv)    # Haiku cache (Opus cache wasted)
response = call_api(model="opus", messages=conv)     # Opus cache COLD again
```

**Fix**: Use subagents for model switching. Main agent stays on one model.

### 5. System prompt rebuilding

```python
# BAD — rebuilding system prompt from scratch each request
def get_system():
    files = glob("*.md")
    content = "\n".join(read(f) for f in files)  # File order may vary!
    return CORE + content
```

**Fix**: Build system prompt once at session start. Cache the result. Only append via system reminders.

## Cache Performance Monitoring

### Logging template

```python
def log_cache_metrics(response):
    usage = response.usage
    total_input = usage.input_tokens
    cache_read = usage.cache_read_input_tokens
    cache_write = usage.cache_creation_input_tokens
    uncached = total_input - cache_read - cache_write

    cache_read_pct = (cache_read / total_input * 100) if total_input > 0 else 0
    cache_write_pct = (cache_write / total_input * 100) if total_input > 0 else 0

    log.info(f"Cache: {cache_read_pct:.1f}% read, {cache_write_pct:.1f}% write, "
             f"{uncached} uncached tokens")

    if cache_read_pct < 80:
        log.warning(f"Low cache hit rate: {cache_read_pct:.1f}%")
        # Trigger investigation
```

### Alerting thresholds

```python
CACHE_THRESHOLDS = {
    "cache_read_min": 0.80,      # Alert if < 80% cache reads
    "cache_write_max": 0.20,     # Alert if > 20% cache writes
    "uncached_max": 0.10,        # Alert if > 10% uncached
    "cost_per_turn_max": 0.05,   # Alert if > $0.05 per turn (after warm-up)
}
```

## "Is It Cache-Safe?" Checklist

Before making any change to your agentic system, run through this checklist:

- [ ] **System prompt**: Am I modifying content before existing cached content?
- [ ] **Tool definitions**: Am I adding, removing, or reordering tools?
- [ ] **Tool descriptions**: Am I changing any tool's name, description, or parameters?
- [ ] **Model**: Am I switching the model for the same conversation?
- [ ] **Message history**: Am I modifying (not appending) existing messages?
- [ ] **Compaction**: Am I preserving the system prompt + tools prefix exactly?

If ANY answer is "yes", your cache will be invalidated. Find an alternative approach:

| Instead of... | Do this... |
|---------------|------------|
| Modify system prompt | Use system reminder messages |
| Remove a tool | Validate at runtime, return error |
| Add a tool | Append at end of tool list |
| Switch models | Use a subagent |
| Rebuild system prompt | Build once, cache the string |
| Reorder tools | Use deterministic ordering |

## Cost Impact Reference

For a typical agentic session with ~50 turns:

| Approach | Cache Hit Rate | Relative Cost |
|----------|---------------|---------------|
| No caching optimization | ~0% | 1.0x (baseline) |
| Basic prefix caching | ~60% | 0.45x |
| Optimized ordering | ~85% | 0.20x |
| Full optimization (all patterns) | ~95% | 0.10x |

The difference between "no optimization" and "full optimization" is roughly **10x cost reduction** for the same workload.
