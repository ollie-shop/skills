# Design Contract — Custom Checkout Components (Ollie Shop)

> Audience: front-end engineer (or AI pairing) writing React + TypeScript + **CSS Modules** components for the Ollie Shop checkout. All tokens below are CSS custom properties exposed in [`packages/config/tailwind.css`](packages/config/tailwind.css). Always consume via `var(--token)`. **Never** use Tailwind utility classes in these components.

---

## 1. Design Tokens

### Colors

DaisyUI convention: `--color-{role}` + `--color-{role}-content` for text/icon on top of that role.

| Family | Tokens | Use for |
|---|---|---|
| Base | `--color-base-100`, `--color-base-200`, `--color-base-300`, `--color-base-content` | Page/card background, subtle surfaces, dividers, default text |
| Primary | `--color-primary`, `--color-primary-content` | CTA, links, focus ring |
| Secondary | `--color-secondary`, `--color-secondary-content` | Brand accent, promotional badges |
| Accent / Neutral | `--color-accent(-content)`, `--color-neutral(-content)` | Punctual highlights, neutral chips |
| Status | `--color-info`, `--color-success`, `--color-warning`, `--color-error` (+ `-content`) | Inline icons/borders |
| Alert surfaces | `--color-alert-{info\|success\|warning\|error}(-content)` | Toast/alert backgrounds |

### Typography

| Token | Use for |
|---|---|
| `--font-primary` | Default UI family |
| `--font-mono` | Numbers, codes, coupons |

No canonical tokens for sizes/weights yet — use `rem` inline. Common scale: `0.75rem` caption, `0.875rem` sm, `1rem` body, `1.125rem` lg, `1.25rem` xl. Weights: `500` labels, `600` CTA/headings.

### Radius

| Token (`0.5rem`) | Use for |
|---|---|
| `--radius-field` | Inputs, selects, buttons |
| `--radius-box` | Cards, modals, containers |
| `--radius-selector` | Checkbox, radio, chip |

Pill/circle: `9999px` inline.

### Spacing, shadows, z-index

No canonical tokens. Use `rem` inline for spacing (`0.25`, `0.5`, `0.75`, `1`, `1.5`, `2`). Inline shadows: `sm 0 1px 2px oklch(0% 0 0 / 0.06)`, `md 0 4px 12px oklch(0% 0 0 / 0.08)`. Z-index: `1` base, `10` dropdown, `20` sticky, `30` overlay, `40` modal, `50` toast.

---

## 2. Breakpoints & responsive

Tailwind v4 breakpoints — use in `@media` directly, **mobile-first**.

| Name | Min-width |
|---|---|
| `sm` | `640px` |
| `md` | `768px` |
| `lg` | `1024px` |
| `xl` | `1280px` |

Checkout layout: single-column on mobile, two-column (fixed summary on the right) from `lg`. Minimum touch target on mobile: **44×44 px**.

---

## 3. Interaction states

### Canonical focus ring (always via `:focus-visible`, never `:focus`)

```css
outline: 2px solid var(--color-primary);
outline-offset: 2px;
border-radius: inherit;
```

For inputs, swap `outline` for a colored `box-shadow` on the border:

```css
outline: none;
border-color: var(--color-primary);
box-shadow: 0 0 0 2px var(--color-primary);
```

### Per element

| Element | default | hover | active | disabled | error |
|---|---|---|---|---|---|
| **Button** | variant bg | darken ~8% via `color-mix(in oklch, var(--color-primary) 92%, black)` | `translateY(1px)` (optional) | `opacity: 0.5`, `cursor: not-allowed` | use `danger` variant |
| **Input** | border `1px solid --color-base-300`, bg `--color-base-100` | border `--color-base-content` | — | `opacity: 0.5`, bg `--color-base-200` | border `2px solid --color-error` + helper text |
| **Link** | `--color-primary`, no underline | underline + `text-underline-offset: 2px` | `opacity: 0.8` | `opacity: 0.5`, `pointer-events: none` | — |
| **Clickable card** | subtle border/shadow | stronger shadow or `--color-primary` border | `transform: scale(0.995)` (optional) | `opacity: 0.5`, `pointer-events: none` | — |

Transitions: `200ms ease-out` standard. Collapse/expand `100ms ease-in-out`. Modal `300ms`.

---

## 4. Component patterns

**Button** — height `2.75rem`, padding `0 1rem`, radius `--radius-field`, `font-family: var(--font-primary)`, `1rem`, weight `600`.

| Variant | Background | Text |
|---|---|---|
| `primary` | `--color-primary` | `--color-primary-content` |
| `secondary` | `--color-secondary` | `--color-secondary-content` |
| `ghost` | transparent (hover `--color-base-200`) | `--color-base-content` |
| `danger` | `--color-error` | `--color-error-content` |

**Input** — height `2.75rem`, padding `0 0.75rem`, radius `--radius-field`. Floating label shrinks to `0.75rem`. Helper/error text: `0.75rem`, `margin-top: 0.25rem`. Placeholder: `--color-base-content` with `opacity: 0.5`. `<label for>` is mandatory — placeholder is not a substitute.

**Card / surface** — bg `--color-base-100`, border `1px solid --color-base-300` **or** light shadow (not both), radius `--radius-box`, padding `1rem` mobile / `1.5rem` desktop.

**List / rows** — separator `1px solid --color-base-300`, gap `0.75rem`, `display: flex; align-items: center`.

**Toast** (`useMessages` from the SDK) — bg `--color-alert-{type}`, text `--color-alert-{type}-content`, radius `--radius-box`, padding `0.75rem 1rem`. Position: top-right on desktop, top full-width on mobile.

**Modal** — overlay `oklch(0% 0 0 / 0.5)`; container `max-width: 32rem` (mobile `calc(100vw - 2rem)`), bg `--color-base-100`, radius `--radius-box`, padding `1.5rem`; close button top-right with `aria-label`; z-index `40`.

**Badge / chip** — height `1.5rem`, padding `0 0.5rem`, `0.75rem`, weight `600`, radius `9999px`. Promo: `--color-secondary`. Success: `--color-alert-success`. Neutral: `--color-base-200`.

---

## 5. Accessibility (mandatory)

- Contrast AA: `4.5:1` for text ≤18px, `3:1` for ≥18px bold.
- `:focus-visible` on every interactive. Never `outline: none` without a visible replacement.
- `@media (prefers-reduced-motion: reduce) { * { transition: none !important; animation: none !important; } }`
- Mobile touch target ≥ `44×44 px`.
- `<label for>` on every input. Decorative icons `aria-hidden="true"`; semantic icons need `aria-label`.

---

## 6. Iconography

Library: custom SVG from `batman-ui` ([`IconSet.tsx`](packages/batman-ui/src/components/Icon/IconSet.tsx), 75 icons). Use `currentColor` (inherits from text). Sizes (px, inline): `xs 16`, `sm 20`, `md 24` (default), `lg 32`, `xl 40`.

---

## 7. Motion

Durations: `fast 100ms` (hover, collapse), `base 200ms` (state changes, floating labels), `slow 300ms` (modal). Easing: `ease-out` (enter), `ease-in` (exit), `ease-in-out` (toggle). **Do not animate** price/total, validation errors, or data-loading placeholders.

---

## 8. UI tone & copy

- Language: **pt-BR**. All strings via `useTranslations` ([`packages/messages/src/`](packages/messages/src/)).
- Formality: **você** (never "tu"); direct imperative for actions ("Finalizar pedido").
- Casing: **Sentence case** for buttons and headings. UPPERCASE reserved for short status badges only (`NOVO`, `PROMO`).
- Error messages: diagnose + next step (`"Não foi possível aplicar o cupom. Verifique o código e tente novamente."`).
- Empty states: fact + suggestion (`"Seu carrinho está vazio. Que tal adicionar alguns produtos?"`).

---

## 9. Don'ts

- ❌ Tailwind utility classes in custom components — CSS Modules only.
- ❌ Hard-coded colors in hex/rgb/oklch inline. Always `var(--color-*)`.
- ❌ `px` for internal spacing — use `rem`. `px` only for borders, breakpoints, icon dimensions.
- ❌ Fonts outside `--font-primary` / `--font-mono`.
- ❌ `outline: none` without a visible `:focus-visible` replacement.
- ❌ Animating `opacity` on price, total, or payment CTA.
- ❌ `!important` in CSS Modules.
- ❌ `theme()` or any Tailwind helper inside CSS Modules.
- ❌ Exposing `className` / `style` as a prop.
- ❌ Inventing tokens — add them to [`tailwind.css`](packages/config/tailwind.css) **before** consuming.

---

## 10. Canonical example

```css
/* example.module.css — primary button, input with error, card */
.card {
  background: var(--color-base-100);
  border: 1px solid var(--color-base-300);
  border-radius: var(--radius-box);
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  font-family: var(--font-primary);
  color: var(--color-base-content);
}
@media (min-width: 768px) { .card { padding: 1.5rem; } }

.input {
  height: 2.75rem;
  padding: 0 0.75rem;
  font: inherit;
  color: var(--color-base-content);
  background: var(--color-base-100);
  border: 1px solid var(--color-base-300);
  border-radius: var(--radius-field);
  transition: border-color 200ms ease-out, box-shadow 200ms ease-out;
}
.input:hover:not(:disabled) { border-color: var(--color-base-content); }
.input:focus-visible {
  outline: none;
  border-color: var(--color-primary);
  box-shadow: 0 0 0 2px var(--color-primary);
}
.input:disabled { background: var(--color-base-200); opacity: 0.5; cursor: not-allowed; }

.inputError,
.inputError:focus-visible {
  border-color: var(--color-error);
  box-shadow: 0 0 0 2px var(--color-error);
}
.errorText { font-size: 0.75rem; color: var(--color-error); margin-top: 0.25rem; }

.button {
  height: 2.75rem;
  padding: 0 1rem;
  font-family: inherit;
  font-size: 1rem;
  font-weight: 600;
  color: var(--color-primary-content);
  background: var(--color-primary);
  border: none;
  border-radius: var(--radius-field);
  cursor: pointer;
  transition: background-color 200ms ease-out, transform 100ms ease-out;
}
.button:hover:not(:disabled) { background: color-mix(in oklch, var(--color-primary) 92%, black); }
.button:focus-visible { outline: 2px solid var(--color-primary); outline-offset: 2px; }
.button:active:not(:disabled) { transform: translateY(1px); }
.button:disabled { opacity: 0.5; cursor: not-allowed; }

@media (prefers-reduced-motion: reduce) {
  .input, .button { transition: none; }
  .button:active:not(:disabled) { transform: none; }
}
```

---

## Not yet defined

Once decided, add to [`tailwind.css`](packages/config/tailwind.css) and update this contract:

- `--color-text-muted`, `--color-border`, `--color-action-hover` (today ad-hoc via `opacity` / `color-mix`)
- Spacing scale (`--space-*`)
- Font-size scale (`--font-size-*`)
- Shadows (`--shadow-sm/md/lg`)
- Z-index tokens
- Motion tokens (`--motion-fast/base/slow`)
- Icon size tokens (`--icon-*`)
