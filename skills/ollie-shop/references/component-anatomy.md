# Component Anatomy

Minimum file layout for a custom checkout component on Ollie Shop. This is the structure that `ollieshop start` discovers and bundles.

## Layout

```
./components/<Name>/
├── index.tsx              # required — default-exports the React component
├── index.module.css       # optional — CSS module imported by index.tsx
└── meta.json              # required IF the project uses per-folder manifests
                           # (skip when the project registers via ollie.json — see below)
```

> The historical convention was `styles.module.css`; the current convention is `index.module.css` (matches the entry file's name, groups the folder alphabetically). Both still work at the bundler level — use `index.module.css` for new code.

- The **folder name** is the canonical name. `ollieshop start` and `ollieshop deploy --name <Name>` both look up the component by this folder name.
- `index.tsx` **must default-export** a React function. Component props are whatever you declare in the function signature; the slot supplies them (defaults come from the component record's `props` JSON in the database).
- A folder that isn't linked to a database record (via either mechanism below) is still built by `ollieshop start` but flagged **unlinked** — visible in Studio, not placed into a slot in the live checkout. Useful for prototyping before you run `ollieshop component create`.

### Two registration patterns (both supported)

A project picks ONE of these to link local folders to their database records. Don't mix them within the same project.

**A. Per-folder `meta.json`** (the original model)

Each component folder carries its own `meta.json` with the component `id` (and optional `slot` for unlinked previews):

```json
// ./components/FreeShippingBar/meta.json
{ "id": "02efb84b-7609-419f-9a93-86011865776d" }
```

Setting `"slot": "<slot-id>"` with `"id": null` lets Studio render the component into that slot for preview purposes.

**B. Root `ollie.json` `components` map** (CLI ≥ 1.5.0)

The root `ollie.json` keys each component UUID to `{ path, slot }` and no per-folder manifest is needed:

```jsonc
// ./ollie.json (root)
{
  "storeId": "b70c7b24-cedb-48d3-bc33-e44f54fb6dc6",
  "components": {
    "02efb84b-7609-419f-9a93-86011865776d": {
      "path": "components/FreeShippingBar",
      "slot": "cart_header_full_page"
    }
  }
}
```

If you inherit an older project that uses (A) and want to migrate to (B), run `ollieshop setup` — it scans the folders, writes each `meta.json` into the config map, and deletes the old files (`--dry-run` to preview). Metas with no `id` (unlinked) are left as-is.

## Minimal example

```tsx
// ./components/FreeShippingBar/index.tsx
import { useCheckoutSession } from "@ollie-shop/sdk";
import styles from "./index.module.css";

export default function FreeShippingBar({ thresholdCents = 19900 }: { thresholdCents?: number }) {
  const { session } = useCheckoutSession();
  // session.totals.items is the items subtotal in minor units (cents).
  const remainingCents = Math.max(0, thresholdCents - session.totals.items);

  if (remainingCents === 0) {
    return <div className={styles.freeShippingBarUnlocked}>Free shipping unlocked!</div>;
  }

  const formatted = new Intl.NumberFormat(session.locale.language, {
    style: "currency",
    currency: session.locale.currency,
  }).format(remainingCents / 100);

  return (
    <div className={styles.freeShippingBarRoot}>Add {formatted} to unlock free shipping.</div>
  );
}
```

```css
/* ./components/FreeShippingBar/index.module.css */
.freeShippingBarRoot {
  padding: 0.75rem 1rem;
  background: var(--color-base-200);
  border-radius: var(--radius-box);
  font: inherit;
}
.freeShippingBarUnlocked {
  padding: 0.75rem 1rem;
  background: var(--color-alert-success);
  color: var(--color-alert-success-content);
  border-radius: var(--radius-box);
  font: inherit;
}
```

The example above assumes pattern (B) — the root `ollie.json` `components` map. If the project uses pattern (A), the equivalent is a `./components/FreeShippingBar/meta.json` file with `{ "id": "02efb84b-7609-419f-9a93-86011865776d" }` and no `components` entry in `ollie.json`.

## Dev loop with `ollieshop start`

1. Run `ollieshop start` in the project root (the one containing `ollie.json` + `components/`).
2. The CLI discovers every `./components/*/index.tsx`, bundles them to `node_modules/.ollie/build/<Name>/index.js`, and serves them on port 4000.
3. Ollie Studio opens in your browser, iframing the live checkout for the configured store + version with your local bundles replacing the deployed ones.
4. Edit `index.tsx` or `index.module.css` — esbuild re-bundles, Studio receives a build event over SSE (`/esbuild`), and the iframe reloads.
5. When you're happy with the result, ship from Studio's preview UI (see "Deploying" in `cli-reference.md`).

If the folder isn't linked yet (just exploring a new component idea), `ollieshop start` still works — the component shows up in Studio as unlinked. Link it (via `meta.json` under A, or the `components` map under B) once you've run `ollieshop component create` and have a real id.

## Stage-specific overrides

Stage variants apply to `ollie.json` only:

- `ollie.dev.json` — picked up when you run `ollieshop start --stage dev`; falls back to `ollie.json` if absent. Under pattern B, each stage's config carries its own `components` map.
- Pattern A's `meta.json` has no stage variant in practice — one folder, one manifest, regardless of stage.

Use stages to target different stores/versions from one project (e.g. a sandbox store for previews vs. production).

## Bundling rules to know

- The bundle is **CommonJS, browser target, ES2020**. Async/await is fine.
- These packages are marked `external` and provided by the host at runtime — do not bundle them, do not duplicate their state:
  - `react`, `react-dom`
  - `next`, `next-intl`
  - `@ollie-shop/sdk`
- Source map is generated alongside the JS bundle.
- CSS in `index.module.css` is treated as a CSS module (locally scoped class names).
- If your component imports static assets (images, fonts), keep them small — the bundle is shipped to the browser on every checkout load.

## When to split files

Full rules — subcomponent extraction triggers and locations, utils granularity (1 / 2–4 / 5+), hooks scoping (component-scoped vs cross-component), types — live in **[`file-organization.md`](file-organization.md) § 2**. That is the single source of truth for anything about layout inside a component folder.

One constraint that belongs here (structural / bundler): avoid sub-routing, server components, or build-time data fetching. The bundle runs as a client-side React component inside the checkout.

## Shared code in `commons/`

The `commons/` folder — layout, two use cases (shared code + component-shaped escape-hatch), icon example, no-inline-`<svg>` / no-emoji rules — lives in **[`file-organization.md`](file-organization.md) § 3**.
