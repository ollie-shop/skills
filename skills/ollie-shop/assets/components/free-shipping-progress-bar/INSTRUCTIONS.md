# Free Shipping Progress Bar

## What it is

A persistent indicator that shows how far the customer is from unlocking free shipping. Renders a progress bar plus a short message ("R$ 49,90 to go" / "You unlocked free shipping"). Visible on the cart and (optionally) on the checkout sidebar.

## Slot(s)

- **Primary:** `cart_sidebar_totalizer` (sidebar variant), `checkout_sidebar_totalizer` (carries over into the checkout steps). Use the alternatives when the store wants the bar to follow the customer through the flow.
- **Alternative:** `cart_header_full_page` — the canonical position above the cart content.

## Behavior

1. Read the cart subtotal (items total, **excluding** shipping and discounts that aren't shipping-related). Source it from `session.totals` when the field is available; otherwise drop to the platform-specific totalizer in `rawSession` for the equivalent value.
2. Compare the subtotal against a `threshold` value (passed as a prop; default chosen by the store).
3. If `subtotal >= threshold`: render the unlocked state — checkmark + "You unlocked free shipping" copy.
4. Otherwise: render the progress bar at `(subtotal / threshold) * 100`% capped at 100, plus copy showing the remaining amount formatted in the session's currency.
5. Smoothly transition between the "remaining" copy and the "unlocked" copy — the unlock moment is the payoff.

## States to cover

- **Below threshold** — bar filled proportionally, remaining-amount copy.
- **At/above threshold (unlocked)** — bar full, success copy + icon.
- **Zero cart** — bar empty, remaining-amount copy still meaningful.
- **Threshold not configured** — render nothing (don't show a confused progress bar).

## Visual structure (semantic only)

- Outer block with two stacked rows: copy line on top, progress bar below.
- Progress bar is a `<div role="progressbar" aria-valuenow aria-valuemin="0" aria-valuemax="100">` wrapping a filled child whose width animates from the prior value to the new one.
- Use a visual icon next to the success copy (checkmark or shipping glyph). Don't replace the copy with only an icon — leave the text for screen readers.
- Promo / discount list (when present) sits below the bar in a separate semantic section.

## SDK touchpoints

- `useCheckoutSession()` — read `session.totals` (or `rawSession` totalizers as the platform-specific fallback) and `session.locale` for currency formatting.

## Without a design reference

- Use CSS variables: `--color-primary` for the filled portion and the unlock copy, `--surface-subtle` for the empty track.
- Render the progress bar with a CSS transition on `width` so the fill animates smoothly when the cart changes.
- Layout: vertical flex, gap between rows. Center the copy + icon on the cross axis.
- Currency formatting via `Intl.NumberFormat(session.locale, { style: "currency", currency: session.locale.currencyCode })` — never a hardcoded locale.

## Antipatterns

- **Don't hardcode `Intl.NumberFormat("en-US", { currency: "USD" })`.** Multi-region stores will ship this component as-is.
- **Don't read the threshold from a hardcoded constant.** Pass it as a prop; the store sets the value via the component record's `props` JSON.
- **Don't compute the threshold check against `session.totals.total`.** Total includes shipping and discounts; the threshold gate is supposed to check the *items* subtotal. Use the items subtotal totalizer.
- **Don't render a progress bar when `threshold` is `undefined` or `<= 0`.** It looks broken. Either render nothing or render a static "Free shipping" badge.
- **Don't use inline JS measurements (`window.innerWidth`) to switch between desktop/mobile copy.** Use CSS media queries; JS breaks SSR.
- **Don't call `revalidate()` to refresh after a cart change.** The session updates on its own when an action mutates it.
