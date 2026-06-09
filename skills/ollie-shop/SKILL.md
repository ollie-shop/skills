---
name: ollie-shop
description: >
  Build, manage, and ship custom checkout components and hub functions on the Ollie Shop platform.
  Use this skill whenever the user mentions ollie-shop, Ollie checkout, the `ollieshop` CLI,
  checkout slots, checkout templates, business rules, or hub functions; or when they want help
  writing a checkout component (with or without a Figma reference), wiring a function that
  intercepts a checkout request/response, or deploying customizations to a store on top of
  any supported commerce platform (VTEX, Shopify, VNDA, or custom).
---

# Ollie Shop — Customizing Checkouts

You are helping a developer customize a checkout on **Ollie Shop**, a platform-agnostic checkout that plugs into commerce backends (currently VTEX, Shopify, VNDA, or a custom integration) and exposes three customization axes: **Components** (replace UI inside template slots), **Functions** (hub middleware on request/response, except PCI-protected paths), and **Admin** settings (colors, translations, toggles via UI).

## How this skill is organized

The skill has **seven modes**. Pick one based on the user's intent (keywords listed). When the request straddles two modes, start with whichever produces the next concrete user-facing decision; you can hand off after.

| Mode | Triggers | Load |
|---|---|---|
| **1. CLI Operations** | "deploy", "run locally", "ollieshop", "login", "business rule", "create a version", "trigger" | `references/cli-reference.md` |
| **2. SDK & Coding** | "how do I use", "hook", "session", "useCheckoutAction", "anatomy", "loading state", "coding standards", "REQUEST guard" | `references/sdk-guide.md` + `references/component-anatomy.md` + `references/coding-standards.md` |
| **3. Slot System** | "which slot", "slot id", "slot lifecycle", "slot visibility", "slot props", "checkout-slots" | `references/slots-reference.md` + `assets/checkout-slots-data.yaml` + `references/slots-catalog.md` |
| **4. Design Contract** | "design token", "css module", "token name", "a11y", "accessibility", "styling rules" | `references/design-contract.md` |
| **5. Library lookup** | "is there a pattern for", "example of", "does Ollie already have", "UI primitive" | `assets/components.csv` / `assets/functions.csv` + the matched entry's `INSTRUCTIONS.md`; for UI primitives: `assets/UI/README.md` |
| **6. Component design flow** | "create a component", "new component", "from Figma", "I want to customize", "designer", "screenshot" | `references/component-design-flow.md` first, then `references/component-authoring.md` (when designer-facing), then the target entry's `INSTRUCTIONS.md` |
| **7. Function authoring** | "hub function", "middleware", "external validation", "call an API", "intercept", "rewrite request" | `references/hub-functions.md` + the target function's `INSTRUCTIONS.md` |

## Question hierarchy — read this before asking the user anything

To avoid repeating yourself or contradicting questions across files, follow this order:

1. **Cross-cutting first** (`component-design-flow.md`): if the request is about creating or changing a component or function, surface the design-flow questions before the entry-specific ones — Figma MCP availability, whether the integration is a native SDK capability or a custom client integration (HAR / docs / payload sample), and so on.
2. **Entry-specific second** (the matched `INSTRUCTIONS.md`): each component/function entry owns its own narrow questions (3–5) about layout, validation, behavior. Those live inside `assets/components/<id>/INSTRUCTIONS.md` or `assets/functions/<id>/INSTRUCTIONS.md`.
3. **Never repeat**: skip any question whose answer is already in the initial user prompt or in a previous answer from the design flow.

If you can't tell which entry matches, finish the design flow first, then re-check the library — the design-flow answers usually narrow the match.

## General principles

- **Be opinionated**. If the user does not specify a layout or integration approach, follow the Ollie default for that component or function (the `Default opinionated pattern` section of each `INSTRUCTIONS.md`). Only ask the user to override the default when the request demands it.
- **Components are self-contained**. A component is a default-exported React function in `./components/<Name>/index.tsx` that uses `@ollie-shop/sdk` hooks for state and dispatch. Never call the underlying commerce platform directly — the SDK is the anti-corruption layer.
- **Slots belong to a template**. The default template ships around 60 slots (see `references/slots-catalog.md`). Slot ids are stable strings; some are **dynamic** with a `{{ variable }}` segment — treat them as a pattern, not a fixed list (e.g. `payment_option_{{ paymentMethodName }}` matches `payment_option_pix`, `payment_option_promissory`, and any future method without a skill update).
- **Functions are HTTP middleware**. A function intercepts a request or response on the hub. Request functions can rewrite `requestInit` (headers, cookies, auth) before forwarding, or respond directly without proxying. Response functions can mutate the payload or reject it. PCI-protected destinations are blocked at the hub.
- **Coding standards** (`commons/` layout, definition-of-done, REQUEST guards, loading states, observability) live in `references/coding-standards.md` — load it alongside `sdk-guide.md` when writing component code.
- **Customization tooling lives in the customer's repo under `.claude/ollie/`** (managed by Ollie, replaced on update). Anything outside that directory is the customer's own and is never touched by tooling sync.

## Local dev loop (single source of truth)

`ollieshop start` is the day-to-day loop:
1. Bundles every `./components/*/index.tsx` with esbuild watch.
2. Serves them on `http://localhost:4000`.
3. Opens Studio (`https://admin.ollie.shop/studio`) iframing the live checkout with your local bundles overriding the deployed ones.
4. On save, esbuild rebuilds and Studio reloads via SSE.

When you're happy with the result, **deploy from Studio's preview UI** (for components) or **upload the zip via the admin form** (for functions). The `ollieshop deploy` CLI command exists but is reserved for CI/scripted flows.

## Working with business rules

Business rules document the merchant's customization intent and can link to specific versions, components, and functions. List, get, and update them with `ollieshop business-rule …` — full commands in `references/cli-reference.md`. Merchant-specific business rules are also documented in markdown under `references/business-rules/` (one file per merchant). When implementing a customization, ask whether there is a business rule to link it to; if not, ask if the user wants you to create one.

## Figma and design integration

If the user mentions Figma, suggest installing the Figma MCP — with MCP the agent reads the design structure directly, which is much more reliable than a screenshot. This is part of the cross-cutting design flow; do not ask "do you have a Figma?" inside an `INSTRUCTIONS.md` — that question lives in `component-design-flow.md`.

