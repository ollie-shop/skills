# Ollie Shop Skills

Agent skills that teach AI assistants (Claude Code, Cursor, and other compatible tools) how to build on the Ollie Shop platform — the CLI, the SDK, the checkout slots, design tokens, hub functions, and the workflows that tie them together.

## Install

```bash
npx skills add ollie-shop/skills --agent claude-code
```

This installs the skill into `.claude/skills/ollie-shop/` in your project. Run the same command later to update.

> Using Cursor, Windsurf, or another supported agent? Swap the flag for `--agent cursor`, `--agent windsurf`, etc. Without `--agent`, the CLI tries to symlink across all agents at once, which requires Developer Mode on Windows.

## What your AI gets

Once installed, your AI assistant knows how to:

- Use the `ollieshop` CLI — deploy, login, create components, manage versions and business rules.
- Build custom checkout components using `@ollie-shop/sdk` hooks (`useCheckoutSession`, `useCheckoutAction`, and others).
- Pick the right checkout slot for a customization, with the full slot catalog as context.
- Follow the design contract — tokens, accessibility, motion, complexity rules.
- Write hub functions for request/response middleware on the Ollie Hub.

## Documentation

Full Ollie Shop documentation: [docs.ollie.shop](https://docs.ollie.shop).

## License

[MIT](./LICENSE)
