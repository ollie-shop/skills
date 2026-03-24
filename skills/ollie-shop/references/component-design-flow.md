# Component Design Flow

A guided workflow for going from a design reference (Figma, screenshot, business rules, live website) to a fully implemented Ollie Shop component.

The goal is not just to write code -- it's to help the developer build a **mental model** of what the component does, why it exists, and how it fits into the checkout flow.

## Table of Contents
1. [Step 1: Gather Context](#step-1-gather-context)
2. [Step 2: Identify Business Rules](#step-2-identify-business-rules)
3. [Step 3: Map to Slots](#step-3-map-to-slots)
4. [Step 4: Design the Component Contract](#step-4-design-the-component-contract)
5. [Step 5: Check the Component Library](#step-5-check-the-component-library)
6. [Step 6: Implement](#step-6-implement)
7. [Step 7: Validate & Deploy](#step-7-validate--deploy)

---

## Step 1: Gather Context

Before writing any code, understand **what you're building and why**.

### From Figma files
- Use the Figma MCP to read the design file
- Extract: layout structure, spacing, colors, typography, interactive states
- Identify which parts are **static** (labels, images) vs. **dynamic** (data-driven, conditional)
- Note any responsive breakpoints or mobile-specific layouts

### From screenshots
- Analyze the visual layout and identify distinct UI regions
- Compare against known Ollie Shop template slots
- Flag ambiguities -- screenshots don't show hover states, error states, or responsive behavior

### From documented business rules
- Query the business rules API (see `references/business-rules-api.md`)
- Extract conditions, actions, and data dependencies
- Map each rule to a specific checkout moment (cart, shipping, payment, confirmation)

### From a live website
- Use browser devtools or screenshots to capture the current behavior
- Document: what the user sees, what happens on interaction, what data drives the UI
- Identify what the **native template already handles** vs. what needs a **custom component**

---

## Step 2: Identify Business Rules

Decompose the component's behavior into explicit rules:

| Rule | Condition | Action | Data Source |
|------|-----------|--------|-------------|
| Example: Free shipping badge | Cart total > $100 | Show "Free Shipping" badge | `cart.total` via `sdk.useCart()` |
| Example: Seller code discount | Valid seller code entered | Apply discount, show confirmation | Custom input + `sdk.useDispatch()` |

For each rule, ask:
- **Where does the data come from?** (SDK hook, user input, store config)
- **What happens on edge cases?** (empty value, invalid input, network failure)
- **Is this rule platform-specific?** (If yes, it should go through the SDK abstraction)

---

## Step 3: Map to Slots

Determine which template slot(s) this component belongs in.

### Decision criteria:
1. **Functional fit** -- What does the component do? (coupon input -> `cart_coupon` slot)
2. **Flow position** -- Where in the checkout journey does it appear?
3. **Data availability** -- Does the slot's contract provide the data this component needs?
4. **Constraints** -- Does the slot impose size or behavior limits?

### Common slot patterns:
<!-- TODO: Replace with actual template slots -->

| Slot Name | Position | Typical Use |
|-----------|----------|-------------|
| `cart_header` | Top of cart | Announcements, promotions |
| `cart_coupon` | Cart section | Coupon/discount inputs |
| `cart_summary_footer` | Below cart summary | Additional totals, badges |
| `shipping_method` | Shipping step | Custom shipping options |
| `payment_header` | Payment step | Payment instructions, warnings |
| `order_summary_footer` | Order review | Final confirmations, legal text |

If no existing slot fits, flag this -- it may require a template extension.

---

## Step 4: Design the Component Contract

Before coding, define the component's interface:

```typescript
// What does this component receive from the slot?
interface SlotContext {
  // TODO: specific to the chosen slot
}

// What actions does this component perform?
interface ComponentActions {
  // e.g., applyDiscount(code: string): Promise<void>
}

// What state does this component manage internally?
interface ComponentState {
  // e.g., inputValue: string, isLoading: boolean, error: string | null
}
```

This contract becomes the blueprint for implementation.

---

## Step 5: Check the Component Library

Before building from scratch, check `assets/component-library.csv` for existing patterns.

Look for:
- **Exact matches** -- A component that does what you need
- **Partial matches** -- A component that handles a similar pattern (e.g., any input + validation + API call)
- **Composable pieces** -- Multiple simple components that combine into what you need

If a match exists, start from that pattern and adapt it.

---

## Step 6: Implement

With the mental model clear, write the component:

1. **Scaffold** -- Create the component file in the correct path
2. **Wire SDK hooks** -- Connect to the data sources identified in Step 2
3. **Implement business rules** -- Translate the rules table into conditional logic
4. **Style** -- Use design tokens from the template; match the Figma/screenshot reference
5. **Handle states** -- Loading, error, empty, success
6. **Accessibility** -- Labels, ARIA attributes, keyboard navigation

---

## Step 7: Validate & Deploy

1. Run `ollie validate` to check against the slot contract
2. Test with `ollie dev` in the local development server
3. Test edge cases: empty cart, logged-out user, slow network, mobile viewport
4. Create a version: `ollie component version create`
5. Deploy: `ollie deploy`
6. Verify in the target store
