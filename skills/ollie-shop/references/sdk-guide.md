# Ollie Shop SDK Guide

The SDK package is **`@ollie-shop/sdk`**. It exposes the React hooks a custom checkout component uses to read checkout state, dispatch actions, and react to UX events, plus the TypeScript types those hooks return.

Everything a custom component needs is available from the single `@ollie-shop/sdk` entry. Don't reach into `@ollie-shop/react` or `@libs/schemas` directly — those are implementation details that may change.

## Component contract

A custom component is a React function exported as `default` from `./components/<name>/index.tsx`. It takes its own props (configured via the component's `props` JSON in the database, surfaced by the slot) and renders into a template slot.

```tsx
// ./components/FreeShippingBar/index.tsx
import { useCheckoutSession } from "@ollie-shop/sdk";
import styles from "./styles.module.css";

export default function FreeShippingBar({ threshold = 199 }: { threshold?: number }) {
  const { session } = useCheckoutSession();
  const remaining = Math.max(0, threshold - (session.totals?.subtotal ?? 0));

  if (remaining === 0) {
    return <div className={styles.unlocked}>You unlocked free shipping!</div>;
  }
  return (
    <div className={styles.bar}>
      Add R$ {remaining.toFixed(2)} more for free shipping.
    </div>
  );
}
```

No `props` field is injected by the SDK. Whatever your component declares as props is what the slot will pass (defaults coming from the component record's `props` JSON).

---

## Hooks

The SDK exports **eight** hooks. Each one connects to a context provider that the host checkout sets up; if you call a hook outside that provider you get inert defaults.

| Hook | What it gives you | When to reach for it |
|---|---|---|
| `useCheckoutSession` | The parsed checkout session + helpers | Read cart, customer, shipping, payment state. |
| `useCheckoutAction` | Type-safe dispatcher for one checkout action | Mutate the session (add item, update coupon, change shipping, etc.). |
| `usePendingActions` | Which actions are currently in flight | Disable buttons / show spinners while a mutation runs. |
| `useMessages` | Toast-like notifications | Add success/error messages from a component (auto-resolves `ServerError` to translated text). |
| `useStoreInfo` | Static store metadata + the active components map | Need to know the platform, store id, or what other components are mounted. |
| `useCheckoutOrder` | The finalized order (after CREATE_ORDER) | Render the order confirmation screen. |
| `useLogin` | Open / close the login modal | Components that gate behavior behind authentication. |
| `useNavigation` | Register a validator that runs before step transitions | Block "next step" until your component's state is valid. |

### `useCheckoutSession`

```ts
const {
  session,        // CheckoutSession — parsed, platform-agnostic snapshot
  rawSession,     // unknown — raw payload from the platform (e.g. VTEX orderForm)
  updateSession,  // (session, raw) => void — manually replace the session
  sessionValidate,// (session, opts?) => { valid: true } | { valid: false, errors }
  sessionValidity,// last validation result
  revalidate,     // () => void — re-fetch from the platform
  revalidateAsync,// () => Promise<...>
  isFallback,     // true if session came from a cached fallback after an API failure
  fallbackError,  // string when isFallback
  clearFallback,  // () => void
} = useCheckoutSession();
```

The session is **generic** over extensions: `useCheckoutSession<MyExtensions>()` lets you read store-specific custom fields the platform attached. Default extensions type is an empty record.

Read state through `session.*` for the typed shape; only drop down to `rawSession` if you genuinely need a platform-specific field that's not on `CheckoutSession`.

### `useCheckoutAction`

The single dispatcher for every mutation. The action name is type-checked, the payload is inferred from the action, and the response is automatically reconciled with the session (except for `REQUEST`, `SIMULATE_SESSION`, and `CREATE_ORDER`, which return their own shape).

```ts
const { execute, executeAsync, isPending, error } = useCheckoutAction(
  "ADD_ITEMS",
  {
    onSuccess: (updatedSession, input) => { /* ... */ },
    onError: ({ serverError, validationErrors }) => { /* ... */ },
  },
);

execute([{ id: "sku-1", quantity: 2 }]); // input is typed per action
```

**Action names** (the keys of `ActionMap`):

| Action | Input | Use for |
|---|---|---|
| `ADD_ITEMS` | `CartItemInput[]` | Add SKUs to cart. |
| `REMOVE_ITEMS` | `number[]` (indexes) | Remove items by their cart index. |
| `UPDATE_ITEMS_QUANTITY` | `{ index, quantity }[]` | Change quantity of existing items. |
| `UPDATE_COUPONS` | `string[]` | Apply / replace coupon codes. |
| `UPDATE_CUSTOMER_DETAILS` | `Partial<CustomerData>` | Name, email, phone, document. |
| `UPDATE_CUSTOMER_PREFERENCES` | `Partial<CustomerPreferences>` | Marketing opt-ins, etc. |
| `UPDATE_SHIPPING_PACKAGES` | `ShippingPackageInput[]` | Pick a shipping option per package. |
| `UPDATE_SHIPPING_ADDRESSES` | `ShippingAddressInput[]` | Change delivery address(es). |
| `UPDATE_PAYMENT_METHODS` | `PaymentMethodInput[]` | Choose payment(s). |
| `UPDATE_GIFT_CARDS` | `GiftCardInput[]` | Apply / remove gift cards. |
| `SIMULATE_SESSION` | `SimulateCheckoutSessionInput` | Preview what the session would look like with hypothetical inputs (does not commit). |
| `CREATE_NEW_SESSION` | `unknown` | Reset to a fresh session. |
| `CREATE_ORDER` | `CreateOrderInput` | Finalize and create the order. |
| `REQUEST` | `{ url, method?, headers?, body?, revalidate? }` | Escape hatch for arbitrary HTTP through the checkout backend (e.g. calling a custom endpoint your function exposes). Pass `revalidate: true` to re-pull the session after. |

`isPending` and the `usePendingActions` hook below both let you reflect the action's loading state.

### `usePendingActions`

```ts
const { hasPendingActions, pendingActions } = usePendingActions();
// pendingActions: ActionType[] — e.g. ["ADD_ITEMS", "UPDATE_COUPONS"]
```

Use this when one component needs to disable itself while a different component's action is in flight (e.g. don't let the user submit the order while a coupon update is still pending).

### `useMessages`

```ts
const { messages, addMessage, removeMessage, clearAll } = useMessages();

// direct
addMessage({ type: "success", content: "Coupon applied" });

// from a ServerError — the message is auto-resolved through the translation chain
addMessage({ type: "error", error: result.error.serverError });
```

`messages` is the current list (each `{ id, type, content, title? }`). The host renders them; you just push/pop.

### `useStoreInfo`

```ts
const { platform, platformStoreId, components } = useStoreInfo();
```

Static metadata for the store and the resolved list of `CustomComponent`s being rendered. The hook also transparently merges in any Studio-mode components when the developer is previewing locally via `ollieshop start`.

### `useCheckoutOrder`

Only meaningful on the order confirmation screen.

```ts
const { order, raw } = useCheckoutOrder();
```

`order` is the parsed `CheckoutOrder`; `raw` is whatever the platform returned. The provider resolves a promise internally, so it suspends until the order is ready.

### `useLogin`

```ts
const { openLogin, closeLogin, isLoginRequired, loginTitle, loginRef } = useLogin();

openLogin({ isRequired: true, title: "Sign in to continue" });
```

`isRequired` disables backdrop close. Use this in components that gate a behavior on authentication (e.g. "save address" needs a logged-in customer).

### `useNavigation`

```ts
useNavigation(() => {
  // return false to block the next-step transition
  return form.isValid;
});
```

Registers a single validator that the host calls before advancing to the next checkout step. The most recently registered validator wins; returning `true` (or not calling the hook at all) allows the transition. The hook also returns `{ unregisterStepValidator }` if you need to clear the validator imperatively.

---

## Styling

Components ship their own CSS modules:

```
./components/FreeShippingBar/
├── index.tsx
├── styles.module.css
└── meta.json
```

Import with `import styles from "./styles.module.css"`. The bundler treats `react`, `react-dom`, `next`, `next-intl`, and `@ollie-shop/sdk` as external, so you should not bundle them yourself — they're provided by the host checkout at runtime.

Avoid importing global CSS or third-party CSS-in-JS frameworks unless you've confirmed they're compatible with the host's runtime. CSS modules are the safe default.

---

## What is intentionally not exposed

The host checkout has additional providers internally (analytics, Studio gateway, reCAPTCHA, etc.). Those are not part of the public SDK and may change without notice. If you find yourself wanting one of them, it's a signal to either:
- Solve the problem with the existing 8 hooks plus a `REQUEST` action, or
- Talk to the Ollie team about promoting the missing capability to the SDK.
