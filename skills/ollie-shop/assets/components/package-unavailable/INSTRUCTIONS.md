# Package Unavailable

## What it is

A package-level slot that renders for packages whose items the platform considers unavailable. Use it to **customize what fills that spot** — copy, layout, actions, or any extra affordances the store wants to surface beyond the platform's default render.

The shape of the customization depends on what the store needs. Three common shapes, from most to least common:

- **Shape 1 — Visual restyle.** Wrap or replace the default render (`children`) with your own container, copy, color scheme, or layout. No SDK mutations involved. This is the typical case when the brief is "make the unavailable block look like our brand".
- **Shape 2 — Action-rich.** On top of a restyle, surface store-specific actions on the package or per item (remove all, contact support, save for later, etc.). Uses `useCheckoutAction("REMOVE_ITEMS")` and similar.
- **Shape 3 — Condition-gated** (advanced). Pair with a `delivery-restriction` function (`assets/functions/delivery-restriction/INSTRUCTIONS.md`) that flags items as blocked by a customer-side condition (missing attachment, unverified profile, absent identifier, etc.). The component reads the function's snapshot flags and surfaces affordances to satisfy the condition, switch to an allowed option, or remove the blocked items. Without the paired function, this shape is unreachable — the snapshot flags will be absent and the component should fall through to `children`.

Pick the shape from the brief; do not jump straight to Shape 3 just because the slot has "unavailable" in its name. Most customers reaching this slot want Shape 1 or 2.

## Props the slot injects (this is the entire contract)

> **The slot passes exactly one prop: `children`** — the platform's default unavailable-pack render. Everything else this component needs (items, logistics info, snapshot flags) must be derived from `useCheckoutSession()`. Do NOT add `items`, `blockedItems`, `condition`, or similar to the component signature; the slot does not pass them and they arrive `undefined` in production.

| Prop | Type | Notes |
|---|---|---|
| `children` | `ReactNode` | The platform's default render for this unavailable package. Forward it when your customization doesn't replace the item visual. |

## Slot(s)

- **Primary:** `shipping_package_unavailable` — rendered once per package that has at least one unavailable item, regardless of why it's unavailable.

## Behavior

### Shape 1 — Visual restyle

Wrap or replace `children`. If you need to enumerate the items the package contains, read `session.cartItems` — it carries the parsed item data (name, image, price, quantity, index) in a platform-agnostic shape. Render your custom container, copy, and styling. No SDK action calls.

### Shape 2 — Action-rich (extends Shape 1)

On top of the restyle, surface actions:

- **Remove all / remove one** — `useCheckoutAction("REMOVE_ITEMS")` with the relevant indexes.
- **Contact support / open chat / save for later** — calls out to store-specific systems (CRM widget, link to a help page, persistent storage). These are outside the SDK.

Use `useMessages` to confirm success and surface failure on every action.

### Shape 3 — Condition-gated (paired with `delivery-restriction`)

Used when the store also runs a `delivery-restriction` function that writes per-item snapshot flags. The component reads the flags to distinguish:

- Items the function **blocked by a customer-side condition** (e.g. `hadDeliveryAvailable && !hasDeliveryAvailable`) — surface affordances to resolve the block.
- Items **truly unavailable** for other reasons (`!hadDeliveryAvailable`) — render `children` for those; the platform's default visual is correct.

For the blocked items, render:

- A prominent banner with title and body explaining the block in customer-facing terms.
- Up to three action CTAs:
  - **Switch to an allowed option** (e.g. pickup) — only when at least one allowed SLA exists for every blocked item. Emit a hint via `useMessages` directing the customer to the standard shipping picker; do not auto-apply.
  - **Satisfy the condition** — opens a sibling component (modal, drawer, redirect) that owns the resolution flow. This slot triggers it but does not contain the form.
  - **Remove all blocked** — `useCheckoutAction("REMOVE_ITEMS")` with the blocked item indexes.
- A list of the blocked items with per-item remove.

Without the paired function, the snapshot flags are absent — pass through to `children` and stop.

## States to cover

- **Simple restyle** (Shape 1) — render your custom wrapper around `children`.
- **Removal in flight** (Shape 2 or 3) — disable the remove buttons; `isPending` from `useCheckoutAction` drives the state.
- **Removal succeeded** — push a success message via `useMessages`.
- **Removal failed** — push an error message; the UI stays as it was.
- **Condition-gated, pass-through** (Shape 3) — no condition-blocked items → render `children` only.
- **Condition-gated, blocked present, allowed alternative exists** (Shape 3) — full gated UI with the alternative CTA visible.
- **Condition-gated, blocked present, no allowed alternative** (Shape 3) — gated UI without the alternative CTA.

## Visual structure (semantic only)

Depends on the shape you chose. Common guidance for all three:

- Outer wrapper representing the package. The visual tone matches the meaning: neutral for Shape 1, negative (border + surface tone via `--color-error` / `--surface-error`) for Shape 3. No hardcoded brand colors.
- If you render a banner, it's a `<header>` with an inner heading (`<h2>` or `<h3>` depending on package nesting) and a description paragraph.
- Action rows are `<div>` of `<button type="button">`. Place reversible actions first, destructive ones last.
- Item lists are `<ul>` of `<li>`. Each row has thumbnail (with `alt=""` since the name follows), name, optional tag, action button(s).
- All interactive elements expose visible hover and focus rings.
- Layouts collapse to stacked on narrow viewports via CSS media query.

## SDK touchpoints

- `useCheckoutSession()` — `session` first, `rawSession` only when needed.
  - Use `session.cartItems` for display data (name, image, price, quantity, index). `session` is the parsed, platform-agnostic snapshot and the simpler path.
  - Drop to `rawSession` only when you need something `session` doesn't carry: snapshot flags a paired hub function wrote (Shape 3) or platform-specific extras (attachments, custom fields, etc.). The `rawSession` object is the unmodified payload from the underlying commerce platform — its shape varies per backend (VTEX uses `shippingData.logisticsInfo[]`; Shopify, VNDA, and custom backends each have their own structure). Treat `rawSession` paths as platform-coupled and adapt per store.
- `useCheckoutAction("REMOVE_ITEMS")` — only when the shape involves removal. `isPending` drives the remove buttons' disabled state.
- `useMessages()` — only when the shape involves actions; surface success and failure on each one.

## Without a design reference

- For Shape 1, prefer minimal markup — wrap `children` with your container and let the platform's render do the heavy lifting.
- For Shape 2 or 3, two semantic regions stacked vertically: optional banner on top, item list or `children` below.
- Item list rows: thumbnail (fixed aspect ratio container so layout doesn't shift while images load), name, action button(s) on the right. Spacing via `gap`, not margins.
- Tokens only: `--color-error`, `--surface-error`, `--color-fg`, `--color-fg-muted`, `--surface-subtle`, etc.

## Antipatterns

- **Don't jump to Shape 3 without confirming the store also wants the paired function.** Most customers want Shape 1 (restyle) or Shape 2 (extra actions). Ask which shape applies before pulling in the gated-pair complexity.
- **Don't infer per-item blocking conditions from raw business signals inside the component on Shape 3.** The paired function owns that decision and writes the snapshot flags. Duplicating the rule in the component means the two sides drift apart the first time the rule changes.
- **Don't auto-apply the alternative delivery option** (Shape 3). The standard shipping picker owns SLA selection; emit a hint via `useMessages` and let the customer confirm.
- **Don't own the satisfy-condition form inside this slot** (Shape 3). Trigger the form (modal, drawer, redirect) but keep it in a sibling component so it can be reused on other surfaces.
- **Don't render condition-gated affordances for items that are truly unavailable** (Shape 3). Showing the "satisfy" CTA for items the customer can never make available is misleading. Render `children` for those.
- **Don't reach into `rawSession` by default.** Read from `session` first — it is the parsed, platform-agnostic shape and is enough for display data (name, image, price, quantity, index). Drop to `rawSession` only when you genuinely need data `session` doesn't carry: function-written snapshot flags (Shape 3), platform-specific attachments, custom fields. The `rawSession` object is the underlying platform's original payload and its paths change per backend, so anything you read from it is platform-coupled.
