# Ollie Shop Skills

Agent skills for developers building custom checkout components on the [Ollie Shop](https://github.com/ollie-shop) platform.

## What Are Skills?

Skills are structured instructions that enhance AI agents (like Claude Code) with domain-specific knowledge. They enable agents to assist developers with coding, CLI operations, design-to-code workflows, and more -- all within the Ollie Shop ecosystem.

## Available Skills

| Skill | Description |
|-------|-------------|
| [ollie-shop](skills/ollie-shop/) | Build, manage, and deploy custom checkout components for the Ollie Shop platform |

## Skill Structure

Each skill follows a standard layout:

```
skills/<skill-name>/
├── SKILL.md              # Main instructions (always loaded when skill triggers)
├── references/           # Deep-dive docs loaded on demand
├── assets/               # Static resources (templates, data files)
└── scripts/              # Executable automation
```

**Progressive disclosure**: The agent loads only what it needs. `SKILL.md` is the entry point -- it routes to reference files based on the developer's intent.

## The ollie-shop Skill

Covers four modes of operation:

1. **CLI Operations** -- Manage stores, components, versions, and deployments via the Ollie CLI
2. **SDK & Coding** -- Write components using the Ollie Shop SDK with proper patterns and best practices
3. **Component Library** -- Browse a catalog of reusable component patterns (CSV-driven, easy to extend)
4. **Component Design Flow** -- Go from Figma designs, screenshots, or business rules to a structured implementation plan

### Component Library

The component library (`assets/component-library.csv`) is a growing catalog of common checkout component patterns. Each entry includes:

- **component_name** -- Identifier
- **description** -- What it does and why
- **suggested_slot** -- Where it fits in the checkout template
- **path** -- Reference implementation path
- **tags** -- For semantic search and matching
- **complexity** -- low / medium / high
- **platform** -- agnostic or platform-specific (e.g., vtex)

### Design Flow

The component design flow helps developers build a **mental model** before writing code:

1. Gather context (Figma, screenshots, business rules, live website)
2. Identify and decompose business rules
3. Map to template slots
4. Design the component contract (types)
5. Check the component library for existing patterns
6. Implement
7. Validate and deploy

## Contributing

### Adding a Component to the Library

Add a row to `skills/ollie-shop/assets/component-library.csv` with all columns filled. If the component has a reference implementation, place it under the corresponding path.

### Updating Skill Documentation

Reference files under `references/` contain placeholder sections marked with `<!-- TODO: ... -->`. Replace these with actual documentation from the Ollie Shop codebase as it becomes available.

## References

- [Vercel Agent Skills](https://github.com/vercel-labs/agent-skills) -- Skill structure inspiration
- [Remotion Skills](https://github.com/remotion-dev/skills) -- Skill structure inspiration
