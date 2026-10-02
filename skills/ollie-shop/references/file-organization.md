# File Organization

Single source of truth for where things live inside a component folder, and how to share code across components. Loaded by default in mode 6 alongside `component-authoring-cheatsheet.md`.

---

## 1. Layout at a glance

```
components/<name>/                # this component's own tree
├── index.tsx                     # required — default export, composes
├── index.module.css              # required — CSS module
├── types.ts                      # custom domain types or non-trivial Props
├── utils.ts                      # 2–4 pure helpers (single file)
├── utils/                        # 5+ helpers
│   ├── index.ts                  # barrel — `export * from "./sortByPrice"; …`
│   └── <helperName>.ts           # one helper per file
├── <Subcomponent>.tsx            # 1 extracted subcomponent → root of the folder
├── components/                   # 2+ extracted subcomponents → subfolder
│   └── <SubName>.tsx
├── hooks/                        # component-scoped data/logic hooks
│   └── use<Name>.ts
└── animations.css                # `@keyframes` live here, NEVER inside a CSS module

# Registration (per-folder `meta.json` or root `ollie.json` map) is handled by
# the customization tooling — the component author does not write it. See
# `component-anatomy.md § Two registration patterns` for the mechanism.

commons/                          # SIBLING of components/ at the project root
├── icons/<IconName>.tsx          # inline-SVG icon components shared by 2+ components
├── utils/<helper>.ts             # cross-component pure helpers
├── hooks/use<Name>.ts            # cross-component hooks
├── types/<Name>.ts               # shared type definitions
└── <ComponentLike>/index.tsx     # component-shaped module NOT registered as a slot
```

Every extracted `.ts` / `.tsx` file starts with a 1–3 line docstring describing what the file provides — the responsibility, not the mechanics. This is the file-top comment rule from `component-authoring-cheatsheet.md` § 10 applied everywhere.

---

## 2. Inside the component folder

### Subcomponents — when to extract

Extract a subcomponent when either:
- The render body of `index.tsx` approaches 100 lines (the complexity ceiling — see `complexity-validation-rules.md`), OR
- There are 2+ conditional branches whose JSX subtree each exceeds ~20 lines (`if (isAvailable) return <Big/> else return <BiggerStill/>` is the exact case)

`index.tsx` then reads: pick the branch, render it, done. It should not carry 100-line JSX blocks itself.

### Subcomponents — where to put them

- **1 extracted** → keep at the folder root as `<SubName>.tsx` next to `index.tsx`. A single sibling file is easier to scan than a subfolder holding one thing.
- **2+ extracted** → create a `components/` subfolder inside the component folder and put them there. Keeps the root uncluttered when there are several.

### Utils granularity

- 1 helper → keep inline in `index.tsx`
- 2–4 helpers → `utils.ts` in the same folder (single file, all helpers)
- 5+ helpers → `utils/` folder with `utils/index.ts` barrel re-exporting each helper (one helper per file)

Above 5 helpers, one-file-per-helper makes the folder browsable at a glance and lets each helper have its own docstring.

### Hooks

- **Component-scoped** (only this component uses the hook) → `hooks/use<Name>.ts` subfolder inside the component. Use the subfolder from the first extracted hook — there is no count threshold for hooks, because hooks usually pack more than a helper does and readers expect them under `hooks/`.
- **Cross-component** (a second slot needs the same hook) → move to `commons/hooks/` per § 3 below. Do not leave a shared hook under one component's `hooks/` — the second consumer will feel weird importing it.

### Types

- Custom domain types or non-trivial Props (more than a few primitive fields) → `types.ts` in the component folder
- Trivial Props like `{ value: string; onChange: (v: string) => void }` → stay inline on the component

---

## 3. `commons/` — cross-component code and escape-hatch modules

`commons/` sits at the project root, a **sibling** of `components/`. The `components/*/index.tsx` discovery glob does not descend into `commons/`, so anything there is safely invisible to slot registration — no matter how it's shaped.

Two use cases:

1. **Shared code** — anything a second component would need to import: an icon (`commons/icons/CheckIcon.tsx`), a formatter (`commons/utils/formatCpf.ts`), a hook (`commons/hooks/useCartSummary.ts`), a domain type (`commons/types/PaymentTerm.ts`). Slot components import via `../../commons/<subdir>/<Name>`.

2. **Escape-hatch for component-shaped modules** — a folder with an `index.tsx` that looks like a slot component but must NOT be registered. Use this when two components would otherwise fight over the same slot: render one inside the other and demote the inner one to `commons/<ComponentLike>/`. Because `commons/` is outside the discovery glob, the inner one stays a plain React component the outer one imports.

### Icon example

```tsx
// commons/icons/TrashIcon.tsx

/** Trash / delete icon used in cart-item remove buttons and address list rows. */
export function TrashIcon({ size = 16 }: { size?: number }) {
  return (
    <svg width={size} height={size} viewBox="0 0 16 16" fill="none" aria-hidden="true">
      <path
        d="M2.5 4h11M6.7 4V2.7h2.6V4M3.8 4l.5 9.3a1 1 0 0 0 1 .9h5.4a1 1 0 0 0 1-.9l.5-9.3"
        stroke="currentColor"
        strokeWidth="1.3"
        strokeLinecap="round"
        strokeLinejoin="round"
      />
    </svg>
  );
}
```

### Rules

- **No inline `<svg>` inside a component's `index.tsx`.** Repeated icons get duplicated across components, the JSX becomes harder to scan, and renaming means editing N files. Import from `commons/icons/`.
- **No Unicode emoji glyphs as icons.** Characters like `🗑`, `✕`, `📷` render differently per OS, browser, and font fallback — the customer sees a different shape than the designer drew, and rendering can even differ between Chrome and Firefox on the same machine. Always use an inline SVG from `commons/icons/`.
- **Don't pre-share single-consumer code.** If only one component uses a helper today, keep it inside that component's folder. Move to `commons/` when a second consumer appears — never before.

---

## 4. Quick decision table

| I'm about to add… | Where does it go? |
|---|---|
| A subcomponent, and this is my first extraction | `<SubName>.tsx` at the component's folder root |
| A subcomponent, and there are already ≥ 1 sibling `<Name>.tsx` | Create `components/<SubName>.tsx` (move the existing one too) |
| A pure helper, single-component use, and it's the 1st | Inline in `index.tsx` |
| A pure helper, single-component use, and it's the 2nd, 3rd, or 4th | `utils.ts` |
| A pure helper, single-component use, and it's the 5th+ | `utils/<helperName>.ts` with a `utils/index.ts` barrel |
| A hook, single-component use | `hooks/use<Name>.ts` (subfolder, from the first hook) |
| A hook, cross-component use | `commons/hooks/use<Name>.ts` |
| An icon | `commons/icons/<Name>.tsx` |
| A helper a second component needs | `commons/utils/<name>.ts` (or `commons/types/`, etc.) |
| A component-like module that should NOT be a slot | `commons/<Name>/index.tsx` |
| Custom domain types or non-trivial Props | `types.ts` in the component folder |
| Types touched by 2+ components | `commons/types/<Name>.ts` |
