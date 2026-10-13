# Component Authoring Cheatsheet

The rules to have in context while writing an Ollie checkout component. Everything here is the strict subset that governs 95% of components; the deep-dive references below cover edge cases.

Load this file whenever you're writing a component, alongside **`file-organization.md`** — that pair is the default context for mode 6. The full deep-dive references (`complexity-validation-rules.md`, `design-contract.md`, `coding-standards.md`, `component-anatomy.md`) exist for cases the cheatsheet doesn't resolve — open them on demand, not by default.

---

## 1. File organization

Full rules — layout tree, subcomponent extraction, utils granularity, hooks scoping, `commons/` — live in **[`file-organization.md`](file-organization.md)** (loaded alongside this file by default in mode 6). The three thresholds you'll hit most often:

- **Extract a subcomponent** when render body approaches 100 lines OR there are 2+ conditional branches with JSX subtrees > 20 lines each. 1 extracted → root of the folder; 2+ → `components/` subfolder.
- **Utils granularity:** 1 helper = inline in `index.tsx`; 2–4 = `utils.ts`; 5+ = `utils/` folder with `utils/index.ts` barrel.
- **Hooks:** component-scoped hooks go in `hooks/use<Name>.ts` (subfolder from the first hook); cross-component hooks move to `commons/hooks/`.

`commons/` is a sibling of `components/` at the project root and holds everything shared across two or more slot components (`commons/icons/`, `commons/utils/`, `commons/hooks/`, `commons/types/`) plus component-shaped modules that must not be registered as slots (`commons/<ComponentLike>/`). Never inline `<svg>` in a component; use `commons/icons/<Name>.tsx`. Never Unicode emoji as icon.

If you need the full decision table, the icon example, or the "why" behind any of these rules, open `file-organization.md`.

---

## 2. Complexity limits (per component / helper)

| Rule | Limit |
|------|-------|
| `useState` count | ≤ 2 |
| `useEffect` count | ≤ 1 |
| `if` nesting in `useEffect` | ≤ 1 level |
| Derived booleans inline | ≤ 2 (extract more to a helper or hook) |
| Render body lines | ≤ 100 (then extract subcomponent — see above) |
| Optional-chaining chain | ≤ 2 levels (`a?.b?.c?.d` → extract `getNested(a)`) |
| Named-function body | ≤ 30 lines |
| Defensive parsing | Always extract to a named helper (`getX()`) |

If your code hits 2+ limits, refactor before moving on. Do not accept "close enough" — extract.

---

## 3. Design tokens (mandatory, no hardcoding)

| Category | Rule |
|---|---|
| **Colors** | `var(--color-primary|secondary|base-100|base-200|base-300|base-content|success|error|warning|info|…)`. Never hex, rgb, or oklch literals. |
| **Radius** | `var(--radius-field)` inputs/buttons; `var(--radius-box)` cards/modals; `var(--radius-selector)` checkbox/radio. Pill = `9999px`, circle = `50%`. No arbitrary px values. |
| **Font-family** | `font: inherit` or `font-family: inherit`. There are NO font tokens; do not hardcode families. |
| **Spacing** | `rem` values (`0.25`, `0.5`, `0.75`, `1`, `1.5`, `2`). `px` only for borders, breakpoints, icon dimensions. |
| **Transitions** | `200ms ease-out` state changes, `100ms` collapse, `300ms` modal. Respect `prefers-reduced-motion`. |

Full accessibility rules (focus indicators, contrast, touch targets, reduced-motion) live in `references/design-contract.md` § 5 — open on demand when you're on an interactive-heavy component.

---

## 4. TypeScript, exports, SDK

- `index.tsx` **must** `export default function <Name>(…)` — the slot loader resolves the default export.
- **No `import React from 'react'`** — automatic JSX runtime is on. Named imports only: `import { useState } from 'react'`.
- Import from `@ollie-shop/sdk` for state and dispatch: `useCheckoutSession`, `useCheckoutAction`, `useMessages`. Never call VTEX / Shopify / VNDA directly — the SDK is the anti-corruption layer.
- Match the hook and prop shapes documented in `sdk-guide.md`; do not invent new signatures.

---

## 5. Naming

| What | Convention |
|---|---|
| Component | PascalCase (`CouponInput`) |
| Props interface | `<Name>Props` (`CouponInputProps`) |
| Handler prop | `on<Action>` (`onApply`, `onChange`) |
| Boolean prop | `is<State>` / `has<Feature>` (`isLoading`, `hasError`) |
| CSS class | camelCase + component-prefixed (`freeShippingBarRoot`, `couponInputField`) — never generic (`.root`, `.card`, `.container`) because bundlers may dedupe |
| Domain variable | Self-documenting. `salesChannel`, not `sc`. `quantity`, not `qty`. Short names only at boundary (body field, header) |

---

## 6. Loading, error, empty states

Every component handles all three. Rules for the loading state:

- **Minimum DOM** — one wrapper + one or two shimmer blocks. Full layout only after hydration.
- **No `@keyframes` inside CSS Modules** — bundlers hash the name, animation silently breaks. Put keyframes in a sibling global `animations.css`, import it for side effect: `import "./animations.css";`.
- **Unique class names per module** — no `.root` `.card` shared across modules.
- **No composed classNames in the skeleton** (`${styles.a} ${styles.b}` multiplies failure modes).
- When in doubt: solid-color `<div>` sized to expected content. Boring beats broken.

For errors: fall back to a safe UI (`[]`, `null`, hide). Every slot is wrapped in an `ErrorBoundary` as a last resort — do not try to catch everything.

---

## 7. Currency and copy

- **Currency:** `new Intl.NumberFormat(session.locale.language, { style: "currency", currency: session.locale.currency }).format(cents / 100)`. **Never** hardcode `"en-US"`, `"USD"`, `"pt-BR"`, `"BRL"` — multi-region stores ship your component as-is.
- **Copy:** pt-BR default; use `useTranslations` from `next-intl` when the host provides it; otherwise expose overridable string props.
- Casing: sentence case for buttons and headings. Formality: `você`, never `tu`.

---

## 8. Observability

- Log prefix: `[<component-name>]` on every log — makes traces greppable.
- **On error** — `console.error("[name] context", { input, errorMessage, errorStack })`. Never `console.error(err)` — minified bundles drop useful fields.
- **On every REQUEST action** — log the input before dispatch.
- **On paint** — log whether the component is rendering live data or fallback.
- Never `console.log` in shipped code.
- Do not log on every render or lifecycle — pollutes prod consoles.

---

## 9. REQUEST guards (only if the component dispatches actions)

- **Validate input before dispatch** — if a URL fragment (`platformStoreId`, `sku`, `categoryId`) is empty/falsy, skip the call and log `[name] skipping REQUEST — <field> missing`.
- **Handle failures via `onError` callback** — not status codes. The SDK converts network / 4xx / 5xx into `ServerError` delivered through `onError` (and `error.serverError`); it does NOT re-throw into your `await`.
- **Unwrap progressively:** `response?.data?.data?.[0]`. Empty result ≠ error (render baseline or hide).
- **Track a `cancelled` flag** inside `useEffect` that dispatches; bail on cleanup. Late responses that `setState` on an unmounted component crash the slot.

---

## 10. Comments

- Every `.ts`/`.tsx` file starts with a 1–3 line docstring at the top explaining what the file provides — the responsibility, not the mechanics.
- **Zero inline comments** in the middle of code.
- **Zero per-prop or per-line comments** on the Props interface.
- Refactor and rename until the code reads itself.
- The one exception is a `// TODO(?): <question>` marker when a non-blocking detail surfaces mid-implementation and the user isn't around — see `coding-standards.md § Ambiguous decisions` for the escape hatch.

---

## 11. Self-check while writing (not a separate final turn)

Do the check as you write each file, not after. When you finish an `index.tsx`, before you save it, quickly walk this list:

- File starts with a docstring (§ 10)
- Uses `export default function` (§ 4)
- `useState` ≤ 2, `useEffect` ≤ 1, render body under 100 lines (§ 2)
- If render body ≥ 100 lines or 2+ big branches, extract subcomponents (§ 1)
- CSS uses only `var(--color-*)` for colors and `var(--radius-*)` for corners (§ 3)
- Any `Intl.NumberFormat` uses `session.locale.*` (§ 7)
- If dispatching REQUEST: input guard, `onError` fallback, optional-chaining unwrap (§ 9)

If a check fails, fix it in-place before you move on. Do not carry a debt list into the next file.

---

## Deep dives (open on demand)

Load these only when the cheatsheet doesn't cover what you need:

- `references/complexity-validation-rules.md` — full complexity examples, hub-function limits, actions/hooks-specific rules
- `references/design-contract.md` — full token catalog, per-element interaction states (button/input/link/card), motion tokens, iconography, don'ts
- `references/coding-standards.md` — full TypeScript/naming/CSS conventions, `commons/` folder rules, ambiguous-decision escape hatch, full observability defaults
- `references/component-anatomy.md` — bundler rules, registration patterns (per-folder `meta.json` vs root `ollie.json`), stage-specific overrides, shared assets pattern
- `references/sdk-guide.md` — full hook and action signatures, session shape, slot lifecycle
