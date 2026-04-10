# Default Template - Native Capabilities

This document describes what the **default checkout template** provides out of the box. Use it during the Template Gap Analysis step to determine what already works natively vs. what requires a custom component.

## Table of Contents
1. [Checkout Flow](#checkout-flow)
2. [Cart Step](#cart-step)
3. [Identification Step](#identification-step)
4. [Shipping Step](#shipping-step)
5. [Address Details Step](#address-details-step)
6. [Payment Step](#payment-step)
7. [Built-in Business Rules](#built-in-business-rules)
8. [UX Behaviors](#ux-behaviors)
9. [Available Slots](#available-slots)
10. [Gap Analysis Template](#gap-analysis-template)

---

## Checkout Flow

The default template implements a **linear, multi-step checkout** with progressive disclosure. Each step is a sub-route under `/checkout/details`.

| Step | URL | Description |
|------|-----|-------------|
| Cart | `/checkout/cart` | Cart review, quantity editing, coupon, express checkout |
| Identification | `/checkout/details?step=shipping` | Email capture (first sub-step on the shipping page) |
| Shipping | `/checkout/details?step=shipping` | Address input (ZIP/postal code), delivery method selection |
| Address Details | `/checkout/details?step=address` | Full address form and receiver information |
| Payment | `/checkout/details?step=payment` | Payment method selection, card form, order completion |

> **Note:** There is no standalone confirmation/review step. The payment step includes a "Complete order" CTA that places the order directly. Previous steps are summarized in collapsible sections at the top of each subsequent step.

### Step Navigation Pattern

- Each step shows **collapsed summaries** of all previously completed steps at the top, with edit buttons to go back.
- Forward navigation: primary CTA button (e.g., "Continue to Payment", "Complete with Credit card").
- Backward navigation: text link (e.g., "Return to Cart", "Return to Shipping").
- Header always shows: store logo + "Back to Cart" link.

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
- PayPal Checkout button
- Google Pay button
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

This step captures the user's email before showing shipping options. It is the first thing displayed when proceeding from cart.

### Layout

- Express checkout buttons at the top (PayPal + Google Pay) with "OR" separator
- "Enter your email" heading
- Email text input (placeholder: "example@email.com")
- "Apply" button (disabled until valid email entered)
- "Waiting for contact email" placeholder for the next section (shipping form is locked)

### Behavior

- Email validation is inline (Apply button enables only when valid format detected)
- Once submitted, the contact section collapses into a summary row: "Contact" label + person icon + email + edit button
- The shipping/address input appears below

---

## Shipping Step

**URL:** `/checkout/details?step=shipping` (after email is captured)

### Address Input

- "Enter your address to see shipping options" instruction
- "You will complete your address in another step" sub-text
- Single "Address" text input (accepts ZIP/postal codes for initial address resolution)
- "Apply" button (disabled until input provided)
- "Waiting for address" placeholder below

Once resolved:
- Address is displayed as a card: map pin icon + "City, State, ZIP" + edit button

### Delivery Method Selection

- **"Shipping and Pickup" heading** with two tabs:
  - **Shipping** tab: icon + "Shipping" + "All eligible" badge (active)
  - **Pickup** tab: icon + "Pickup" + "None eligible" badge (disabled when no pickup options available)

- **Method list** (radio-style cards):
  - Each method shows: ETA (e.g., "in 6 days"), method name (e.g., "VTEX Delivery"), price
  - Scheduled delivery option shows: "Choose date and time", method name (e.g., "Scheduled Delivery"), price

- **Package preview**: item thumbnails with quantity badges + "See details" link

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

This step appears between shipping method selection and payment. It collects the full delivery address and receiver details.

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
- **"Add card or coupon"** link (for gift cards/vouchers)

### Payment Methods (Accordion Radio List)

Each method is a radio button with an expand/collapse chevron. Selecting a method expands its content and collapses others.

The available payment methods are **inherited from the platform configuration** — the template renders whatever methods the store has configured (e.g., credit card, boleto, Pix, digital wallets, etc.). Each method displays its icon/logo and, when expanded, shows method-specific fields or instructions.

### Credit Card Form

All sensitive fields are rendered inside **PCI-compliant iframes** hosted on `pci.ollie.shop`:

| Field | Type | Details |
|-------|------|---------|
| Card number | Text input (iframe) | With card brand detection icon |
| Expiration date | Text input (iframe) | MM/YY format |
| Security code | Text input (iframe) | With help tooltip: "The security code is a three or four digit number located on the back of your card." |
| Name on card | Text input (iframe) | Cardholder name |

Additional options:
- **"Save card in my account"** checkbox (checked by default)
- **"Billing address same as shipping address"** checkbox (checked by default) — shows address preview below

### Order Completion CTA

- Dynamic label based on selected method: "Complete with [Method]" (e.g., "Complete with Credit card", "Complete with Promissory")
- Disabled until payment method is fully filled
- **"Return to Shipping"** link for backward navigation

---

## Built-in Business Rules

These rules are **already handled** by the native template. Custom components should not re-implement them unless overriding is explicitly required.

| Rule | Step | Behavior |
|------|------|----------|
| Empty cart handling | Cart | Shows empty state with illustration and "Go to Home" CTA |
| Quantity controls | Cart | Increment/decrement buttons with inline text input; line totals update |
| Item removal | Cart | "Remove" button per item with trash icon |
| Coupon validation | Cart | Input + Apply; toast error for invalid coupons; discount line in summary |
| Price display with discounts | Cart | Original price struck through, discounted price shown; line totals reflect discounts |
| Delivery grouping | Cart/Shipping | Items grouped by delivery method with method name, ETA, item count, and cost |
| Subtotal calculation | All | Live-updating subtotal across steps (items x qty) |
| Shipping cost calculation | Shipping+ | Fetches shipping options from platform API based on address; updates summary |
| Discount calculation | All | Calculates and displays discount total; shown in accent color as negative value |
| Total calculation | All | Subtotal + Shipping - Discounts |
| Email validation | Identification | Inline validation; Apply button disabled until valid email format |
| Address resolution | Shipping | Resolves ZIP/postal code to city/state; displays resolved address with edit option |
| Address form validation | Address | Required field validation for address, first name, last name, phone |
| Shipping method selection | Shipping | Radio-style cards with first option pre-selected |
| Scheduled delivery | Shipping | Date + time slot picker modal with confirm action |
| Pickup option | Shipping | Tab-based toggle; disabled when no pickup locations available |
| Payment method selection | Payment | Accordion radio list; only one expanded at a time |
| PCI-compliant card input | Payment | Sensitive card fields rendered in iframes from PCI-certified domain |
| Card brand detection | Payment | Auto-detects card brand from number and shows logo |
| Save card option | Payment | Checkbox to save card for future purchases (default: checked) |
| Billing address reuse | Payment | Checkbox to reuse shipping address as billing (default: checked) |
| Dynamic CTA label | Payment | "Complete with [Method]" updates based on selected payment method |
| Express checkout | Cart/Identification | PayPal and Google Pay available as express options |
| Step summary collapse | All | Completed steps collapse into summary rows with edit buttons |
| reCAPTCHA | Payment | Google reCAPTCHA v3 protection (badge visible on payment step) |

---

## UX Behaviors

### Navigation

- Linear multi-step flow: Cart → Identification → Shipping → Address → Payment
- Collapsible summary of completed steps at the top of each subsequent step
- Each completed step summary has an edit button to navigate back
- Forward: primary CTA button
- Backward: text link (e.g., "Return to Cart", "Return to Shipping")
- "Back to Cart" link always available in header

### Responsiveness

**Desktop (>768px):**
- Two-column layout: main content (left) + sidebar summary (right)
- Sidebar contains: coupon input, summary breakdown, express checkout buttons, "PROCEED TO CHECKOUT" CTA
- Completed step summaries show full details (name, email, phone, address, method, ETA)

**Mobile (<768px):**
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
- "Apply" buttons disabled during async operations
- "Waiting for contact email" / "Waiting for address" placeholder text for locked sections
- Disabled CTA buttons until required data is provided

### Error Handling

- Toast notifications for validation errors (e.g., invalid coupon) with:
  - Alert icon
  - Descriptive error message
  - Dismiss button
- Required field indicators (*) on form fields

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

---

## Available Slots

Slots where custom components can be injected:

<!-- TODO: Confirm exact slot names from the Ollie Shop SDK documentation -->

| Slot Name | Step | Position | Contract Summary |
|-----------|------|----------|-----------------|
| `cart_header` | Cart | Above item list | Read-only cart data |
| `cart_items` | Cart | Replaces or extends item list | Cart items, quantities, prices |
| `cart_coupon` | Cart | Below item list / above summary | Cart data + dispatch for coupon actions |
| `cart_summary_footer` | Cart | Below totals | Cart totals, item count |
| `identification_header` | Identification | Above email form | User session data |
| `shipping_header` | Shipping | Above address input | Address data |
| `shipping_method` | Shipping | Below method selection | Address data, available methods, selected method |
| `shipping_footer` | Shipping | Below method selection | Selected method, shipping cost |
| `address_form` | Address | Extends or replaces address form | Address fields, receiver fields |
| `payment_header` | Payment | Above payment methods | Order total, available methods |
| `payment_methods` | Payment | Extends payment method list | Available methods, selection state |
| `payment_footer` | Payment | Below payment form | Selected method, payment data |
| `order_summary_footer` | Payment | Below order summary in sidebar | Full order data |

---

## Gap Analysis Template

Use this table when comparing client requirements against native capabilities:

| Requirement | Native Support | Gap | Action |
|-------------|---------------|-----|--------|
| _e.g., "Show loyalty points in cart"_ | No | Full gap | Build custom component for `cart_summary_footer` |
| _e.g., "Coupon input"_ | Yes (built-in) | None | Use native -- no custom component needed |
| _e.g., "Custom shipping labels"_ | Partial (shows method name + ETA but no custom labels) | Partial gap | Extend via `shipping_method` slot |
| _e.g., "Gift card payment"_ | Partial ("Add card or coupon" link exists natively) | Verify | Test native behavior first; extend via `payment_methods` if insufficient |
| _e.g., "Custom address fields (e.g., tax ID)"_ | No | Full gap | Build custom component for `address_form` slot |
| _e.g., "Installment selection"_ | Depends on payment connector | Verify | Check if payment method connector handles installments natively |
| _e.g., "Order review step before payment"_ | No (payment step is the final step) | Full gap | Would require significant flow customization |
