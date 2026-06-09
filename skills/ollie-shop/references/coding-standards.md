# Coding Standards — Ollie Shop Components

Standards that apply to every custom checkout component. Load this file when writing component code alongside `references/sdk-guide.md`.

---

## TypeScript & SDK

- **TypeScript required.** All files are `.tsx` with explicit types.
- **Functional components only** — no class components.
- The component entry `index.tsx` **must** `export default function ComponentName(...)` (the Ollie slot loader resolves the default export). Subcomponents inside the folder may use named exports.
- **No `import React`** — the builder uses the automatic JSX runtime, so a top-level `import React from 'react'` is not needed (biome's `noUnusedImports` will strip it). Import named hooks/types only: `import { useState } from 'react'`, `import type { ReactNode } from 'react'`.
- Match the component signature documented in `references/sdk-guide.md` — do not invent hook or prop shapes.
- Prefer SDK state (via hooks) over component-local state for anything other components may need to see. `useState` is fine for UI-only state (focused, open, hovered); `useEffect` only for visual concerns (focus management, animation) — **never** for data fetching or business logic (use SDK actions).
- Always handle loading and error states — checkout UX must be resilient.

---

## Naming & prop conventions

- **Components:** PascalCase (`CouponInput`).
- **Props interface:** `<ComponentName>Props` (`CouponInputProps`).
- **Handler props:** `on<Action>` (`onApply`, `onChange`) — callbacks only; the parent decides behavior.
- **Boolean props:** `is<State>` / `has<Feature>` (`isLoading`, `isDisabled`).

Common prop patterns:

- **Inputs/fields:** `value`, `onChange`, optional `placeholder`, `label`, `helperText`, `errorMessage`, `isDisabled`.
- **Actions:** `onApplyCoupon`, `onRemoveItem`, `onToggle`.
- **Async UI:** `isLoading`, `errorMessage`, `successMessage` — toggle visuals only.
- **Lists:** `items` (array), `onSelectItem`, `onRemoveItem`.

---

## CSS class naming

- All classes **camelCase** and **prefixed with the component's own name** → `freeShippingBarRoot`, `couponInputField`. (See §Loading states below for the unique-class-name rule that prevents bundler dedupe.)
- CSS Modules for all styling; prefer HTML + CSS over JS-driven styling.
- Responsiveness via CSS media queries, **not** `isMobile` props.
- Use only tokens from `references/design-contract.md` — never invent token names.

---

## Images

Pass `src` as a prop so integrators can wire real assets. Use a plain `<img>` (do not assume a framework `<Image />`). Keep the markup inside the component's own folder tree. For icons, follow `references/component-anatomy.md` §Shared assets (inline SVG components, never emoji glyphs).

---

## Comments

**Default to none.** Only add one when the WHY is non-obvious (hidden constraint, workaround for a specific bug, behavior that would surprise a reader). Don't narrate WHAT the code does. No JSDoc on trivial helpers, no section-divider comments.

---

## `commons/` for shared / import-only modules

**Components without a `meta.json` are NOT slot components** — they are import-only/shared modules.

Keep them in a top-level **`commons/`** folder (a sibling of `components/`, NOT inside it) so the `components/*/index.tsx` discovery glob never registers them to a slot. A real slot component (with `meta.json`) imports them directly (e.g. `import Banner from "../../commons/Banner"`).

Use this when two components would otherwise fight over the same slot: render one inside the other and demote the inner one to `commons/`.

---

## Definition of done

Before handoff, verify all of:

1. Output in `components/<name>/` at the consumer repo root (or `commons/<name>/` for import-only modules).
2. `index.tsx` + `index.module.css` + `meta.json` present.
3. `tsc` clean and lint clean.
4. Loading + error + empty states rendered.
5. Slot id verified against `assets/checkout-slots-data.yaml`.
6. `index.tsx` entry at the component root (subfolders/cross-folder imports within the component tree are fine).
7. Only tokens from `references/design-contract.md` used in CSS.

---

## Defer ambiguous decisions with `// TODO(?)`

When the spec doesn't pin down a platform detail (envelope shape, name→id resolution, fallback when a lookup fails, replace vs augment, retry policy, etc.), do NOT stop to ask in chat. Implement the simplest default that works and leave a one-line `// TODO(?): <question>` next to it. The user resolves TODOs in review.

Examples:
- `// TODO(?): when categoryId lookup fails, fall back to ft= name search or hide?`
- `// TODO(?): retry on 5xx or fail closed?`

---

## Observability defaults

Every component logs with the prefix `[<component-name>]`. Three logs are mandatory:

- **On error** — `console.error` with `{ input, errorMessage, errorStack }` extracted explicitly (do NOT log raw `err` — minified bundles drop useful info).
- **On every `useCheckoutAction("REQUEST")` call** — log the input before dispatch.
- **On the paint decision** — log whether the component is rendering with live data or a fallback.

Wrap every `REQUEST` in `try/catch` that returns a safe fallback (`[]`, `null`) so the slot stays alive on failure. Do not log on every render or every lifecycle hook — that pollutes prod consoles.

---

## Guards on every `REQUEST` action

A REQUEST has many failure modes beyond network throws. All of these are mandatory, not optional:

- **Validate input before dispatch.** If a required URL fragment (`platformStoreId`, `categoryId`, `sku`, etc.) is empty or falsy, skip the call entirely and log `skipping REQUEST — <field> missing`. Never build a URL with an empty segment and wait for a 404.
- **Check HTTP status BEFORE parsing.** `REQUEST` does NOT throw on 4xx/5xx — it resolves with `{ status: 500, data: … }`. Guard explicitly: `if (!response || response.status >= 400) { log + return fallback }`.
- **Unwrap progressively with optional chaining.** Never `response.data.data[0]` directly — each layer can be missing. Use `response?.data?.data?.[0]` and handle `undefined` at every step. The platform envelope may be single-wrapped, double-wrapped, or empty depending on the endpoint.
- **Empty is not an error.** `products?.length === 0` means "no match" — render the baseline or hide, do not throw or show "Error loading". Only network / 5xx / shape-mismatch are errors.
- **Abort on unmount.** If your `useEffect` dispatches a REQUEST, track a `cancelled` flag and bail in cleanup. A late response that calls `setState` on an unmounted component can crash the slot.

---

## Loading states — keep them minimal

Skeletons, placeholders, and any UI rendered during `isLoading` or SSR bootstrap are prime spots for **hydration mismatches**, **CSS-module keyframe scoping issues**, and **class-name collisions** across modules — failures that crash the whole slot with a generic "client-side exception" and are hard to debug. Constraints:

- **Minimum DOM.** One wrapper + one or two shimmer blocks. No nested card-like structures in the loading state. Render the full layout only AFTER hydration.
- **No `@keyframes` inside CSS Modules.** Keyframe scoping behavior varies across bundlers — `@keyframes shimmer` defined next to `.shimmer { animation: shimmer }` can silently fail to resolve the hashed name. Use a static placeholder (solid background, no animation) or define the keyframe in a global stylesheet outside the module.
- **Unique class names per module.** Do not reuse generic names like `.root`, `.card`, `.container` across two modules in the same component folder — some bundlers dedupe. Prefer `.skeletonRoot` + `.cardRoot`, or one module per visual concern.
- **Avoid composed classNames in the loading state.** `${styles.a} ${styles.b} ${styles.c}` chains multiply the surface for hashed-name resolution failures. One class per element in the skeleton.
- **No `aria-live` on the loading container.** `aria-hidden="true"` on the wrapper is enough; a single `<span className="sr-only">Loading…</span>` covers screen readers.
- When in doubt, ship a solid-color `<div>` sized to the expected content and call it a day. A working dumb skeleton beats a broken clever one.
