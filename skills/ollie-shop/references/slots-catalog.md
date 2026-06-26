# Slots Catalog — Default Template

Slots are the extension points the checkout template exposes. A **slot** has a stable id (a string) and lives at a specific position in the rendered tree. Injecting a custom component into a slot **replaces** the template's default content for that position. Multiple stores can target the same slot id and each gets its own component.

**One custom component per slot.** A store's components are resolved by finding the single entry whose `slot` matches the id — there is no stacking, ordering, or fallback chain. If two requirements target the same slot, consolidate them into one component (render one inside the other), or move one to a narrower/adjacent slot. Your component always *receives* the slot's default UI as `children`: render `{children}` to keep it (augment) or omit it to take over (replace). See `sdk-guide.md` §Component Interface → "Children: replace vs augment".

This catalog covers the **default template** only. Slots belong to a template — other templates would publish their own catalog with potentially different ids. The machine-readable source is [`assets/checkout-slots-data.yaml`](../assets/checkout-slots-data.yaml).

## Dynamic slots

Some slot ids are **patterns**, not literal strings. The host emits one slot per resolved variable value, so a single dynamic entry covers an unbounded set of concrete ids — including future ones the skill never gets updated about.

Treat any id with `{{ ... }}` segments as a pattern: resolve the variable at the call site, then point the customization at the resolved id. The known instances listed below are illustrative, not exhaustive.

| Pattern | Variable resolves to | Use case |
|---|---|---|
| `payment_option_{{ paymentMethodName }}` | The method id configured for the store (`pix`, `credit_card`, `debit_card`, `boleto`, `paypal`, `apple_pay`, `google_pay`, `affirm`, `nubank`, etc.) | Customize the tile for one specific payment method — change copy, add disclaimers, replace the form, or hide the method behind a condition. New methods added to the store render through the same pattern with no skill update. |
| `payment_{{ paymentType }}_content` | The method's normalized type | Extend the inside of a payment option (e.g. extra copy below the credit card fields) without overriding the whole option. |
| `{{ stepId }}_summary` | A checkout step id (`contact`, `shipping`, custom step ids) | Replace the collapsed summary row shown above the active step. |

When a user says "customize the PIX section", resolve to `payment_option_pix` — that's the dynamic pattern with `paymentMethodName = pix`.

## General

| Slot ID | Notes |
|---|---|
| `header` | Store logo + login row, shared between cart and checkout pages. |
| `cart_header_full_page` | Empty by default. Full-width band above the cart page. |
| `checkout_header_full_page` | Empty by default. Full-width band above the checkout steps. |
| `footer_all_container` | Wraps the entire footer area. |
| `footer` | Footer content (full width). |
| `footer_sidebar` | Footer rendered in sidebar context. |

## Cart Page

| Slot ID | Notes |
|---|---|
| `cart_empty` | Empty-cart state. Default: illustration + "Back to Home". |
| `cart_title` | Cart heading (e.g. "Cart · N items"). |
| `cart_items` | Item list area. Default renders the native list with quantity controls. |
| `cart_item` | One row per cart item. Receives the item as a prop. |
| `cart_item_addons` | Empty by default. Renders once per item, with `item` prop. Good place for upsell or compliance content. |
| `card_item_base` | Base wrapper around an item card. |
| `cart_items_package_list_item_header` | Header inside a shipping-package group on the cart. |
| `cart_address_card` | Inline address card on the cart page. |
| `cart_address_card_wrapper` | Wrapper around the address card. |
| `cart_sidebar` | Full right sidebar on desktop. |
| `cart_sidebar_login` | Login row inside the cart sidebar. |
| `cart_sidebar_totalizer` | Totals breakdown inside the cart sidebar. |
| `cart_sidebar_checkout_button` | Primary CTA at the bottom of the cart sidebar. |
| `cart_coupon_form` | Coupon input + apply button. |
| `cart_express_options` | Express checkout buttons (Apple Pay, Google Pay, PayPal). |
| `cart_totalizer` | Totals breakdown rendered in the main content (not sidebar). |

## Checkout Steps

| Slot ID | Notes |
|---|---|
| `checkout_sidebar` | Full right sidebar shared across steps. |
| `checkout_sidebar_items_summary` | Items summary inside the checkout sidebar. |
| `checkout_sidebar_login` | Login row inside the checkout sidebar. |
| `checkout_sidebar_totalizer` | Totals breakdown inside the checkout sidebar. |
| `checkout_sidebar_checkout_button` | Primary CTA inside the checkout sidebar. |
| `checkout_coupon_form` | Coupon input + apply button (checkout sidebar). |
| `checkout_totalizer` | Totals rendered inside the payment step. |
| `checkout_summaries` | Wrapper around previous-step summaries shown above the active step. |

## Summaries

| Slot ID | Notes |
|---|---|
| `contact_summary` | Collapsed contact summary row. |
| `shipping_summary` | Collapsed shipping summary row. |
| `{{ stepId }}_summary` | **Dynamic**. One slot per step that defines a summary. See "Dynamic slots" above. |

## Shipping

| Slot ID | Notes |
|---|---|
| `shipping_title` | Shipping section heading. |
| `shipping_delivery_method_selector` | Toggle between delivery / pickup. |
| `shipping_address_card` | Address card shown during the shipping step. |
| `shipping_list_packages` | Package list area. Receives `type` (`delivery` | `pick_up`). |
| `shipping_package_shipping_options` | Per-package shipping option picker. |
| `shipping_package_unavailable` | Section for items that can't be shipped to the address. Receives `items`. |
| `shipping_package_unavailable_item` | One row per unavailable item. |
| `items_gallery_modal_content` | Modal that lists the items of a shipping package. |
| `items_gallery_item_content` | One row inside the items gallery modal. |

## Address

| Slot ID | Notes |
|---|---|
| `address_page` | Whole address page body. |
| `shipping_address_details_form` | Address + receiver form fields. |

## Payment

| Slot ID | Notes |
|---|---|
| `payment_method_options` | Renders the list of available payment methods (the "tiles"). |
| `step_navigation_place_order_button` | "Complete with X" CTA at the bottom of the payment step. |
| `payment_option_{{ paymentMethodName }}` | **Dynamic**. One slot per payment method configured for the store. See "Dynamic slots" above. |
| `payment_{{ paymentType }}_content` | **Dynamic**. Inner content area inside a payment option. See "Dynamic slots" above. |

> **Visibility gotcha — `payment_option_*`.** Every `payment_option_*` slot (except `payment_option_gift_card`) only mounts when the user actively clicks that payment method's tab — it does **not** exist in the DOM until then. Never place a primary CTA, confirm button, or anything that must be visible as soon as the payment step renders at a `payment_option_*` slot. Use `payment_method_options` for content that must be present immediately.

## Order Page

| Slot ID | Notes |
|---|---|
| `order_header` | Order ID + customer info header. |
| `order_details` | Items + shipping + payment summary. |
| `order_totalizer` | Full order breakdown. |
| `order_footer` | Account link + legal text. |

## Modals

| Slot ID | Notes |
|---|---|
| `modal_profile_summary` | Profile summary content shown inside modals. |
| `modal_user_identified` | "Confirm your identity / switch accounts" modal body. |
| `modal_change_delivery_address` | Delivery address edit modal body. |
| `modal_change_pickup_address` | Pickup address edit modal body. |

## Navigation

| Slot ID | Notes |
|---|---|
| `cart_mobile_navigation` | Mobile sticky bottom bar on the cart page. |
| `cart_mobile_navigation_total_label` | Price label inside the mobile cart nav. |
| `cart_express_wrapper` | Wrapper around express checkout buttons. |
| `cart_proceed_to_checkout_button` | "Proceed to checkout" CTA. |
| `step_navigation` | Bottom navigation bar across checkout steps. |
| `step_navigation_total_label` | Price label inside the step navigation bar. |

## How to use this catalog

1. **Find the slot for what the user described.** Search by group first, then by id. If the request is about a specific payment method, jump straight to the dynamic pattern.
2. **Confirm the slot still makes sense for the target template.** Slots are template-bound — if the store uses a custom template, ask which one.
3. **Pass the resolved id to `ollieshop component create --slot <slot-id>`.**
4. **For dynamic slots, write the resolved id, not the pattern.** E.g. `--slot payment_option_pix`, never `--slot payment_option_{{ paymentMethodName }}`.

When no existing slot fits the request, that's a sign the customization needs a template extension — flag it back to the user instead of forcing the fit.
