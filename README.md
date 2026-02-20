# ai-engineering-by-anilsoylu

A Claude Code plugin providing AI engineering best practices for building high-performance agentic applications. Focused on prompt caching, context management, and cost optimization patterns used by production systems like Claude Code and Manus.

## Available Skills

| Skill | Description | Key Topics |
|-------|-------------|------------|
| `prompt-caching` | Prompt caching optimization for agentic AI | Prefix matching, cache-safe patterns, compaction, monitoring |

## What You'll Learn

### Prompt Caching
- **Core principle**: How prefix matching works and why request layout order matters
- **System prompt ordering**: Static-first, dynamic-last for maximum cache hits
- **Tool stability**: Why mutating tool definitions destroys your cache
- **Subagent pattern**: Model switching without breaking the parent cache
- **Cache-safe compaction**: Summarizing old messages while preserving the prefix
- **Cache-safe forking**: Exploring multiple approaches with shared prefix
- **Auto-caching vs manual**: When to use `cache_control` breakpoints vs letting the API handle it
- **Monitoring**: Tracking cache hit rates and debugging cache misses

## Installation

### As a Claude Code Plugin

```bash
# Clone the repository
git clone https://github.com/anilsoylu/ai-engineering-by-anilsoylu.git \
  ~/.claude/plugins/local/ai-engineering-by-anilsoylu

# The plugin is auto-discovered by Claude Code from ~/.claude/plugins/local/
```

### Manual Setup

1. Create the plugin directory:
   ```bash
   mkdir -p ~/.claude/plugins/local/ai-engineering-by-anilsoylu
   ```
2. Copy all files into the directory
3. Restart Claude Code — the plugin will be auto-discovered

## Usage

The skill triggers automatically when you discuss relevant topics. Example phrases:

- "How does prompt caching work?"
- "Optimize my agent's cache hit rate"
- "Why is my cache miss rate high?"
- "How should I structure my system prompt for caching?"
- "What's the best way to handle tool definitions?"
- "How do I compact conversation context safely?"
- "Explain auto-caching vs manual cache control"
- "How much does caching save on API costs?"

## File Structure

```
ai-engineering-by-anilsoylu/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   └── prompt-caching/
│       ├── SKILL.md             # Main skill content (~240 lines)
│       └── references/
│           ├── caching-patterns.md    # Detailed patterns & pseudocode
│           └── auto-caching-api.md    # Auto-caching & pricing guide
├── LICENSE                      # MIT License
└── README.md                    # This file
```

## Skill Design

This plugin follows Claude Code's progressive disclosure pattern:

1. **SKILL.md** — Loaded into context when triggered. Contains actionable guidance, patterns, and anti-patterns. Optimized to be comprehensive yet concise.
2. **references/** — Detailed deep-dives linked from SKILL.md. Only loaded when the user needs more depth on a specific topic.

## Contributing

Contributions are welcome! To add a new skill:

1. Create a directory under `skills/` with your skill name
2. Add a `SKILL.md` with YAML frontmatter (`name`, `description`) and skill content
3. Optionally add a `references/` directory for detailed supplementary material
4. Submit a PR

## License

[MIT](LICENSE) — Anil Soylu
