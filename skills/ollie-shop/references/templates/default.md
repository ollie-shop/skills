# Default Template - Native Capabilities

> **PLACEHOLDER** -- Replace with the actual native template documentation.

This file describes what the **default checkout template** provides out of the box. Use it during the Template Gap Analysis step to determine what already works natively vs. what requires a custom component.

## Table of Contents
1. [Checkout Flow](#checkout-flow)
2. [Built-in Business Rules](#built-in-business-rules)
3. [UX Behaviors](#ux-behaviors)
4. [Available Slots](#available-slots)

---

## Checkout Flow

The default template implements the following checkout steps natively:

<!-- TODO: Document the actual steps -->

| Step | Description | Native Behavior |
|------|-------------|-----------------|
| Cart | Cart review and editing | Item list, quantity update, remove item, subtotal calculation |
| Identification | User identification | Login/register, guest checkout, email capture |
| Shipping | Address and delivery | Address form, shipping method selection, cost calculation |
| Payment | Payment selection | Payment method list, credit card form, installment selection |
| Confirmation | Order review and placement | Order summary, place order button, confirmation screen |

---

## Built-in Business Rules

These rules are **already handled** by the native template. Custom components should not re-implement them unless overriding is explicitly required.

<!-- TODO: Replace with actual native rules -->

| Rule | Description | Step | Behavior |
|------|-------------|------|----------|
| Cart minimum | Enforces minimum cart value to proceed | Cart | Disables checkout button, shows warning |
| Address validation | Validates shipping address fields | Shipping | Inline field validation, ZIP code lookup |
| Shipping calculation | Fetches shipping options from platform | Shipping | Calls platform API, displays options with prices and ETAs |
| Payment validation | Validates payment fields before submission | Payment | Card number format, expiry date, CVV |
| Coupon application | Native coupon/promo code input | Cart | Input field, apply button, discount display |
| Order total | Calculates subtotal + shipping - discounts | All | Live-updating total across steps |
| Stock validation | Checks item availability before placing order | Confirmation | Alerts if items went out of stock during checkout |
| Installment rules | Displays installment options based on payment method and total | Payment | Platform-driven installment calculation |

---

## UX Behaviors

Native UX patterns the template provides without custom components:

<!-- TODO: Replace with actual UX behaviors -->

### Navigation
- Step-by-step linear flow with back/forward navigation
- Step indicator showing current position
- Collapsible summary of completed steps

### Responsiveness
- Mobile-first layout with breakpoints at 768px and 1024px
- Sticky order summary on desktop, collapsible on mobile

### Loading States
- Skeleton loaders during shipping/payment calculation
- Disabled buttons during async operations
- Optimistic updates for cart quantity changes

### Error Handling
- Inline field validation with error messages
- Toast notifications for async errors (network, API failures)
- Retry mechanisms for transient failures

### Accessibility
- Keyboard navigable checkout flow
- ARIA labels on all interactive elements
- Screen reader announcements for step transitions and errors

---

## Available Slots

Slots where custom components can be injected:

<!-- TODO: Replace with actual slot definitions -->

| Slot Name | Step | Position | Contract Summary |
|-----------|------|----------|-----------------|
| `cart_header` | Cart | Above item list | Read-only cart data |
| `cart_coupon` | Cart | Below item list | Cart data + dispatch for coupon actions |
| `cart_summary_footer` | Cart | Below totals | Cart totals, item count |
| `identification_header` | Identification | Above form | User session data |
| `shipping_method` | Shipping | Below address form | Address data, available methods |
| `shipping_footer` | Shipping | Below method selection | Selected method, shipping cost |
| `payment_header` | Payment | Above payment methods | Order total, available methods |
| `payment_footer` | Payment | Below payment form | Selected method, payment data |
| `order_summary_footer` | Confirmation | Below order summary | Full order data |

---

## Gap Analysis Template

Use this table when comparing client requirements against native capabilities:

| Requirement | Native Support | Gap | Action |
|-------------|---------------|-----|--------|
| _e.g., "Show loyalty points"_ | No | Full gap | Build custom component for `cart_summary_footer` |
| _e.g., "Coupon input"_ | Yes (built-in) | None | Use native -- no custom component needed |
| _e.g., "Custom shipping labels"_ | Partial (shows options but no custom labels) | Partial gap | Extend via `shipping_method` slot |
