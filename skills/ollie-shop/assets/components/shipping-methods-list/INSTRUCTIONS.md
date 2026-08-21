# Shipping Methods List

## What it is

The delivery method picker shown inside a single shipping package. Lists every quote that can deliver all the package's items, lets the customer pick one, and persists the choice to the session.

## Slot(s)

- **Primary:** `shipping_package_shipping_options` — rendered once per delivery package on the shipping step. The template passes the package and item count as props.
- Do not use this for pickup; pickup uses a different selector. If the request is about pickup, flag it back to the user — there's no library entry for that yet.

## Props the slot provides

- `selectedPackage` — the current package (id, type, items as indexes, current quote id, address id, store name, timeSlot).
- `itemsCountInPackage` — number of items inside this package, useful when showing "X items will ship" copy.

## Behavior

1. Read `session.shipping.availableQuotes`. Keep only quotes that (a) have `type === "delivery"` and (b) cover every item in `selectedPackage.items` (every item index appears in the quote's `availableItems`).
2. Determine the currently selected quote by matching `selectedPackage.id` against the filtered quotes.
3. Render one card per filtered quote, with the current selection visually distinguished.
4. On click, call `useCheckoutAction("UPDATE_SHIPPING_PACKAGES")` with the **full delivery-packages payload** (see Antipatterns below) and put the targeted card into a loading state.
5. While the action is pending: disable further clicks across the list, keep the previous selection visible.
6. On success: confirm the new selection and clear the per-card loading state. The session is updated by `useCheckoutAction` itself — do not call `revalidate()`.
7. On error: clear loading, leave the previously selected card as selected, and surface an error via `useMessages` (the SDK auto-resolves `ServerError` to translated copy).

## States to cover

- **Idle, no selection** — no quote id matches the package yet; nothing highlighted.
- **Idle, with selection** — one card visually marked as selected.
- **Loading (per card)** — show a spinner on the card the user just clicked; the rest of the list is non-interactive but stays mounted.
- **Error** — keep the prior selection, push a `useMessages` error; do not block the list permanently.
- **Empty** — `availableQuotes` came back filtered to zero. Show short copy explaining no delivery option covers all items and a CTA to review the cart/address. Do not render an empty list silently.

## Visual structure (semantic only)

- A vertical list (`<ul>` of `<li>` rows) wrapping a list of selectable cards.
- Each card is a `<button type="button">` so keyboard activation works.
- Per card, two semantic rows:
  - Top row, two columns: estimated arrival on the left, price on the right.
  - Bottom row: method name in a less-prominent treatment.
- Selected card uses a stronger border (theme color via CSS variables). Do not rely on color alone — also set `aria-pressed="true"` on the button.
- On narrow viewports, the estimated arrival uses a shorter format (e.g. `Est. Wed, Oct 1`); on wider viewports use the long format (e.g. `Estimated Arrival, Wednesday, October 1st`). Pick the format via CSS media query, not via JS measuring the viewport.

## SDK touchpoints

- `useCheckoutSession()` — read `session.shipping.availableQuotes` and `session.shipping.packages`.
- `useCheckoutAction("UPDATE_SHIPPING_PACKAGES", { onSuccess, onError })` — dispatch the selection. Use `onSuccess` to clear the per-card loading state; use `onError` to clear loading and push the error via `useMessages`.
- `useMessages()` — surface a translated error in `onError`.

## Without a design reference

If the user has no Figma / screenshot:

- Use neutral CSS-modules markup: `<ul>` of `<li><button>` rows. No third-party UI kit.
- Style with the host's CSS variables for color (`--color-primary`, `--surface`, etc.). Never hardcode brand colors.
- Mobile-first; two-column top row uses `display: flex; justify-content: space-between;`.
- Selected state: 2px border in `var(--color-primary)`, `aria-pressed="true"`. Resting state: 1px neutral border at low opacity.
- Currency: derive locale and currency code from `session.locale`. Do not hardcode `"en-US"` / `"USD"` — see Antipatterns.
- Date formatting: use `Intl.DateTimeFormat` with the session locale; choose `weekday: "long"` for desktop and `weekday: "short"` for mobile, both gated by CSS media queries.

## Antipatterns

- **Don't send only the selected package in `UPDATE_SHIPPING_PACKAGES`.** Send the full list of delivery packages (with each package's `items` array), updating only the targeted package's `id`. Sending only one package makes the SDK's merge loop reprocess sibling packages by SLA id and duplicate them in the UI. This is a real bug fixed in production — preserve the workaround.
- **Don't hardcode `Intl.NumberFormat("en-US", { currency: "USD" })`.** Read locale + currency from `session.locale`. Different stores ship the same component with different currencies.
- **Don't use `<div onClick>` or `<a>` for the cards.** Use `<button type="button">` so keyboard and screen readers work without extra ARIA.
- **Don't use JS `window.innerWidth` checks to switch desktop/mobile date formats.** Use CSS media queries — JS measurement breaks SSR and causes layout shift.
- **Don't render an empty list silently** when filtered quotes is empty. Show explanatory copy; otherwise the user thinks the page is broken.
- **Don't call `revalidate()` after dispatching an action.** `useCheckoutAction` already updates the session on success. Calling `revalidate()` triggers a redundant round-trip and can race with the action's own update.
