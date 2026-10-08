# Pattern: Skipping optional content on slow connections

Recommendation shelves, upsell strips and any other **optional** block on the cart are the
heaviest thing a custom component adds: a catalog search, a grid of product images, a
skeleton. On a slow link that traffic competes with the checkout itself, and the shopper
pays for it with a slower cart for content they never asked for.

The fix is to **not mount the optional block at all** when the browser says the shopper is
on a slow connection or asked to save data. This is [adaptive
loading](https://web.dev/articles/adaptive-loading-cds-2019): serve the lighter experience
to the shoppers who need it, the full one to everyone else.

This is a best practice for every store, for every optional block on the cart page. The
rules below come from a production cart that shipped the gate twice — the first version
flickered, and the second one is what this page documents.

---

## The rule

Skip the optional content when **either** is true:

- `navigator.connection.saveData === true` — the shopper turned on data saver.
- `navigator.connection.effectiveType` is `"2g"` or `"slow-2g"`.

Otherwise render it. If there is no `navigator.connection`, **treat the shopper as not slow**.

`navigator.connection` is the [Network Information
API](https://developer.mozilla.org/en-US/docs/Web/API/Network_Information_API). It is a
WICG draft with limited availability: Chromium browsers have it, Safari, every iOS browser
and Firefox do not. `effectiveType` is also a coarse estimate, not a measurement. So this is
**progressive enhancement** — the gate helps the shoppers whose browser can tell you, and
changes nothing for the rest. Never make the cart depend on it.

---

## The four rules

### 1. Decide once, after mount — never during render

The checkout server-renders slots, and there is no `navigator` on the server. A check made
during render therefore answers "not slow" in the HTML, ships the shelf skeleton, and then
disagrees with itself on hydration: the skeleton appears, then is pulled. That is a flicker
and a hydration mismatch.

Keep the verdict in state that starts **undecided**, and set it from `useEffect`. While it
is undecided, render nothing:

| State | Meaning | Renders |
|---|---|---|
| `undefined` | not decided yet (server, first paint) | nothing |
| `null` | decided, connection is fine | the shelf |
| `"saveData"` / `"2g"` | decided, skip | nothing |

Using `useMemo` (or a plain call) in the render body is the version that flickers.

### 2. Gate above the shelf, not inside it

If the shelf component decides for itself, it has already mounted — and its fetch, its
images and its skeleton have already started. Put the check in the **parent**, or in a thin
default export that renders the real shelf only once the verdict is "fine". A slow
connection then costs zero requests.

When the shelf owns its own slot, the default export is the gate and the shelf is an
extracted subcomponent (see `file-organization.md` § 2).

### 3. Gate optional content only

Never apply this to a required checkout step, the totals, the coupon form, or anything the
shopper needs to finish the order. If the content is not optional, it does not get a gate.

### 4. Do not use page-load timing as a signal

First Contentful Paint, `performance.timing`, time-to-first-byte and similar are **not**
slow-shopper signals. They mostly measure slow servers and redirects: measured on a live
store, FCP was 80–90% server time, and even a fast, unthrottled host showed 2–4 seconds of
streamed server rendering. A paint-based rule flags the store, not the shopper — and since
the paint can land after the shelf has mounted, it makes the shelf appear and then vanish.
`saveData` and `effectiveType` exist at mount, so the verdict never changes under the shopper.

---

## Implementation

The classifier (`commons/slowConnection.ts`) — a plain module, no React:

```ts
/**
 * Decides whether optional cart content should stay off: data saver on, or a 2g link.
 * Safari, iOS and Firefox expose no connection, so they always get the content.
 */
type Connection = { saveData?: boolean; effectiveType?: string };

export type SlowReason = "saveData" | "2g";

const LOG_TAG = "[cart-items]";

export function classifyConnection(
  connection: Connection | undefined,
): SlowReason | null {
  if (connection?.saveData) return "saveData";
  if (
    connection?.effectiveType === "2g" ||
    connection?.effectiveType === "slow-2g"
  )
    return "2g";
  return null;
}

let logged = false;

export function slowConnectionReason(): SlowReason | null {
  if (typeof navigator === "undefined") return null;
  const connection = (navigator as Navigator & { connection?: Connection })
    .connection;
  const reason = classifyConnection(connection);
  if (reason && !logged) {
    logged = true;
    console.info(`${LOG_TAG} skipped-slow-connection`, {
      reason,
      effectiveType: connection?.effectiveType ?? null,
    });
  }
  return reason;
}
```

The gate (`components/CartItems/index.tsx`), in the component that owns the slot. The state
starts `undefined` and the effect runs once, after mount:

```tsx
/**
 * Owns the `cart_items` slot: the item list, then the recommendations shelf
 * when the shopper is not on a slow connection.
 */
import { useEffect, useState, type ReactNode } from "react";
import RecommendationsShelf from "../RecommendationsShelf";
import { slowConnectionReason, type SlowReason } from "../../commons/slowConnection";

export default function CartItems({ children }: { children?: ReactNode }) {
  const [slowReason, setSlowReason] = useState<SlowReason | null>();
  useEffect(() => setSlowReason(slowConnectionReason()), []);

  return (
    <div>
      {children}
      {slowReason === null ? <RecommendationsShelf /> : null}
    </div>
  );
}
```

`slowReason === null` is deliberate: `undefined` (not decided) and a reason string (skip) both
render nothing, and only a decided "fine" mounts the shelf. Do not simplify it to
`!slowReason` — that mounts the shelf during the undecided first paint, which is rule 1's
bug.

Notes on the pieces:

- **The `logged` latch.** The checkout re-evaluates a slot bundle many times while the
  session hydrates (16 times in ~55 ms in production), and each evaluation gets a fresh
  module-level flag, so expect ~16 skip lines per page load. Count skips per session, not per
  line.
- **`console.info`, not `console.log`.** The skill's observability rules forbid `console.log`
  in shipped code; the `[component-name]` prefix keeps the line greppable and countable (for
  example in PostHog).
- **The `Navigator` cast.** `connection` is not in the DOM typings, so the cast is the
  type-safe way to read it. This is a browser API, not an SDK helper, so it does not conflict
  with `sdk-guide.md` § Known-absent APIs.

---

## Checklist

- [ ] The verdict starts `undefined` and is set from `useEffect` — no `useMemo` or direct call in render
- [ ] While undecided, the optional block renders nothing (no skeleton, no placeholder)
- [ ] The check lives above the shelf: on a slow connection the shelf never mounts — no fetch, no images
- [ ] Only `saveData` and `effectiveType` (`"2g"` / `"slow-2g"`) are read — no FCP, no page timing
- [ ] A missing `navigator.connection` means "not slow"
- [ ] Only optional content is gated — never a required step, totals or the coupon form
- [ ] The skip is logged once with its `reason` (`skipped-slow-connection`)

---

## Related

- [component-authoring-cheatsheet.md](../component-authoring-cheatsheet.md) § 6 — loading, error and empty states
- [slots-catalog.md](../slots-catalog.md) — `cart_items`, `cart_item_addons` and the other cart-page slots
- [file-organization.md](../file-organization.md) — where `commons/` helpers and extracted subcomponents live
