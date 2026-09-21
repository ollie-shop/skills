# Coding Standards — Ollie Shop Components

Standards that apply to every custom checkout component. Load this file when writing component code alongside `references/sdk-guide.md`.

**Related documents:**
- **[Component Authoring Cheatsheet](component-authoring-cheatsheet.md)** — the strict subset of rules loaded by default in mode 6
- **[Complexity Validation Rules](complexity-validation-rules.md)** — metric-based limits for code structure (useState, useEffect, nesting, etc.)

---

## TypeScript & SDK

- **TypeScript required.** All files are `.tsx` with explicit types.
- **Functional components only** — no class components.
- The component entry `index.tsx` **must** `export default function ComponentName(...)` (the Ollie slot loader resolves the default export). Subcomponents inside the folder may use named exports.
- **No `import React`** — the builder uses the automatic JSX runtime, so a top-level `import React from 'react'` is not needed (biome's `noUnusedImports` will strip it). Import named hooks/types only: `import { useState } from 'react'`, `import type { ReactNode } from 'react'`.
- Match the component signature documented in `references/sdk-guide.md` — do not invent hook or prop shapes.
- Prefer SDK state (via hooks) over component-local state for anything other components may need to see. `useState` is fine for UI-only state (focused, open, hovered); `useEffect` only for visual concerns (focus management, animation) — **never** for data fetching or business logic (use SDK actions). Follow limits in [Complexity Validation Rules § Components](complexity-validation-rules.md#components-react): `useState ≤ 2` and `useEffect ≤ 1` per component.
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

**One docstring at the top of every `.ts`/`.tsx` file, 1–3 lines, describing what the file provides** — the responsibility, not the mechanics. Every subcomponent, every helper file, every types file starts with one.

**Zero inline comments** in the middle of code. **Zero per-prop or per-line comments** on the Props interface. Refactor and rename until the code reads itself; when the code isn't clear, the fix is renaming or extracting, not adding a comment.

The one exception is a `// TODO(?): <question>` marker when a non-blocking detail surfaces mid-implementation and the user isn't around — see §Ambiguous decisions below.

---

## `commons/` — the umbrella for cross-component code

Full rules — folder layout, two use cases (shared code + escape-hatch for component-like modules), hooks scoping — are in **[`file-organization.md`](file-organization.md) § 3**. Kept there so cheatsheet, coding-standards, and component-anatomy don't drift apart.

---

## Definition of done

The full self-check list — file organization, complexity, tokens, focus, currency, REQUEST guards, docstrings — lives in `component-authoring-cheatsheet.md` § 11 "Self-check while writing". Run it inline as you write each file, not at the end.

The structural must-haves specifically owned here (repo layout / registration):

1. Output in `components/<name>/` at the consumer repo root (or `commons/<name>/` for import-only modules — see §`commons/` above).
2. Registration the project uses — either a `meta.json` in the folder or an entry in the root `ollie.json` `components` map. See `component-anatomy.md` §Two registration patterns.
3. `tsc` clean and lint clean.
4. Slot id verified against `slots-catalog.md` / `assets/checkout-slots-data.yaml`.

---

## Ambiguous decisions — ask first, `// TODO(?)` as a fallback

When the spec doesn't pin down a platform detail (envelope shape, name→id resolution, fallback when a lookup fails, replace vs augment, retry policy, etc.), **ask the user** — preferably batched up front, per the pause-and-ask rule in `references/component-design-flow.md`. Anything that changes the approach or could ship the wrong behavior is a question, not a guess.

When asking isn't worth it — a minor, non-blocking detail that surfaces mid-implementation, or the user isn't around — implement the simplest default that works and leave a one-line `// TODO(?): <question>` next to it so it's resolved in review. Never silently guess on something load-bearing.

Examples of TODO-worthy (minor) calls:
- `// TODO(?): when categoryId lookup fails, fall back to ft= name search or hide?`
- `// TODO(?): retry on 5xx or fail closed?`

---

## Observability defaults

**Logging prefix:** Every component logs with the prefix `[<component-name>]` — see [Complexity Validation Rules § Actions](complexity-validation-rules.md#actions-hooks--api-functions) (line 126) for reasoning.

Three logs are mandatory:

- **On error** — `console.error` with `{ input, errorMessage, errorStack }` extracted explicitly (do NOT log raw `err` — minified bundles drop useful info).
- **On every `useCheckoutAction("REQUEST")` call** — log the input before dispatch.
- **On the paint decision** — log whether the component is rendering with live data or a fallback.

REQUEST failures don't reach a component `try/catch` — `ky` throws server-side, the action-client converts it to a `ServerError`, and you receive it via the `onError` callback / `error.serverError` (your `await` resolves, it doesn't throw). Handle errors there and return a safe fallback (`[]`, `null`) for the paint; every slot is already wrapped in an `ErrorBoundary` as a last resort. Do not log on every render or every lifecycle hook — that pollutes prod consoles.

---

## Guards on every `REQUEST` action

A REQUEST has many failure modes beyond network throws. All of these are mandatory, not optional:

- **Validate input before dispatch.** If a required URL fragment (`platformStoreId`, `categoryId`, `sku`, etc.) is empty or falsy, skip the call entirely and log `skipping REQUEST — <field> missing`. Never build a URL with an empty segment and wait for a 404.
- **Handle failures via `onError` / `error.serverError`, not a status check.** On a 4xx/5xx (or network failure) the platform call throws server-side; the SDK's action-client converts it to a `ServerError` and delivers it through the `onError` callback (and `error.serverError`). It is **not** re-thrown into your `await`, and you do **not** get a `{ status: 500 }` object back — so put fallback logic in `onError`, and also treat a missing/`undefined` result from `executeAsync` as failure before using it.
- **Unwrap progressively with optional chaining.** Never `response.data.data[0]` directly — each layer can be missing. Use `response?.data?.data?.[0]` and handle `undefined` at every step. The platform envelope may be single-wrapped, double-wrapped, or empty depending on the endpoint. See [Complexity Validation Rules § Parsing & Type Safety](complexity-validation-rules.md#parsing--type-safety) for detailed guidance and examples.
- **Empty is not an error.** `products?.length === 0` means "no match" — render the baseline or hide, do not throw or show "Error loading". Only network / 5xx / shape-mismatch are errors.
- **Abort on unmount.** If your `useEffect` dispatches a REQUEST, track a `cancelled` flag and bail in cleanup. A late response that calls `setState` on an unmounted component can crash the slot.

---

## Loading states — keep them minimal

Skeletons, placeholders, and any UI rendered during `isLoading` or SSR bootstrap are prime spots for **hydration mismatches**, **CSS-module keyframe scoping issues**, and **class-name collisions** across modules — failures that crash the whole slot with a generic "client-side exception" and are hard to debug. Constraints:

- **Minimum DOM.** One wrapper + one or two shimmer blocks. No nested card-like structures in the loading state. Render the full layout only AFTER hydration.
- **No `@keyframes` inside CSS Modules.** Keyframe scoping behavior varies across bundlers — and with **bun** (and esbuild's local-css loader) a `@keyframes shimmer` defined inside a `*.module.css` gets scoped/hashed while the `animation: shimmer` reference does **not** resolve to the hashed name, so the animation silently never runs. **Put every `@keyframes` in a separate global (non-module) `.css` file** — e.g. `animations.css` next to the component — and import it for its side effect from the component entry (`import "./animations.css";`). The keyframe names stay literal (global), and the module's `animation: <name>` refs resolve against them. Keep the `animation:` shorthand in the module CSS; only the `@keyframes` blocks move out. (Or, for a loading skeleton, just ship a static placeholder with no animation.)
- **Unique class names per module.** Do not reuse generic names like `.root`, `.card`, `.container` across two modules in the same component folder — some bundlers dedupe. Prefer `.skeletonRoot` + `.cardRoot`, or one module per visual concern.
- **Avoid composed classNames in the loading state.** `${styles.a} ${styles.b} ${styles.c}` chains multiply the surface for hashed-name resolution failures. One class per element in the skeleton.
- **No `aria-live` on the loading container.** `aria-hidden="true"` on the wrapper is enough; a single `<span className="sr-only">Loading…</span>` covers screen readers.
- When in doubt, ship a solid-color `<div>` sized to the expected content and call it a day. A working dumb skeleton beats a broken clever one.
