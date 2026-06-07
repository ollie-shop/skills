# Cart Item

## What it is

A single row in the cart's item list. Renders one product with image, name, SKU info, quantity controls, price, and remove action. The template iterates the cart and renders this slot once per item.

## Props the slot injects (this is the entire contract)

> **The slot passes exactly these three props. Nothing else.** Everything else you need — price extras, prescription/subscription state, addons, platform-specific fields, pending-action flags, promotional labels, lab-discount eligibility — **must be derived inside the component** from `rawSession`, `useCheckoutSession()`, `usePendingActions()`, or other hooks. Adding props like `prescription`, `subscription`, `labDiscount`, `unitPriceCents`, or `pendingQuantity` to the component signature is the single most common mistake; the slot does not pass them and they arrive `undefined` in production.

| Prop | Type | Notes |
|---|---|---|
| `item` | `CartItem` | The parsed cart item: `id`, `name`, `quantity`, `price`, `originalPrice`, `available`, `index`, `image`, `url`. |
| `onQuantityChange?` | `(quantity: number) => void` | Emit when the user changes quantity. The parent dispatches the SDK action. |
| `onRemoveItem?` | `(index: number) => void` | Emit when the user clicks remove. The parent dispatches the SDK action. |

## Slot(s)

- **Primary:** `cart_item` — the canonical per-item slot. The template iterates the cart and renders this slot once per item.

## Behavior

1. Render the product image (clickable, goes to PDP via `item.url`).
2. Render product name as a link to the PDP. Open in a new tab with `rel="noreferrer"`.
3. Render SKU info under the name (e.g. `SKU: <id>`).
4. Quantity controls: hold the typed value in optimistic local state so the input feels instant. On change and on blur, clamp to a minimum of `1` and call `onQuantityChange`.
5. Price block: show unit price with strike-through original price when discounted, plus line total. Currency and locale come from `session.locale`.
6. Remove button: on click, call `onRemoveItem(item.index)`.

## States to cover

- **Available** — full controls, normal price.
- **Unavailable** — items returned in an unavailable group don't get quantity controls; show muted styling and skip the price block.
- **Discounted** — strike-through original price beside the discounted price.

## Visual structure (semantic only)

- Two-column row on desktop: image on the left, info block on the right.
- Inside the info block: name (link) on top, SKU below, controls + price aligned to the right edge.
- Quantity controls: `<button>` decrement, `<input type="number">` with `min="1"`, `<button>` increment. All same line.
- On mobile, the row collapses to: image at the top-left, name + SKU on the top-right, controls + price as a separate row below the info block.
- Remove control as a `<button>` (icon + text). Don't rely on icon alone.

## SDK touchpoints

- `useCheckoutSession()` — read `session.locale` for currency and locale formatting. Drop down to `rawSession` only when you need a platform-specific extra (e.g. attachments, fulfillment hints) that isn't on `session.cartItems[]`.
- Quantity and remove **do not** dispatch SDK actions from inside this component. The parent that uses `cart_item` is responsible for calling `useCheckoutAction("UPDATE_ITEMS_QUANTITY")` / `useCheckoutAction("REMOVE_ITEMS")` after receiving the callback.

## Without a design reference

- Two-column flex on desktop (`display: flex; gap: ...`); single column on mobile via media query.
- Image wrapped in a fixed-aspect container so layout doesn't shift while the image loads.
- Name uses `<a target="_blank" rel="noreferrer">`. Quantity controls are real `<button>` and `<input type="number">`.
- Strike-through for original price via `<s>` (semantic), not a `text-decoration: line-through` div.
- Use CSS variables for colors (`--color-fg`, `--color-fg-muted`, `--color-primary`). No hardcoded brand colors.
- Hover/focus visible states on every interactive element.

## Antipatterns

- **Don't dispatch checkout actions from inside the component.** Emit `onQuantityChange` / `onRemoveItem` and let the parent decide. Otherwise the row can't be reused in other lists (summaries, unavailable groups, etc.).
- **Don't read the cart from `session.cartItems` to render this row.** The slot passes you `item`; using the array directly causes double-render and bugs when items reorder.
- **Don't hardcode `Intl.NumberFormat("en-US", { currency: "USD" })`.** Derive locale + currency code from `session.locale`. Multi-region stores ship the same component.
- **Don't write quantity directly to the SDK on every keystroke.** Keep an optimistic local value, debounce or commit on blur — otherwise the user can't type "12" without "1" firing first.
