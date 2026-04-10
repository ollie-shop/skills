# Default Template - Native Capabilities

This document describes what the **default checkout template** provides out of the box. Use it during the Template Gap Analysis step to determine what already works natively vs. what requires a custom component.

## Table of Contents
1. [Checkout Flow](#checkout-flow)
2. [Cart Step](#cart-step)
3. [Identification Step](#identification-step)
4. [Shipping Step](#shipping-step)
5. [Address Details Step](#address-details-step)
6. [Payment Step](#payment-step)
7. [Order Confirmation Page](#order-confirmation-page)
8. [Built-in Business Rules](#built-in-business-rules)
9. [UX Behaviors](#ux-behaviors)
10. [Available Slots](#available-slots)
11. [Custom Steps & Extensibility](#custom-steps--extensibility)
12. [Gap Analysis Template](#gap-analysis-template)

---

## Checkout Flow

The default template implements a **linear, multi-step checkout** with progressive disclosure. Steps are configured via a `defaultSteps` array and routed through a `StepManager` component.

| Step | ID | URL | displayCondition | Description |
|------|----|-----|------------------|-------------|
| Contact | `contact` | _(embedded in shipping page)_ | — | Email capture; no dedicated page — rendered as a sub-section of Shipping |
| Shipping | `shipping` | `/checkout/details?step=shipping` | `smart` | Address input (ZIP/postal code), delivery/pickup method selection |
| Address | `address` | `/checkout/details?step=address` | `byCondition` | Full address form and receiver info — **only shown for delivery, skipped for pickup-only** |
| Payment | `payment` | `/checkout/details?step=payment` | `smart` | Payment method selection, card form, order completion |
| Order | — | `/checkout/order/{orderGroup}` | — | Post-purchase confirmation page (not a checkout step) |

The Cart page lives at `/checkout/cart` and is separate from the step flow.

### Step Configuration

Each step in the `defaultSteps` array has:
- **`id`**: Unique identifier, used as `?step=` param
- **`name`**: Display name
- **`page`**: Component to render (maps to built-in pages or a custom `<Slot>`)
- **`summary`**: Label for the collapsed summary row on subsequent steps
- **`displayCondition`**: `"smart"` (auto-show based on session validity), `"byCondition"` (conditional), or `"required"` (always show)
- **`displayMode`**: `"Default"` (two-column layout) or `"Full"` (single-column, no sidebar)

### Step Navigation Pattern

- Each step shows **collapsed summaries** of all previously completed steps at the top, with edit buttons to go back.
- Forward navigation: primary CTA button (e.g., "Continue to Payment", "Complete with Credit card").
- Backward navigation: text link (e.g., "Return to Cart", "Return to Shipping").
- Header always shows: store logo + "Back to Cart" link.
- **Auto-redirect**: If a user navigates to a step without completing prerequisites, `getStepIncompleteToRedirect()` redirects to the first incomplete step. Empty cart redirects to `/cart`.

---

## Cart Step

**URL:** `/checkout/cart`

### Layout

- **Two-column layout** (desktop): left column for cart items, right column for summary/actions.
- **Single-column layout** (mobile): items first, then summary, sticky bottom bar for actions.

### Header

- Store logo (links to storefront)
- User email (if identified) + "Login" button
- On mobile: logo + cart icon

### Cart Items

Each item displays:
- Product thumbnail image (links to PDP)
- Product name (links to PDP)
- Unit price (with strike-through if discounted, showing original and discounted price)
- Quantity controls: decrement (-) button, text input, increment (+) button
- Line total (unit price x qty, with strike-through if discounted)
- "Remove" button (trash icon + text)
- **Add-ons slot** (`cart_item_addons`) per item for custom extensions

**Optimistic updates**: Quantity changes use debounced batching (1s window). The UI updates immediately while syncing with the server in the background. Failed removals restore the item in the UI.

When the user has already entered shipping details, the cart additionally shows:
- **Address card**: street address + city/state with edit button and map pin icon
- **Delivery group card**: shipping method name, ETA, item count, and shipping cost — items are grouped under their delivery method

### Coupon Code

- Text input with "Coupon code" placeholder (required field)
- "Apply" button
- On success: coupon is applied and discount appears in summary
- On failure: toast notification with error message (e.g., "Coupon code is invalid or has expired. Please try again with a different code.") with alert icon and dismiss button

### Summary Panel (Desktop Sidebar / Mobile Inline)

| Line | Description |
|------|-------------|
| Subtotal (N items) | Sum of all line totals |
| Shipping | Shipping cost or "Enter shipping address" placeholder |
| Discounts | Discount total (shown in accent color, negative value) — only when applicable |
| **Total** | Final total |

### Express Checkout

Located in the summary panel (desktop) or sticky bottom bar (mobile):
- "Checkout Express" label
- **Apple Pay** button (when available in the browser)
- **PayPal** Checkout button
- **Google Pay** button
- "or" separator
- "PROCEED TO CHECKOUT" primary CTA button (purple)

On mobile, the sticky bottom bar shows:
- Collapsible "Summary (N items)" with prices
- "Express" button (compact, with Apple Pay/PayPal/GPay icons)
- "Go to checkout" CTA

### Empty Cart State

- Cart icon illustration
- "Empty cart" heading
- "You haven't added any items to your cart yet. Go back to the store and select the products you'd like to buy." message
- "Go to Home" button (purple CTA)

---

## Identification Step

**URL:** `/checkout/details?step=shipping` (first sub-step)

The Contact step has no dedicated page — it is embedded as the first section within the Shipping page. It captures the user's email before showing shipping options.

### Layout

- Express checkout buttons at the top (Apple Pay + PayPal + Google Pay) with "OR" separator
- "Enter your email" heading
- Email text input (placeholder: "example@email.com")
- "Apply" button (disabled until valid email entered)
- "Waiting for contact email" placeholder for the next section (shipping form is locked)

### Behavior

- Email validation is inline (Apply button enables only when valid format detected)
- **Email domain spell-checker**: suggests corrections for common typos (e.g., "gmial.com" → "gmail.com") via `@zootools/email-spell-checker`
- Once submitted, the contact section collapses into a summary row: "Contact" label + person icon + email + edit button
- The shipping/address input appears below
- **Returning users**: If a logged-in user is detected, a `ModalUserIdentified` appears to confirm identity or allow switching accounts

---

## Shipping Step

**URL:** `/checkout/details?step=shipping` (after email is captured)

### Address Input

- "Enter your address to see shipping options" instruction
- "You will complete your address in another step" sub-text
- Single "Address" text input with **autocomplete** (accepts ZIP/postal codes or address search)
- "Apply" button (disabled until input provided)
- "Waiting for address" placeholder below

Once resolved:
- Address is displayed as a card: map pin icon + "City, State, ZIP" + edit button

### Delivery Method Selection

- **"Shipping and Pickup" heading** with tab-based selector:
  - **Shipping** tab: icon + "Shipping" + eligibility badge
  - **Pickup** tab: icon + "Pickup" + eligibility badge (disabled when no pickup locations available)
  - **Custom delivery** tab: supported via `ModalCustomDelivery` for non-standard options

- **Method list** (radio-style cards):
  - Each method shows: ETA (e.g., "in 6 days"), method name (e.g., "VTEX Delivery"), price
  - Scheduled delivery option shows: "Choose date and time", method name (e.g., "Scheduled Delivery"), price

- **Package preview**: item thumbnails with quantity badges + "See details" link

### Pickup Flow

When "Pickup" tab is selected and eligible:
- `PackagePickup` component renders pickup location options
- Each location shows availability badges (`PickupAvailabilityBadge`)
- Address availability checking (`use-address-availability` hook)

### Unavailable Items

If items in the cart are not available for the selected shipping method:
- `PackageUnavailable` component renders a list of unavailable items
- `UnavailableItemsConfirmModal` asks the user to confirm or remove items
- Step validation fails if unavailable items exist (triggers redirect)

### Scheduled Delivery Modal

When "Scheduled Delivery" is selected, a modal opens:
- **"Scheduling options"** heading with calendar icon and close (X) button
- **"Date and time"** sub-heading
- **Date picker**: horizontal scrollable row of day buttons (format: "Mon 04/13"), with prev/next arrow navigation
- **Time slots**: list of available windows (e.g., "9:00 AM - 11:00 AM", "11:00 AM - 2:00 PM", etc.)
- **"Confirm"** button (disabled until both date and time are selected)

### Navigation

- "Return to Cart" link
- "Continue to Payment" button (purple CTA)
- PayPal button (inline)

---

## Address Details Step

**URL:** `/checkout/details?step=address`

This step appears between shipping method selection and payment. It collects the full delivery address and receiver details. **It is conditional** — only shown when delivery is selected (`displayCondition: "byCondition"`), skipped for pickup-only orders.

### Collapsed Summaries

- **Contact**: email + edit button
- **Shipping**: method name + ETA + item count link + map pin + city/state + edit button

### Address Form

- **"Address details"** heading with "*Required Fields" indicator
- Resolved location shown: map pin icon + "City, State, Country" + "Update" button
- **Address fields**:
  - Address* (required)
  - Apartment, suite, etc (optional)
- **Receiver section** ("Receiver" heading):
  - First name* (required)
  - Last name* (required)
  - Phone number* (required)

**Validation**: Uses Zod schema validation with `react-hook-form`. Fields have max length counters and special character validation for international addresses (Unicode-aware regex). Document/CPF validation available for Brazilian stores.

**Returning users**: Can select from previously saved addresses via `ModalEditAddressesDelivery`, or add a new one.

### Navigation

- "Return to Shipping" link
- "Continue to Payment" button (purple CTA)
- PayPal button (inline)

---

## Payment Step

**URL:** `/checkout/details?step=payment`

### Collapsed Summaries

- **Contact**: name + email + phone + edit button
- **Shipping**: method + ETA + item count link + full address + edit button

### Payment Section

- **"Payment"** heading with lock icon + "Encrypted" badge
- "Select a payment method to continue." instruction
- **"Add card or coupon"** link (for gift cards)

### Gift Card

- Native gift card input and management (`GiftCard` component)
- If gift card covers the full order amount: `GiftCardFullPaidModal` appears, other payment methods are disabled, and `GiftCardPaidBanner` shows the applied amount

### Saved Cards

- `SavedCardOption` component for returning users with stored payment methods
- Shows masked card number, brand icon, and selection radio

### Payment Methods (Accordion Radio List)

Each method is a radio button with an expand/collapse chevron. Selecting a method expands its content and collapses others. Method selection uses debounced updates (1.5s).

The available payment methods are **inherited from the platform configuration** — the template renders whatever methods the store has configured. Each method displays its icon/logo and, when expanded, shows method-specific fields or instructions.

Natively supported method types: `credit_card`, `pix`, `promissory`, `paypal`, `paypalCp`, `google_pay`, `apple_pay`, `boleto`, `picpay`, `nupay`, `pagalevePix`, `pagaleve_parcelado`, `affirm`, `click_to_pay`.

### Credit Card Form

All sensitive fields are rendered inside **PCI-compliant iframes** hosted on `pci.ollie.shop`:

| Field | Type | Details |
|-------|------|---------|
| Card number | Text input (iframe) | With card brand detection icon |
| Expiration date | Text input (iframe) | MM/YY format |
| Security code | Text input (iframe) | With help tooltip: "The security code is a three or four digit number located on the back of your card." |
| Name on card | Text input (iframe) | Cardholder name (max 50 chars, no numbers) |

Additional options:
- **"Save card in my account"** checkbox (checked by default)
- **"Billing address same as shipping address"** checkbox (checked by default) — shows address preview below
- **Installments**: Fetched dynamically via `use-fetch-installments` hook based on selected card and order total

### Payment Verification

After external payment redirects (e.g., PayPal popup, Pix), a `VerifyTransaction` component handles the callback:
- Detects `?og=orderGroup` URL param
- Calls `gatewayCallbackAction` to verify payment status
- On success: redirects to `/order/{orderGroup}`
- On failure: removes param and shows error toast

### Order Completion CTA

- Dynamic label based on selected method: "Complete with [Method]" (e.g., "Complete with Credit card", "Complete with Promissory")
- Disabled until payment method is fully filled
- **"Return to Shipping"** link for backward navigation

---

## Order Confirmation Page

**URL:** `/checkout/order/{orderGroup}`

This is a standalone page rendered after successful payment — not a checkout step.

### Content

- **Header**: Order ID, customer info, package tracking numbers
- **Contact summary**: Customer email and phone
- **Shipping summary**: Delivery addresses with tracking packages (copy tracking number to clipboard)
- **Payment summary**: Payment methods used
- **Totalizer**: Order breakdown (items, shipping, discounts, tax, total)
- **Print button**: Print-friendly order summary
- **Account link**: Link to account orders page
- **Multi-fulfillment support**: Handles orders split across multiple fulfillments

### Platform-Specific Content

- **Bank Invoice** (`PlatformBankInvoice`): Renders boleto/bank invoice for VTEX
- **Payment Conditions** (`PlatformConditions`): Post-purchase payment details (Pix QR code modal, PagaLeve details)

---

## Built-in Business Rules

These rules are **already handled** by the native template. Custom components should not re-implement them unless overriding is explicitly required.

| Rule | Step | Behavior |
|------|------|----------|
| Empty cart handling | Cart | Shows empty state with illustration and "Go to Home" CTA |
| Optimistic cart updates | Cart | Quantity changes debounced (1s), batched, and synced with server; UI updates immediately |
| Quantity controls | Cart | Increment/decrement buttons with inline text input; line totals update |
| Item removal | Cart | "Remove" button per item; item hidden immediately, restored on failure |
| Coupon validation | Cart | Input + Apply; toast error for invalid coupons; discount line in summary |
| Price display with discounts | Cart | Original price struck through, discounted price shown; line totals reflect discounts |
| Delivery grouping | Cart/Shipping | Items grouped by delivery method with method name, ETA, item count, and cost |
| Subtotal calculation | All | Live-updating subtotal across steps (items x qty) |
| Shipping cost calculation | Shipping+ | Fetches shipping options from platform API based on address; updates summary |
| Discount calculation | All | Calculates and displays discount total; shown in accent color as negative value |
| Total calculation | All | Subtotal + Shipping - Discounts |
| Email validation | Identification | Inline validation; Apply button disabled until valid email format |
| Email spell-check | Identification | Suggests corrections for common domain typos |
| Address resolution | Shipping | Resolves ZIP/postal code to city/state via autocomplete; displays resolved address |
| Address form validation | Address | Zod schema validation; required fields, max length, special character rules |
| Document/CPF validation | Address | Brazilian document validation and formatting (when applicable) |
| Shipping method selection | Shipping | Radio-style cards with first option pre-selected |
| Scheduled delivery | Shipping | Date + time slot picker modal with confirm action |
| Pickup option | Shipping | Tab-based toggle; shows pickup locations with availability badges |
| Unavailable items detection | Shipping | Detects items not available for shipping; shows confirmation modal; blocks checkout |
| Guest/login enforcement | Cart/Details | Feature flags `allowGuest.cart` and `allowGuest.details`; shows login modal or redirects |
| Payment method selection | Payment | Accordion radio list; only one expanded at a time; debounced updates (1.5s) |
| Gift card payment | Payment | Native gift card input; full-amount detection disables other methods |
| Saved card management | Payment | Returning users can select previously saved cards |
| PCI-compliant card input | Payment | Sensitive card fields rendered in iframes from PCI-certified domain |
| Card brand detection | Payment | Auto-detects card brand from number and shows logo |
| Installment calculation | Payment | Dynamically fetches installment options based on card and total |
| Save card option | Payment | Checkbox to save card for future purchases (default: checked) |
| Billing address reuse | Payment | Checkbox to reuse shipping address as billing (default: checked) |
| Dynamic CTA label | Payment | "Complete with [Method]" updates based on selected payment method |
| Payment verification | Payment | Handles external payment callbacks (PayPal, Pix); verifies and redirects |
| Express checkout | Cart/Identification | Apple Pay, PayPal, and Google Pay available as express options |
| Step summary collapse | All | Completed steps collapse into summary rows with edit buttons |
| Step auto-redirect | All | Navigating to an incomplete step redirects to first missing prerequisite |
| reCAPTCHA | Payment | Google reCAPTCHA v3 protection |
| i18n / Internationalization | All | Full internationalization via `next-intl`; locale-aware formatting |

---

## UX Behaviors

### Navigation

- Linear multi-step flow: Cart → Contact → Shipping → Address (conditional) → Payment → Order
- Collapsible summary of completed steps at the top of each subsequent step
- Each completed step summary has an edit button to navigate back
- Forward: primary CTA button
- Backward: text link (e.g., "Return to Cart", "Return to Shipping")
- "Back to Cart" link always available in header
- Mobile: breadcrumb step indicator (`StepBreadcrumb`)

### Responsiveness

**Desktop (≥1024px):**
- Two-column layout: main content (left) + sticky sidebar summary (right)
- Sidebar contains: login, coupon input, summary breakdown, express checkout buttons, "PROCEED TO CHECKOUT" CTA
- Completed step summaries show full details (name, email, phone, address, method, ETA)
- CSS custom properties for layout customization: `--details-container-lg-width`, `--details-sidebar-lg-width`, `--details-container-lg-max-width`

**Mobile (<1024px):**
- Single-column layout
- Sidebar summary moves below main content
- **Sticky bottom bar** with:
  - Collapsible "Summary (N items)" toggle with subtotal and total
  - Express checkout compact button (icons only: Apple Pay, PayPal, GPay)
  - Primary CTA button (e.g., "Go to checkout", "Complete with Credit card")
- Completed step summaries become compact single-line with dropdown chevron (e.g., "Contact · test@email.com · John Doe")
- Header shows logo + cart icon (no text links)
- "Return to [Step]" link with arrow moves to top-left

### Loading States

- Loading spinner on coupon validation
- Skeleton loaders during shipping package loading (`SkeletonPackages`)
- "Apply" buttons disabled during async operations
- "Waiting for contact email" / "Waiting for address" placeholder text for locked sections
- `PaymentLoadingModal` during payment processing
- Disabled CTA buttons until required data is provided

### Error Handling

- Toast notifications (sticky, auto-dismiss after 5s) with types: `error`, `warning`, `info`, `success`
  - Alert icon
  - Descriptive error message
  - Dismiss button
- `UnauthorizedPaymentModal` for payment failures
- Required field indicators (*) on form fields
- Inline field-level validation errors via `react-hook-form`
- Failed API operations restore previous UI state (e.g., cart item removal rollback)

### Accessibility

- Semantic HTML: headings hierarchy (h1-h5), navigation landmarks
- ARIA: alert roles with `live="assertive"` for dynamic notifications
- Image alt text on all product images, icons, and logos
- Keyboard-focusable interactive elements
- Descriptive button labels (e.g., "Edit" with aria-description)
- Labeled form inputs with visible placeholders
- reCAPTCHA integration

### Security

- PCI-compliant card fields via isolated iframes (hosted on `pci.ollie.shop`)
- "Encrypted" badge visible on payment step (lock icon)
- Google reCAPTCHA v3 for bot protection
- "Save card in my account" opt-in (not silent)
- Guest/login enforcement via feature flags

### Observability

- **Grafana Faro** integration for real-time observability
- Tracked actions: `begin_checkout`, `changeQuantity`, `removeFromCart`, `navigateToOrder`, `selectPayment`, `placeOrder`, `updateAddress`, `updateReceiver`
- Per-action timing, success/failure tracking
- GA4 analytics events (e.g., `begin_checkout`, order conversion)
- `PageViewTracker` for page-level analytics

---

## Available Slots

The template uses a `<Slot id="...">` component (from `@ollie-shop/react`) as the extension point mechanism. Each slot wraps a default implementation — injecting a custom component into a slot **replaces** the default content.

### Cart Page

| Slot ID | Position | Default Content |
|---------|----------|-----------------|
| `cart_header_full_page` | Full-width above cart | _(empty)_ |
| `header` | Cart header area | Store logo + login |
| `cart_title` | Cart heading | "Cart N items" |
| `cart_items` | Item list area | Native item list with quantity controls |
| `cart_item_addons` | Below each cart item | _(empty)_ — receives `item` prop |
| `cart_empty` | Empty cart state | Empty cart illustration + "Go to Home" |
| `cart_coupon_form` | Coupon section (desktop inline + sidebar) | Coupon input + Apply button |
| `cart_totalizer` | Totals section in main content | Subtotal, shipping, discounts, total |
| `cart_sidebar` | Right sidebar (desktop) | Full sidebar with login, coupon, totals, express |
| `cart_sidebar_login` | Sidebar login section | Login button |
| `cart_sidebar_totalizer` | Sidebar totals | Subtotal, shipping, discounts, total |
| `cart_checkout_express` | Express checkout section | Apple Pay, PayPal, Google Pay buttons |
| `cart_express_wrapper` | Wrapper around express buttons | Express button container |
| `cart_mobile_navigation` | Mobile sticky bottom bar | Summary toggle + Express + CTA |
| `cart_mobile_navigation_total_label` | Total label in mobile nav | Price display |

### Checkout Layout (Details Pages)

| Slot ID | Position | Default Content |
|---------|----------|-----------------|
| `checkout_header_full_page` | Full-width above checkout | _(empty)_ |
| `header` | Checkout header | Store logo + "Back to Cart" |
| `checkout_summaries` | Previous step summaries area | Contact + Shipping summaries |
| `checkout_sidebar` | Right sidebar | Coupon, totals, checkout button |
| `checkout_sidebar_login` | Sidebar login section | Login button |
| `checkout_coupon_form` | Sidebar coupon | Coupon input + Apply |
| `checkout_sidebar_totalizer` | Sidebar totals | Summary breakdown |
| `checkout_sidebar_checkout_button` | Sidebar checkout CTA | _(empty)_ |
| `checkout_totalizer` | Totals inside payment step | Summary breakdown |
| `checkout_express_options` | Express checkout in modal | Express method selection |

### Step Summaries

| Slot ID | Position | Default Content |
|---------|----------|-----------------|
| `summaries` | Wrapper around all summaries | All step summaries |
| `contact_summary` | Contact summary row | Email + name + phone |
| `shipping_summary` | Shipping summary row | Method + ETA + address |
| `{step.summary}_summary` | Dynamic per custom step | _(empty)_ — receives step props |

### Shipping Step

| Slot ID | Position | Default Content |
|---------|----------|-----------------|
| `shipping_title` | Shipping section heading | "Delivery Method" / method type label |
| `shipping_list_packages` | Package list area | Package list with shipping/pickup options — receives `type` prop (`"delivery"` or `"pick_up"`) |
| `shipping_package_unavailable` | Unavailable items section | Unavailable item list — receives `items` prop |
| `shipping_address_details_form` | Address form in Address step | Address + receiver form fields |

### Payment Step

| Slot ID | Position | Default Content |
|---------|----------|-----------------|
| `payment_option_gift_card` | Gift card section | Gift card input and management |
| `saved_card_options` | Saved cards section | List of saved payment cards |
| `payment_method_options` | Payment method list | All available payment methods |
| `payment_option_credit_card` | Credit card method content | PCI card form |
| `payment_option_boleto` | Boleto method content | Boleto instructions |
| `payment_option_pix` | Pix method content | Pix payment UI |
| `payment_option_promissory` | Promissory method content | Promissory note instructions |
| `payment_option_paypal` | PayPal method content | PayPal redirect button |
| `payment_option_apple_pay` | Apple Pay method content | Apple Pay button |
| `payment_option_google_pay` | Google Pay method content | Google Pay button |
| `payment_option_click_to_pay` | Click to Pay content | Click to Pay UI |
| `payment_option_affirm` | Affirm method content | Affirm installment UI |
| `payment_option_nubank` | NuPay method content | NuPay UI |
| `payment_option_pagaleve_parcelado` | PagaLeve installment content | PagaLeve parcelado UI |
| `payment_option_pagaleve_avista_transparente` | PagaLeve à vista content | PagaLeve transparent UI |
| `payment_option_pagaleve_parcelado_transparente` | PagaLeve parcelado transparent | PagaLeve transparent installment UI |
| `payment_option_pagaleve_mensal_transparente` | PagaLeve monthly transparent | PagaLeve monthly UI |
| `payment_{method_type}_content` | Dynamic per method type | Method-specific content area |

### Navigation

| Slot ID | Position | Default Content |
|---------|----------|-----------------|
| `step_navigation` | Bottom navigation bar | Prev/Next buttons + summary toggle |
| `step_navigation_total_label` | Price label in nav bar | Total amount display |
| `step_navigation_place_order_button` | Place order button | "Complete with [Method]" CTA |

### Order Confirmation Page

| Slot ID | Position | Default Content |
|---------|----------|-----------------|
| `order_header` | Order page header | Order ID + customer info |
| `order_details` | Order details section | Items, shipping, payment summary |
| `order_totalizer` | Order totals | Full order breakdown |
| `order_footer` | Order page footer | Account link + legal |

### Footer

| Slot ID | Position | Default Content |
|---------|----------|-----------------|
| `footer_all_container` | Entire footer wrapper | Full footer |
| `footer_sidebar` | Footer in sidebar context | Sidebar footer |
| `footer` | Footer content | "Checkout powered by Ollie" |

### Modals

| Slot ID | Position | Default Content |
|---------|----------|-----------------|
| `modal_user_identified` | User identification modal | Confirm identity or switch accounts |
| `modal_profile_summary` | Profile summary in modals | Email edit / profile display |

---

## Custom Steps & Extensibility

### Adding Custom Steps

Custom steps can be added to the checkout flow via the store's `props.steps` configuration array. The `StepManager` renders custom steps using the `<Slot>` mechanism:

```typescript
// Example: Adding a custom "Review" step before payment
{
  id: "review",
  name: "Review",
  page: "my_review_step",           // → renders <Slot id="my_review_step">
  summary: "Review",                // → summary uses <Slot id="review_summary">
  displayCondition: "smart",
  displayMode: "Default"            // "Default" = two-column, "Full" = single-column
}
```

Custom steps receive `prev` and `next` navigation props via the Slot.

### Custom Step Validation

- Custom step completion is tracked via `session.extensions.checkoutSteps[stepId].isCompleted`
- Steps with `displayCondition: "smart"` participate in auto-redirect logic
- Steps with `displayCondition: "byCondition"` are skipped by the navigation flow unless conditions are met
- Steps with `displayCondition: "required"` are always shown

### Feature Flags

The template supports feature flags via `props.flags`:

| Flag | Default | Effect |
|------|---------|--------|
| `allowGuest.cart` | `true` | Allow guest users on cart page |
| `allowGuest.details` | `true` | Allow guest users in checkout flow |
| `useOllieLogin` | `true` | Use Ollie OAuth login vs. platform redirect |
| `showLoginOnSidebar` | `true` | Show login button in sidebar |
| `enableShippingSimulation` | `true` | Enable shipping simulation on cart |
| `enablePendingPix` | `true` | Enable pending Pix payment tracking |

> **Note:** Flags default to "enabled" when `null` or `undefined`.

---

## Gap Analysis Template

Use this table when comparing client requirements against native capabilities:

| Requirement | Native Support | Gap | Action |
|-------------|---------------|-----|--------|
| _e.g., "Show loyalty points in cart"_ | No | Full gap | Build custom component for `cart_sidebar_totalizer` or `cart_totalizer` slot |
| _e.g., "Coupon input"_ | Yes (built-in) | None | Use native — no custom component needed |
| _e.g., "Custom shipping labels"_ | Partial (shows method name + ETA but no custom labels) | Partial gap | Extend via `shipping_title` or `shipping_list_packages` slot |
| _e.g., "Gift card payment"_ | Yes (built-in) | None | Native gift card flow; extend via `payment_option_gift_card` slot if customization needed |
| _e.g., "Custom address fields (e.g., tax ID)"_ | No | Full gap | Build custom component for `shipping_address_details_form` slot |
| _e.g., "Installment selection"_ | Yes (credit card) | None | Native installment fetch; extend via `payment_option_credit_card` slot if custom UI needed |
| _e.g., "Order review step before payment"_ | No | Full gap | Add custom step with `displayCondition: "smart"` before payment in steps config |
| _e.g., "Custom per-item add-ons"_ | No | Full gap | Build custom component for `cart_item_addons` slot |
| _e.g., "Custom mobile navigation"_ | Partial | Partial gap | Extend via `cart_mobile_navigation` or `step_navigation` slot |
| _e.g., "Custom order confirmation"_ | Partial | Partial gap | Extend via `order_header`, `order_details`, `order_totalizer`, or `order_footer` slots |
