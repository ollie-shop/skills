# Component Anatomy

Minimum file layout for a custom checkout component on Ollie Shop. This is the structure that `ollieshop start` discovers and bundles.

## Layout

```
./components/<Name>/
├── index.tsx              # required — default-exports the React component
├── styles.module.css      # optional — CSS module imported by index.tsx
└── meta.json              # required for deploy — links the folder to a DB row
```

- The **folder name** is the canonical name. `ollieshop start` and `ollieshop deploy --name <Name>` both look up the component by this folder name.
- `index.tsx` **must default-export** a React function. Component props are whatever you declare in the function signature; the slot supplies them (defaults come from the component record's `props` JSON in the database).
- `meta.json` carries `id` (the component UUID returned by `ollieshop component create`) and optional `slot` for unlinked previews. The `id` field is what tells the host that this local folder corresponds to a real component in the database. Without it, `ollieshop start` builds the folder anyway but flags it as **unlinked** (visible in Studio, not placed into a slot in the live checkout). Setting `"slot": "<slot-id>"` with `"id": null` lets Studio render the component into that slot for preview purposes — useful for prototyping before running `ollieshop component create`.

## Minimal example

```tsx
// ./components/FreeShippingBar/index.tsx
import { useCheckoutSession } from "@ollie-shop/sdk";
import styles from "./styles.module.css";

export default function FreeShippingBar({ thresholdCents = 19900 }: { thresholdCents?: number }) {
  const { session } = useCheckoutSession();
  // session.totals.items is the items subtotal in minor units (cents).
  const remainingCents = Math.max(0, thresholdCents - session.totals.items);

  if (remainingCents === 0) {
    return <div className={styles.unlocked}>Free shipping unlocked!</div>;
  }

  const formatted = new Intl.NumberFormat(session.locale.language, {
    style: "currency",
    currency: session.locale.currency,
  }).format(remainingCents / 100);

  return (
    <div className={styles.bar}>Add {formatted} to unlock free shipping.</div>
  );
}
```

```css
/* ./components/FreeShippingBar/styles.module.css */
.bar {
  padding: 12px 16px;
  background: var(--surface-subtle, #f4f4f5);
  border-radius: 8px;
}
.unlocked {
  padding: 12px 16px;
  background: var(--success-bg, #ecfdf5);
  color: var(--success-fg, #065f46);
}
```

```json
// ./components/FreeShippingBar/meta.json
{
  "id": "02efb84b-7609-419f-9a93-86011865776d"
}
```

## Dev loop with `ollieshop start`

1. Run `ollieshop start` in the project root (the one containing `ollie.json` + `components/`).
2. The CLI discovers every `./components/*/index.tsx`, bundles them to `node_modules/.ollie/build/<Name>/index.js`, and serves them on port 4000.
3. Ollie Studio opens in your browser, iframing the live checkout for the configured store + version with your local bundles replacing the deployed ones.
4. Edit `index.tsx` or `styles.module.css` — esbuild re-bundles, Studio receives a build event over SSE (`/esbuild`), and the iframe reloads.
5. When you're happy with the result, ship from Studio's preview UI (see "Deploying" in `cli-reference.md`).

If you don't have a `meta.json` yet (just exploring a new component idea), `ollieshop start` still works — the component shows up in Studio as `studio-<Name>`. Add `meta.json` once you've run `ollieshop component create` and have a real id.

## Stage-specific overrides

Both `ollie.json` and `meta.json` can have stage variants:

- `ollie.dev.json` — picked up when you run `ollieshop start --stage dev`.
- `meta.dev.json` — picked up for that same stage, falling back to `meta.json` if absent.

Use stages to target different stores/versions from one project (e.g. a sandbox store for previews vs. production).

## Bundling rules to know

- The bundle is **CommonJS, browser target, ES2020**. Async/await is fine.
- These packages are marked `external` and provided by the host at runtime — do not bundle them, do not duplicate their state:
  - `react`, `react-dom`
  - `next`, `next-intl`
  - `@ollie-shop/sdk`
- Source map is generated alongside the JS bundle.
- CSS in `styles.module.css` is treated as a CSS module (locally scoped class names).
- If your component imports static assets (images, fonts), keep them small — the bundle is shipped to the browser on every checkout load.

## When to split files

Default to a single `index.tsx`. Split when:
- The component has more than ~150 lines of JSX/logic.
- You're reusing helpers across components in the same project — extract a `./components/<Name>/utils.ts` or share via a sibling folder.

Avoid sub-routing, server components, or build-time data fetching. The bundle runs as a client-side React component inside the checkout.

## Shared assets — icons & utils

When something is used by more than one component (icons, formatters, helper hooks, types), put it in a shared directory at the project root and import from there. Don't inline these things into each component's `index.tsx`.

```
./
├── icons/
│   ├── TrashIcon.tsx
│   ├── UploadIcon.tsx
│   └── CheckIcon.tsx
├── utils/
│   ├── formatPhone.ts
│   └── parseCpf.ts
└── components/
    ├── ProductCard/
    │   └── index.tsx              # imports `../../icons/TrashIcon`
    └── ShippingPicker/
        └── index.tsx              # also imports `../../icons/TrashIcon`
```

Each icon is a tiny React component exporting an inline `<svg>`:

```tsx
// ./icons/TrashIcon.tsx
export function TrashIcon({ size = 16 }: { size?: number }) {
  return (
    <svg width={size} height={size} viewBox="0 0 16 16" fill="none" aria-hidden="true">
      <path d="M2.5 4h11M6.7 4V2.7h2.6V4M3.8 4l.5 9.3a1 1 0 0 0 1 .9h5.4a1 1 0 0 0 1-.9l.5-9.3" stroke="currentColor" strokeWidth="1.3" strokeLinecap="round" strokeLinejoin="round" />
    </svg>
  );
}
```

Two rules to keep:

- **No inline `<svg>` inside `index.tsx`.** Repeated icons get duplicated across components, the JSX of each component becomes harder to scan, and renaming the icon means editing N files. Importing from `./icons/` keeps each component focused on its own job.
- **No Unicode emoji glyphs as icons.** Characters like `🗑`, `✕`, `📷` render differently per OS, browser, and font fallback — the customer sees a different shape than the designer drew, and the rendering may even change between Chrome and Firefox on the same machine. Always use an inline SVG (from the shared `./icons/` folder).

Same pattern applies to formatters, hooks, types, and any other code that two or more components touch — put it in `./utils/`, `./hooks/`, or `./types/` at the project root and import it. Anything used by only one component stays inside that component's folder.
