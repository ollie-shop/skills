# Ollie Shop SDK Guide

Reference for building custom checkout components using the `@ollie-shop/sdk`.

## Table of Contents

1. [Component Interface](#component-interface)
2. [SDK Hooks](#sdk-hooks)
3. [State Management](#state-management)
4. [Styling & Design Tokens](#styling--design-tokens)
5. [Platform Abstraction](#platform-abstraction)
6. [Error Handling](#error-handling)
7. [Known-absent APIs](#known-absent-apis)
8. [Best Practices](#best-practices)

---

## Component Interface

Every Ollie Shop component exports a default function matching the `CustomComponent` type from `@ollie-shop/sdk`.

### Component Signature

```tsx
import { useCheckoutSession } from "@ollie-shop/sdk";
import type { ReactNode } from "react";

interface MyProps {
  myProp?: string;
  children?: ReactNode;
}

export default function MyComponent({ myProp = "default", children }: MyProps) {
  const { session } = useCheckoutSession();

  // Option A — augment: keep the native default and add on top
  // return <>{children}<MyExtra /></>;

  // Option B — replace: ignore children and take over the slot
  // return <MyOwnUI />;
}
```

**Every prop arrives FLAT at the top level of the signature.** The runtime renders the component as:

```tsx
// packages/react/src/components/slot/custom-component.tsx
<Component {...component.props} {...contextProps}>{children}</Component>
```

so the component receives, all on the same level:

- **admin-configured props** — the component record's `props` JSON (set at `component create`, overridable in the Admin), spread one key at a time
- **slot context props** — runtime values the slot forwards (e.g. `item`, `onToggleSummary`)
- `children` — the slot's native default UI.

> **There is no nested `props` object.** `function MyComponent({ props })` destructures a key that never exists: `props` is always `undefined`, every admin value is silently dropped, and the component runs on its code defaults forever — with no error to point at. This is the single most common wiring bug. If a value configured in the Admin "isn't arriving", this is why.

```tsx
// ❌ WRONG — `props` is undefined at runtime; myProp is always the default
export default function MyComponent({ props, children }) {
  const { myProp = "default" } = props || {};
}

// ✅ RIGHT — destructure admin props directly
export default function MyComponent({ myProp = "default", children }) {}

// ✅ ALSO RIGHT — collect the admin props via rest when there are many
export default function MyComponent({ children, ...props }: MyProps) {
  const { myProp = "default" } = props; // rest object, never undefined — no `?? {}` needed
}
```

The legacy `CustomComponent<P>` type from `@ollie-shop/sdk` still types the callback as `({ props, children })`. **Do not follow it** — it does not match what the runtime passes. Type the signature yourself with a plain interface, as above.

#### Children: replace vs augment

A custom component always **receives** the slot's default UI as `children`. What it does with them is a deliberate choice:

| Strategy | What the component does | When to use |
|---|---|---|
| **Replace** | Ignores `children` and renders its own UI from scratch | The component owns the entire slot concern (e.g. a coupon input that fully substitutes the native coupon form). |
| **Augment** | Renders `{children}` plus extra UI around/below | Keep the native behavior and add on top (e.g. a free-shipping badge above the native totalizer). **Default choice when in doubt** — preserves native UX. |

Rule of thumb: if omitting your component would leave a gap in the UX, render `{children}`. If your component *is* the UX, don't.

To read cart, customer, shipping, payment, totals, extensions, etc., always use the **`useCheckoutSession()`** hook (see below). Do **not** destructure `cart` from the function arguments — it is not provided that way.

### Slot context props vs admin props

Some slots forward `context_props` — runtime values the slot passes to its custom component (e.g. `item` on `cart_item_addons`, `item`/`formatPrice`/`onRemoveItem` on `cart_item`). The authoritative list of what a given slot forwards lives in that component's `assets/components/<id>/INSTRUCTIONS.md` and in `assets/checkout-slots-data.yaml` (`context_props:`).

**Context props and admin props share the same flat namespace** — both land at the top level of the signature, alongside `children`. Admin props are spread first, so a context prop of the same name wins. Pick admin prop names that don't collide with the slot's `context_props`.

```tsx
// cart_item_addons — forwards `item` as a context prop
export default function MyAddon({ item, children }) {
  if (item?.available === false) return null;
  return (
    <>
      {children}
      <MyExtra />
    </>
  );
}
```

**`item` runtime shape** (forwarded by any slot that declares `item` in its `context_props`):

```ts
type CartItem = {
  id: string; // SKU code (NOT a productId / refId)
  sellerId?: string; // marketplace seller/vendor id (slug-like, e.g. "acme-store")
  name: string; // display name
  variant?: string; // e.g. "Pack of 1", "Size Large", "Red"
  brand?: string; // brand DISPLAY name (e.g. "Nike") — NOT a brandId
  category?: string; // category DISPLAY name (e.g. "Electronics") — NOT a categoryId
  price: number; // current/sale price in MINOR units (cents)
  originalPrice: number; // non-sale price in minor units
  quantity: number;
  available: boolean;
  index: number; // position in the cart
  image: string; // image URL
  uniqueId: string; // cart-line unique id (distinct from `id` — two lines of same SKU can exist)
  url?: string; // PDP URL
  variantDetails?: Record<string, string>; // e.g. { color: "Red", size: "M" }
};
```

Typing tip: declare context props and admin props in ONE flat interface:

```tsx
interface SlotProps {
  item?: CartItem; // context prop — forwarded by the slot
  giftWrapSkuId?: string; // admin prop — set on the DB component record
  children?: ReactNode; // the slot's native default UI
}

export default function MyAddon({
  item,
  giftWrapSkuId = "35127",
  children,
}: SlotProps) {
  // ...
}
```

When the component has many admin props, collect them with a rest element and destructure with defaults in one place:

```tsx
export default function MyAddon({ children, ...props }: SlotProps) {
  const { giftWrapSkuId = "35127", item } = props;
}
```

**Verifying what a slot actually gives you:** `console.log` the whole props object (`{ ...props }` or `arguments[0]`) and inspect every key — then check the component's `INSTRUCTIONS.md` and `checkout-slots-data.yaml` for the documented context-prop shape. If a key you configured in the Admin isn't in that log, the value never reached the component (wrong prop name, or the component record's `props` JSON doesn't carry it) — it is NOT hiding under a nested `props` key.

**Display names vs platform IDs (pitfall):** context-prop fields like `item.category`, `item.brand`, `item.seller` arrive as **human-readable strings** (e.g. `"Shoes"`), NOT as the platform-native IDs (e.g. VTEX `categoryId = 123`). Substituting the name into a URL that expects an ID — `fq=C:{categoryId}` on VTEX Search, brand/seller filters, etc. — will silently fail or return wrong results.

**Where to get the ID.** The SDK does **not** own platform catalog data. `useCheckoutAction("REQUEST")` is a passthrough — it proxies your HTTP call to whatever absolute URL you give it and returns the raw response untouched. For anything outside the `CheckoutSession` (product catalog, categoryId lookups, brand metadata, stock), hit the **platform API directly** via `REQUEST` and read the **platform's own docs** for the endpoint + response shape. There is no SDK-level abstraction of those responses.

**Practical options:**

1. **Name→id map in the component record's `props`** (set via `component create --props`, or the Admin) — admin-configurable per merchant, avoids a live lookup. Best when the category set is small and stable.
2. **Live lookup via `REQUEST` to the platform's catalog API** — e.g. VTEX `/api/catalog_system/pub/category/tree/N` to resolve a name to an id. Adds a round-trip and you own the parsing.
3. **Fallback to the name-based filter the platform offers** — e.g. VTEX `ft=<name>` instead of `fq=C:{id}`. Cheapest, but matching is fuzzier.

### File Layout & Registration

The bundler discovery rules (`components/*/index.tsx` glob, `commons/` sibling for import-only modules), the extraction thresholds for utils and subcomponents, and both registration patterns (per-folder `meta.json` vs root `ollie.json` `components` map with schema) live in **[`references/component-anatomy.md`](component-anatomy.md)** — that is the single source of truth. Do not duplicate them here.

The only SDK-specific rule to remember here: the component entry `index.tsx` **must** default-export a React function whose props signature is flat (see the previous section) — the runtime spreads admin props + slot context props at the same level of the destructure, alongside `children`.

---

## SDK Hooks

All hooks are imported from `@ollie-shop/sdk`. These are the complete set of hooks available to custom components.

### Defensive reads (applies to every hook)

Hook return values may be **`undefined`** during early renders (HMR rebuild, pre-hydration, provider race). The SDK type signatures show non-optional returns, but the runtime can give you `undefined` and a direct destructure will throw `TypeError: Cannot destructure property 'X' of undefined`, killing the slot.

**Never destructure a hook result directly.** Always coalesce first, treat every field as optional:

```ts
// ❌ Will crash if the hook returns undefined during the first render
const { platformStoreId } = useStoreInfo();

// ✅ Safe
const storeInfo = useStoreInfo() ?? {};
const platformStoreId =
  (storeInfo as { platformStoreId?: string }).platformStoreId ?? "";

// ✅ Same pattern for useCheckoutSession, useMessages, etc.
const { session } = useCheckoutSession() ?? {};
const cartItems = session?.cartItems ?? [];
```

If the field is missing, render the loading or empty state — do not throw.

### `useCheckoutSession`

```ts
function useCheckoutSession<
  Extensions extends Record<string, unknown> = Record<string, unknown>,
>(): CheckoutSessionContextValue<Extensions>;
```

**Returns:**

| Field             | Type                                                            | Description                               |
| ----------------- | --------------------------------------------------------------- | ----------------------------------------- |
| `session`         | `CheckoutSession<Extensions>`                                   | Parsed, platform-agnostic session         |
| `rawSession`      | `unknown`                                                       | Raw platform data (VTEX order form, etc.) |
| `updateSession`   | `(session, rawSession) => void`                                 | Imperatively override session in context  |
| `sessionValidity` | `ValidationSuccess \| ValidationFailure \| undefined`           | Live Zod validation result                |
| `sessionValidate` | `(session, options?) => ValidationSuccess \| ValidationFailure` | On-demand validation                      |
| `revalidate`      | `() => void`                                                    | Re-fetch session from platform            |
| `revalidateAsync` | `() => Promise<...>`                                            | Same, awaitable                           |
| `isFallback`      | `boolean`                                                       | True when session is from stale cache     |
| `fallbackError`   | `string \| undefined`                                           | Error message when `isFallback` is true   |
| `clearFallback`   | `() => void`                                                    | Reset fallback state                      |

**Session Shape (key fields):**

```ts
CheckoutSession {
  id: string
  cartItems: CartItem[]
  customer?: { email, firstName, lastName, phone, document, addresses[] }
  shipping?: { packages[], availableQuotes[], addresses[], availableCountries[] }
  payment?: { availableMethods[], selectedPayments[], savedCards[] }
  totals: { total, items, shipping?, tax?, discounts?, interest? }
  locale: { language, currency }
  campaign?: { coupons[], giftCards[] }
  extensions?: Record<string, unknown>
  readOnly: boolean
}
```

**The shape above is a sketch, not the contract. Read the real type from the installed package.**

```ts
import type { CheckoutSession } from "@ollie-shop/sdk";
```

`@ollie-shop/sdk` re-exports every schema type, so the customer's editor and `tsc` resolve the exact shape for the SDK version *they* have installed. `CheckoutSession`, `CheckoutOrder`, `FulfillmentOrder`, `SelectedPaymentOrder`, `ShippingAddress` and `StoreInfo` are all importable. Hover a field, or open `node_modules/@ollie-shop/sdk/dist/index.d.ts`, and you get the union members and optionality that no hand-written table in this skill can keep current. Never guess a field name, and never trust a snapshot here over the installed types.

### `useCheckoutAction`

The primary mutation API. All checkout state changes flow through typed server actions.

```ts
function useCheckoutAction<Action extends ActionType, Input, ResponseType>(
  actionType: Action,
  callback?: {
    onSuccess?: (data?: ResponseType, input?: Input) => void;
    onError?: ({ serverError, validationErrors }) => void;
  },
): {
  execute: (input: Input) => void;
  executeAsync: (input: Input) => Promise<ResponseType | undefined>;
  isPending: boolean;
  error?: {
    serverError?: ServerError;
    validationErrors?: ValidatorErrors<Input>;
  };
};
```

**Available Actions:**

| Action name                   | Input type                                       | Effect                                                                                                                                                                                                            |
| ----------------------------- | ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ADD_ITEMS`                   | `CartItemInput[]`                                | Adds items to the cart                                                                                                                                                                                            |
| `REMOVE_ITEMS`                | `number[]` (item indexes)                        | Removes items from the cart                                                                                                                                                                                       |
| `UPDATE_ITEMS_QUANTITY`       | `{ index: number; quantity: number }[]`          | Updates quantities                                                                                                                                                                                                |
| `UPDATE_COUPONS`              | `string[]` (coupon codes)                        | Applies/removes coupons                                                                                                                                                                                           |
| `UPDATE_CUSTOMER_DETAILS`     | `Partial<CustomerData>`                          | Updates customer (email, name, etc.)                                                                                                                                                                              |
| `UPDATE_CUSTOMER_PREFERENCES` | `Partial<CustomerPreferences>`                   | Updates customer preferences                                                                                                                                                                                      |
| `UPDATE_SHIPPING_PACKAGES`    | `ShippingPackageInput[]`                         | Selects shipping method for packages                                                                                                                                                                              |
| `UPDATE_SHIPPING_ADDRESSES`   | `ShippingAddressInput[]`                         | Sets/updates shipping addresses                                                                                                                                                                                   |
| `UPDATE_PAYMENT_METHODS`      | `PaymentMethodInput[]`                           | Selects payment method(s)                                                                                                                                                                                         |
| `UPDATE_GIFT_CARDS`           | `GiftCardInput[]`                                | Applies/removes gift cards                                                                                                                                                                                        |
| `SIMULATE_SESSION`            | `SimulateCheckoutSessionInput`                   | Simulates session (no state update)                                                                                                                                                                               |
| `REQUEST`                     | `{ url, method?, headers?, body?, revalidate? }` | Raw HTTP request via server action. **`url` MUST be absolute** (`https://...`) — relative paths starting with `/` will fail because the request runs server-side and has no storefront origin to resolve against. See response envelope note below. |
| `CREATE_NEW_SESSION`          | `unknown`                                        | Creates a fresh checkout session                                                                                                                                                                                  |

**`REQUEST` response envelope.** The action wraps the platform's response in its own metadata envelope:

```ts
type RequestResponse<T> = {
  data: T;                          // platform's native payload
  status: number;                   // HTTP status from the platform
  headers: [string, string][];      // response headers
};
```

If the upstream platform itself wraps responses (VTEX does on many endpoints), you get **double-nesting**: `response.data.data` is the actual payload. Always unwrap explicitly rather than passing `response` straight into a parser. Pattern:

```ts
const response = await execute({ url: "https://acme.vtexcommercestable.com.br/api/...", method: "GET" });
const platformPayload = response?.data;      // unwrap RequestResponse
const actualData = platformPayload?.data ?? platformPayload; // unwrap platform envelope if present
// TODO(?): does this endpoint double-wrap? confirm against platform docs.
```

**Input Type Definitions:**

```ts
type CartItemInput = {
  id: string; // product/SKU identifier
  quantity: number;
  sellerId?: string; // marketplace seller
};

type PaymentMethodInput = {
  methodId: string; // matches PaymentMethod.id
  referenceValue: number; // amount in minor units (cents)
  installments?: number; // defaults to 1 if omitted
  accountId?: string; // saved card account ID
};

type ShippingAddressInput = {
  id?: string;
  type?: "home" | "billing" | "work" | "pick_up" | "search" | "other";
  street?: string;
  number?: string;
  complement?: string;
  reference?: string;
  neighborhood?: string;
  city?: string;
  country?: string;
  stateOrProvince?: string;
  postalCode?: string;
  receiverName?: string;
};

type ShippingPackageInput = {
  id: string; // package identifier
  items: number[]; // cart item indexes
  addressId?: string;
  selectedTimeSlotId?: string; // delivery time slot
  type?: "delivery" | "pick_up";
};

type GiftCardInput = {
  code: string;
  provider?: string;
};

type CustomerData = {
  id?: string;
  email?: string;
  firstName?: string;
  lastName?: string;
  document?: string;
  phone?: string;
  addresses?: CustomerAddress[];
};

type CustomerPreferences = {
  saveData?: boolean;
  locale?: string;
};
```

**Usage example (split payments):**

```ts
const { execute } = useCheckoutAction("UPDATE_PAYMENT_METHODS");
execute([{ methodId: "201", referenceValue: orderTotal, installments: 3 }]);
```

### `useCheckoutOrder`

```ts
function useCheckoutOrder(): {
  order: CheckoutOrder;
  raw: unknown;
};
```

Read the confirmed order on the order-confirmation page (`/order/[orderId]`).

The order is a shell around `fulfillmentOrders`; read the fields off `CheckoutOrder` and `FulfillmentOrder` from `@ollie-shop/sdk`. Single-fulfillment is the common case, but a store that splits by seller produces several, so iterate rather than assuming `[0]`.

`SelectedPaymentOrder` carries `total` and `installments` but no per-installment amount, so a "3x de R$ 99,90" line has to divide.

### `useMessages`

```ts
function useMessages(): {
  messages: Message[];
  addMessage: (message: Omit<Message, "id">) => void;
  removeMessage: (messageId: string) => void;
  clearAll: () => void;
};

interface Message {
  id: string;
  type: "info" | "success" | "warning" | "error";
  content: React.ReactNode;
  title?: React.ReactNode;
}
```

### `useStoreInfo`

```ts
function useStoreInfo(): Omit<StoreInfo, "messages">;
```

The hook can return `undefined` during early renders — apply the defensive read pattern (see top of §SDK Hooks) before destructuring.

The fields below are the documented set. The runtime payload may carry additional keys not listed here (e.g. `logo`, `versionId`); inspect `console.log(useStoreInfo())` if you suspect a field exists but isn't documented yet.

| Field             | Type                                   | Notes                                                                                                                                                                                                                                                                                                                                                                                                              |
| ----------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `storeId`         | `string \| undefined`                  | Ollie store identifier                                                                                                                                                                                                                                                                                                                                                                                             |
| `logo`            | `string \| undefined`                  | Store logo URL. This is the supported way for a custom header or branded component to render the store's logo — do not take it as an admin prop.                                                                                                                                                                                                                                                                    |
| `versionId`       | `string \| undefined`                  | Active version of the store                                                                                 
| `platformStoreId` | `string`                               | Platform-native store identifier. On VTEX this is the **account name** (e.g. `"acme-store"`) used to build `https://{platformStoreId}.vtexcommercestable.com.br/...` URLs. Sourced from `ollie.json` at the store root. Prefer this over exposing a `vtexAccount` prop on components — every platform-specific catalog call (best sellers, category tree, brand lookup, stock, etc.) should read it from here. |
| `platform`        | `string`                               | e.g. `"vtex"`                                                                                                                                                                                                                                                                                                                                                                                                      |
| `template`        | `TemplateType \| undefined`            | Active template key                                                                                                                                                                                                                                                                                                                                                                                                |
| `theme`           | `Record<string, string> \| undefined`  | Design token overrides                                                                                                                                                                                                                                                                                                                                                                                             |
| `settings`        | `Record<string, unknown> \| undefined` | Arbitrary store settings                                                                                                                                                                                                                                                                                                                                                                                           |
| `props`           | `StoreInfoProps \| undefined`          | Flags, shipping, integrations, steps                                                                                                                                                                                                                                                                                                                                                                               |
| `components`      | `CustomComponent[] \| undefined`       | Registered slot components                                                                                                                                                                                                                                                                                                                                                                                         |

### `usePendingActions`

```ts
function usePendingActions(): {
  hasPendingActions: boolean;
  pendingActions: ActionType[];
  addActionNameForLoading: (action: ActionType, isEarly?: boolean) => void;
  removeActionNameForLoading: (action: ActionType) => void;
};
```

Use to show loading states while actions are in flight.

### `useLogin`

```ts
function useLogin(): {
  openLogin: (options?: { isRequired?: boolean; title?: string }) => void;
  closeLogin: () => void;
  isLoginRequired: boolean;
  loginTitle: string | null;
};
```

Opens and closes the checkout's login modal. The host checkout renders and wires the modal itself, so a custom component only decides **when** it opens. Call `openLogin({ isRequired: true })` when the flow cannot continue without an identified customer, and read `isLoginRequired` or `loginTitle` if the component needs to reflect that state.

The context object also carries a `loginRef`. That is provider-internal plumbing for the host's `<dialog>`, not part of the customization surface. Do not destructure or attach it.

### `useNavigation`

```ts
function useNavigation(validator: () => boolean): {
  unregisterStepValidator: () => void;
};
```

Registers a **step validator** — a function the checkout calls before advancing to the next step. Return `true` to allow navigation, `false` to block it (e.g. your custom step has invalid input). The validator is registered on mount and torn down on unmount; call the returned `unregisterStepValidator` to remove it early.

This is **not** an imperative router — there is no `push`/`navigate`/`back`. It only gates step progression. For actually moving between built-in steps, use the `prev`/`next` props the host forwards to custom step-page slots (when a custom step's `page` string is used as a slot id, e.g. `<Slot id="LoyaltyStep" prev={...} next={...} />`).

```tsx
useNavigation(() => {
  // block "continue" until the user accepts the terms
  return termsAccepted;
});
```

---

## State Management

### SDK State (Shared Across Components)

All custom components share the same React and SDK instances from the host application. Checkout state is managed via the SDK hooks — `useCheckoutSession()` for reading, `useCheckoutAction()` for mutations. This is the primary mechanism for sharing state.

### Component-Local State

Use standard React `useState`/`useReducer` for UI-local state (form inputs, toggles, animations). Keep local state minimal — prefer SDK state for anything that other components need to see.

### Cross-Component Communication

Each custom component runs as an **isolated bundle**. Components do NOT share user-defined modules with each other.

**Shared React Context across components does NOT work** — each component bundle creates its own context object. A Provider in ComponentA will never reach a Consumer in ComponentB. There is no "wrapper slot" where a shared Provider can be mounted above multiple slots.

**Patterns for sharing state between components (in order of preference):**

1. **SDK checkout hooks** — when the shared state fits the checkout session model. All components read the same session via `useCheckoutSession()` and mutate it via `useCheckoutAction()`.

2. **Custom DOM events** — for signaling between components when the state does not belong in the checkout session:

```tsx
// Producer component
window.dispatchEvent(
  new CustomEvent("my-store:card-selected", {
    detail: { cardId, last4 },
  }),
);

// Consumer component
useEffect(() => {
  const handler = (e: CustomEvent) => setCard(e.detail);
  window.addEventListener("my-store:card-selected", handler);
  return () => window.removeEventListener("my-store:card-selected", handler);
}, []);
```

3. **Window globals with event notification** — for complex shared state that needs to persist beyond a single event.

### Window-Level Session API

External scripts (non-React) can subscribe to session changes:

```js
window.addEventListener("checkoutSessionUpdated", () => {
  const session = window.__CHECKOUT_SESSION__;
  const raw = window.__RAW_CHECKOUT_SESSION__;
});
```

These globals are updated after every successful mutation action.

---

## Styling & Design Tokens

CSS Modules are supported (`import styles from './index.module.css'`). Runtime theme overrides are exposed via `useStoreInfo().theme`.

For the canonical design-token list, interaction-state rules, responsive rules, accessibility requirements, copywriting tone, and the canonical CSS Module example, see **`references/design-contract.md`**. Do not duplicate those rules here.

---

## Platform Abstraction

The SDK provides an **anti-corruption layer over checkout session state** — cart, customer, shipping, payment, totals normalize across VTEX, Shopify, VNDA, and custom backends. Reads go through `useCheckoutSession()`; mutations through `useCheckoutAction()` action names (see §SDK Hooks).

Anything **outside** the checkout session (product catalog, category/brand lookups, stock, search, reviews, loyalty) is NOT abstracted — `useCheckoutAction("REQUEST")` is a pure HTTP proxy that forwards your call server-side and returns the platform's response unchanged. The full guidance on catalog lookups (name→id pitfall, three practical resolution routes) is above in §Slot context props vs admin props.

### Session Extensions

The `CheckoutSession` accepts a generic `Extensions` parameter for platform-injected data:

```ts
type CheckoutSession<E extends Record<string, unknown>> = {
  // ... standard fields ...
  extensions?: E;
};

// Usage in a component:
interface MyExtensions {
  loyaltyData: LoyaltyData;
}
const { session } = useCheckoutSession<MyExtensions>();
// session.extensions?.loyaltyData is now typed
```

Extensions are typically populated by Hub Functions that enrich the session server-side. See `references/hub-functions.md`.

**Template contract — payment-method badges.** The default template reads a discount pill on the payment selector from `extensions.badges.payment_method_options[<method name>]` (keyed by the method's display name, value rendered as HTML). A response function stamps `orderForm.extensions.badges.payment_method_options` to light it up — the plain field on `paymentData.paymentSystems[]` would be dropped by the platform mapper. Full details + snippet in `assets/functions/payment-restriction/INSTRUCTIONS.md` → "Tagging a payment method".

### When to Use External APIs

Prefer using a Function to enrich the session server-side, then read the data via `useCheckoutSession()`. Only call external APIs directly from a component when the scope is purely UI or restricted to a single slot/component.

---

## Error Handling

Three layers to know about:

- **Action errors** — every `useCheckoutAction` call exposes structured `error.serverError` (network / platform rejection) and `error.validationErrors` (Zod input failures) via the `onError` callback. Session is not updated when either fires. Full signature in §SDK Hooks / `useCheckoutAction`.
- **Fallback state** — when the platform API fails, `useCheckoutSession()` may serve a stale cache. Read `isFallback` + `fallbackError` and surface a warning; call `revalidate()` / `revalidateAsync()` to force fresh. Fields listed in §SDK Hooks / `useCheckoutSession`.
- **Slot error boundaries** — every `<Slot>` is wrapped in an `ErrorBoundary`, so a crash in your component never takes down the parent tree (Studio shows a debug UI; production shows a generic fallback). Own your in-component error handling anyway — the boundary is a last resort.
- **Messages system** — `useMessages().addMessage({ type, content })` surfaces user-facing errors. Full signature in §SDK Hooks / `useMessages`.

---


## Known-absent APIs

The SDK surface documented above is **exhaustive** for the current release. If a capability is not listed in §SDK Hooks, it does not exist. Do NOT stub fake helpers or invent namespaces to paper over a gap.

Two concrete pitfalls seen in the wild:

- **No imperative `sdk.cart.*` namespace.** Mutations go through `useCheckoutAction(...)` (e.g. `UPDATE_COUPONS`, `ADD_ITEMS`, `UPDATE_ITEMS_QUANTITY`). There is no `sdk.cart.applyCoupon()` / `sdk.cart.setMarketingTags()` / etc.
- **No shared React Context across components.** Each component is its own bundle; `React.createContext(...)` values do not cross bundle boundaries. Coordinate state via SDK actions, not context.

Anything else you reach for that isn't in §SDK Hooks — telemetry, toasts, imperative navigation, a11y announcers, persistent storage helpers, etc. — also does not exist. If your component genuinely needs one, surface the gap to the skill maintainer instead of stubbing `sdk.foo()` in shipped code.

---

## Best Practices (SDK-specific)

The general authoring rules (loading/error states, design tokens, a11y, complexity limits, comments, file organization) live in **[`component-authoring-cheatsheet.md`](component-authoring-cheatsheet.md)**. What is genuinely SDK-specific and worth reinforcing here:

- **Never fetch platform data outside the SDK.** All checkout-session state comes from `useCheckoutSession()`; all mutations go through `useCheckoutAction()`. Data outside the session (catalog, stock, etc.) goes through `useCheckoutAction("REQUEST")` — never `fetch()` a platform URL directly from a component.
- **Never assume a specific template layout.** Design against the slot's contract (see the slot's `INSTRUCTIONS.md`), not the current template's chrome. Templates evolve.
- **Do not use slot ids outside the `ComponentSlot` enum.** Made-up ids fail silently at registration.
- **Do not share React Context across component bundles.** Each component is bundled independently — `React.createContext(...)` values do not cross bundles. Coordinate via SDK actions or the DOM-event pattern in §State Management.
- **Asset imports** — only `.ts` / `.tsx` / `.js` / `.jsx` / `.css` are supported by the bundler. Use inline SVG React components or data URIs for images; see `component-anatomy.md` §Shared assets for the canonical icon pattern.
