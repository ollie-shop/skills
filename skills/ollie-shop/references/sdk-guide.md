# Ollie Shop SDK Guide

> **PLACEHOLDER** -- Replace this content with the actual SDK documentation.

## Table of Contents
1. [Component Interface](#component-interface)
2. [SDK Hooks](#sdk-hooks)
3. [State Management](#state-management)
4. [Styling & Design Tokens](#styling--design-tokens)
5. [Platform Abstraction](#platform-abstraction)
6. [Error Handling](#error-handling)
7. [Testing Components](#testing-components)
8. [Best Practices](#best-practices)

---

## Component Interface

Every Ollie Shop component exports a default function that conforms to the slot contract.

```typescript
// TODO: Replace with actual SDK types and interfaces

import type { SlotContext, ComponentProps } from '@ollie-shop/sdk';

export default function MyComponent({ context, sdk }: ComponentProps) {
  // context: slot-specific data (cart, user, store config)
  // sdk: methods for interacting with the checkout orchestrator
  return (
    // your component JSX
  );
}
```

<!-- TODO: Document full ComponentProps interface, SlotContext variations per slot -->

---

## SDK Hooks

```typescript
// TODO: Replace with actual hook signatures

// Access cart data
const cart = sdk.useCart();

// Access current user session
const user = sdk.useUser();

// Access store configuration
const config = sdk.useStoreConfig();

// Dispatch actions to the checkout orchestrator
const dispatch = sdk.useDispatch();

// Subscribe to checkout events
sdk.useEvent('checkout:step-change', (step) => { /* ... */ });
```

<!-- TODO: Document all available hooks, their return types, and when to use each -->

---

## State Management

<!-- TODO: Document how components manage local vs. shared state -->
<!-- Topics: SDK state store, component-local state, cross-component communication -->

---

## Styling & Design Tokens

<!-- TODO: Document the design token system -->
<!-- Topics: available tokens, how to access them, CSS modules vs. Tailwind, responsive behavior -->

---

## Platform Abstraction

The SDK abstracts the underlying e-commerce platform. Components never interact with VTEX (or any other platform) directly.

<!-- TODO: Document the abstraction layer -->
<!-- Topics: what's abstracted (cart, payments, shipping), how to extend for new platforms -->

---

## Error Handling

<!-- TODO: Document error handling patterns -->
<!-- Topics: SDK error types, fallback rendering, error boundaries, logging -->

---

## Testing Components

<!-- TODO: Document testing setup -->
<!-- Topics: test utilities provided by SDK, mocking slot context, integration testing with ollie dev -->

---

## Best Practices

### Do
- Use SDK hooks for all data access -- never fetch directly from platform APIs
- Handle loading and error states in every component
- Use design tokens for styling consistency
- Keep components focused on a single responsibility
- Test with multiple cart states (empty, single item, many items, edge cases)

### Don't
- Don't hard-code platform-specific logic (e.g., VTEX API calls)
- Don't store sensitive data in component state
- Don't rely on global DOM manipulation -- work within the slot boundary
- Don't assume a specific template layout -- use slot constraints
- Don't skip accessibility (a11y) -- checkout must be accessible

<!-- TODO: Expand with real-world patterns and anti-patterns from production -->
