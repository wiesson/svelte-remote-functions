# SvelteKit Remote Functions - Claude Code Skill

A comprehensive Claude Code skill for building type-safe SvelteKit applications using remote functions. This skill provides AI assistants with patterns, best practices, and complete examples for implementing client-server communication in SvelteKit.

## What This Skill Provides

This skill helps AI assistants write correct code for:

- **Type-safe data fetching** - Query patterns with full TypeScript support
- **Progressive enhancement forms** - Form submissions that work without JavaScript
- **Imperative server actions** - Command patterns for event handlers
- **Optimistic updates** - Instant UI feedback with automatic error recovery
- **Query invalidation** - Efficient data refresh and cache management
- **N+1 prevention** - Batch query patterns
- **Validation** - Zod schema integration and validation error handling

## Installation

### Option 1: Direct Clone

```bash
cd ~/.claude/skills/
git clone https://github.com/wiesson/svelte-remote-functions
```

### Option 2: Manual Installation

1. Create the directory:
   ```bash
   mkdir -p ~/.claude/skills/svelte-remote-functions
   ```

2. Download the skill files from this repository

3. Place them in `~/.claude/skills/svelte-remote-functions/`

### Verify Installation

The skill should appear in Claude Code's available skills. The skill will automatically activate for SvelteKit projects.

## Skill Structure

```
svelte-remote-functions/
├── SKILL.md                    # Main entry point for AI
├── README.md                   # This file (for humans)
└── references/
    ├── quick-reference.md      # Syntax and common patterns
    ├── query.md               # Complete query function documentation
    ├── form.md                # Form functions with progressive enhancement
    ├── command.md             # Imperative command functions
    └── invalidation.md        # Data refresh and optimistic updates
```

## What Makes This Skill Effective

### Optimized for AI Consumption

- **Right-sized**: Main file is ~350 lines, reference files are 200-300 lines each
- **Pattern-focused**: Shows complete server + client integration patterns
- **Scannable structure**: Clear headers and code blocks
- **No fluff**: Every section serves a practical purpose

### Boosts Code Correctness

1. **Complete patterns** - Not just syntax, but full working examples
2. **Common pitfalls highlighted** - "Commands cannot be called during render"
3. **Best practices front-and-center** - Single-flight mutations, validation requirements
4. **Real workflow examples** - Create → Redirect + Refresh, Delete → Optimistic Remove
5. **Validation emphasis** - Every example uses Zod properly

## Coverage

### Core Patterns

- ✅ Query functions (read data)
- ✅ Form functions (write with progressive enhancement)
- ✅ Command functions (imperative actions)
- ✅ Prerender functions (static data)

### Advanced Features

- ✅ Batch queries (N+1 prevention)
- ✅ Isolated form instances (form.for(id))
- ✅ Sensitive field protection (_password pattern)
- ✅ Custom validation error handling
- ✅ Optimistic updates
- ✅ Single-flight mutations
- ✅ Request context access

### Validation

- ✅ Zod integration (primary)
- ✅ Standard Schema support (Valibot, Arktype)
- ✅ Complex nested schemas
- ✅ Optional and default values

## For AI Assistants

This skill is designed to be consumed by Claude Code. The main content is in `SKILL.md`, which provides:

1. **When to Use** guidance
2. **Quick Start Patterns** for immediate implementation
3. **Common Workflows** for real-world scenarios
4. **Best Practices** for decision-making
5. **References** to detailed documentation

## Requirements

This skill assumes:
- SvelteKit project
- TypeScript (recommended)
- Zod for validation (recommended)
- Remote functions feature enabled in svelte.config.js

## Contributing

Contributions welcome! Please ensure:

1. Examples use Zod for validation
2. No mentions of "experimental" status
3. Patterns are complete (server + client)
4. Code is practical and actionable

## License

MIT License - See LICENSE file for details

## Acknowledgments

Based on official SvelteKit remote functions documentation from https://svelte.dev/docs/kit/remote-functions/
