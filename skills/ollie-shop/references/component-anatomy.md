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
- `meta.json` carries `id` (the component UUID returned by `ollieshop component create`) and optional `slot` for unlinked previews. The `id` field is what tells the host that this local folder corresponds to a real component in the database. Without it, `ollieshop start` builds the folder anyway but flags it as **unlinked** (visible in Studio, not placed into a slot in the live checkout).

## Minimal example

```tsx
// ./components/FreeShippingBar/index.tsx
import { useCheckoutSession } from "@ollie-shop/sdk";
import styles from "./styles.module.css";

export default function FreeShippingBar({ threshold = 199 }: { threshold?: number }) {
  const { session } = useCheckoutSession();
  const subtotal = session.totals?.subtotal ?? 0;
  const remaining = Math.max(0, threshold - subtotal);

  if (remaining === 0) {
    return <div className={styles.unlocked}>Free shipping unlocked!</div>;
  }
  return (
    <div className={styles.bar}>
      Add R$ {remaining.toFixed(2)} to unlock free shipping.
    </div>
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
