# References Guide

Reference material for building Ollie checkout components. Organized as two layers.

---

## Layer 1 — Cheatsheet + File Organization (loaded by default when writing a component)

Two files paired together — mode 6 loads both by default.

### [Component Authoring Cheatsheet](component-authoring-cheatsheet.md)

The strict subset of rules that governs 95% of components. Covers, in ~150 lines:

- Complexity limits (useState, useEffect, render body, derived booleans, nesting)
- Design tokens (colors / radius / font / spacing / focus / transitions)
- TypeScript, SDK, and export conventions
- Naming (component, props interface, handlers, booleans, CSS classes)
- Loading, error, empty states (with the `@keyframes`-outside-modules rule)
- Currency and copy (Intl.NumberFormat with `session.locale`, pt-BR)
- Observability (log prefix, three mandatory log points)
- REQUEST guards (input validation, `onError` handling, progressive unwrap)
- Comments (one docstring at file top, zero inline)
- **Self-check while writing** — the checklist you run per file before saving

### [File Organization](file-organization.md)

Single source of truth for where files live. Covers, in ~130 lines:

- Layout tree for `components/<name>/` and `commons/`
- Subcomponent extraction (trigger + 1-at-root vs 2+-in-`components/` rule)
- Utils granularity (1 inline / 2–4 in `utils.ts` / 5+ in `utils/` with barrel)
- Hooks scoping (component-scoped `hooks/` vs cross-component `commons/hooks/`)
- Types (`types.ts` for non-trivial Props / cross-component in `commons/types/`)
- `commons/` — two use cases (shared code + component-shaped escape-hatch)
- Icon example + rules (no inline `<svg>`, no Unicode emoji)
- Quick decision table

If the cheatsheet + file-organization pair answers your question, do not open the deep-dive files. That is what keeps mode 6 fast.

---

## Layer 2 — Deep dives (open on demand)

Each file below is loaded only when the cheatsheet doesn't cover the specific case you're on.

| Deep dive | When to open |
|---|---|
| [Complexity Validation Rules](complexity-validation-rules.md) | Full rule tables with rationale + before/after examples. Hub-function limits (try-catch envelope, guards). Action-layer rules (sensitive data, cache fallback). |
| [Design Contract](design-contract.md) | Full token catalog per family, interaction states per element (button/input/link/card), motion tokens, iconography, don'ts, canonical CSS example. |
| [Coding Standards](coding-standards.md) | Full TypeScript/naming/CSS conventions, `commons/` folder pattern, ambiguous-decision escape hatch, observability defaults. |
| [Component Anatomy](component-anatomy.md) | Bundler rules (CJS/ES2020, external packages), registration patterns (per-folder `meta.json` vs root `ollie.json` map), stage-specific overrides, shared assets pattern (`./icons`, `./utils` at project root). |
| [SDK Guide](sdk-guide.md) | Full hook and action signatures, session shape, replace vs augment strategy, slot context props vs admin props, error handling, known-absent APIs. |
| [Hub Functions](hub-functions.md) | Hub function handler shape, trigger configuration, resolver/enricher/enforce patterns, cross-function state via cookies. |
| [Function Debugging](function-debugging.md) | When a deployed function misbehaves: triage order, symptom → cause → confirmation map, and the runtime limits that fail silently (`sessionStorage` ceiling, invocation time budget). |
| [Slots Catalog](slots-catalog.md) | The list of stable slot ids in the default template, plus dynamic slot patterns (e.g. `payment_option_{{ paymentMethodName }}`). |
| [CLI Reference](cli-reference.md) | Every `ollieshop` command, flags, examples. Deploy flows, what the deploy bundle carries and how `.ollieignore` trims it, version management, business rules. |
| [Component Design Flow](component-design-flow.md) | The cross-cutting questions (Figma, Case A vs B, template gap, library match) that run *before* opening a component's `INSTRUCTIONS.md`. Mode 6 loads this first, alongside the cheatsheet. |

---

## Related material outside `references/`

- **Component library:** `assets/components.csv` and per-component `assets/components/<id>/INSTRUCTIONS.md` — the behavior contract, states to cover, antipatterns, and "without a design reference" fallback for each canonical component.
- **Function library:** `assets/functions.csv` and per-function `assets/functions/<id>/INSTRUCTIONS.md` — same, for hub functions.
- **Slot data:** `assets/checkout-slots-data.yaml` — the raw catalog of slots with their context props.
- **Business rules:** `references/business-rules/*` — merchant-specific documentation.

---

## Making changes to this guide

If you update a deep-dive rule that should also govern authoring (e.g. a new hard limit on complexity, a new mandatory token), update the cheatsheet too. The cheatsheet is what Claude actually reads while writing — a rule that only exists in a deep-dive file is a rule Claude may not see.
