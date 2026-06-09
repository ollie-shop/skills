# Ollie Shop SDK Guide

Reference for building custom checkout components using the `@ollie-shop/sdk`.

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

Every Ollie Shop component exports a default function matching the `CustomComponent` type from `@ollie-shop/sdk`.

### Component Signature

```tsx
import type { CustomComponent } from "@ollie-shop/sdk";
import { useCheckoutSession } from "@ollie-shop/sdk";

const MyComponent: CustomComponent<MyProps> = ({ props, children }) => {
  const { myProp = "default" } = props || {};
  const { session } = useCheckoutSession();

  // Option A — augment: keep the native default and add on top
  // return <>{children}<MyExtra /></>;

  // Option B — replace: ignore children and take over the slot
  // return <MyOwnUI />;
};

export default MyComponent;
```

The `CustomComponent<P>` generic type injects:

- `props` — admin-configured prop values (overrides code defaults)
- `children` — the slot's native default UI. Render `{children}` to keep it (augment) or omit it to replace. See `references/slots-reference.md` §Children: Replace vs Augment.

To read cart, customer, shipping, payment, totals, extensions, etc., always use the **`useCheckoutSession()`** hook (see below). Do **not** destructure `cart` from the function arguments — it is not provided that way.

### Slot context props vs admin props

The slot registry (`assets/checkout-slots-data.yaml`) declares `context_props` for some slots — runtime values the slot forwards to its custom component (e.g. `item` on `cart_item_addons`, `item`/`formatPrice`/`onRemoveItem` on `cart_item`).

**Context props arrive at the TOP LEVEL of the component signature, alongside `children` — NOT nested inside `props`.** The `props` key is reserved for admin-configured values declared in `meta.json`.

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

Typing tip: if your component has both context props and admin-configured props, destructure both levels:

```tsx
interface AdminProps {
  giftWrapSkuId?: string;
}
interface SlotProps {
  item?: CartItem;
  props?: AdminProps;
  children?: ReactNode;
}

export default function MyAddon({ item, props, children }: SlotProps) {
  const { giftWrapSkuId = "35127" } = props || {};
  // ...
}
```

**Known quirks to confirm live:** the canonical SDK type `CustomComponent<P>` shows only `({ props, children })`, but the runtime also forwards context props at top level. When in doubt, `console.log(arguments[0])` and inspect every key the slot gives you — the slot registry's `context_props_shape` field (when populated) is the authoritative reference for each slot.

**Display names vs platform IDs (pitfall):** context-prop fields like `item.category`, `item.brand`, `item.seller` arrive as **human-readable strings** (e.g. `"Shoes"`), NOT as the platform-native IDs (e.g. VTEX `categoryId = 123`). Substituting the name into a URL that expects an ID — `fq=C:{categoryId}` on VTEX Search, brand/seller filters, etc. — will silently fail or return wrong results.

**Where to get the ID.** The SDK does **not** own platform catalog data. `useCheckoutAction("REQUEST")` is a passthrough — it proxies your HTTP call to whatever absolute URL you give it and returns the raw response untouched. For anything outside the `CheckoutSession` (product catalog, categoryId lookups, brand metadata, stock), hit the **platform API directly** via `REQUEST` and read the **platform's own docs** for the endpoint + response shape. There is no SDK-level abstraction of those responses.

**Practical options:**

1. **Name→id map in `meta.json -> props`** — admin-configurable per merchant, avoids a live lookup. Best when the category set is small and stable.
2. **Live lookup via `REQUEST` to the platform's catalog API** — e.g. VTEX `/api/catalog_system/pub/category/tree/N` to resolve a name to an id. Adds a round-trip and you own the parsing.
3. **Fallback to the name-based filter the platform offers** — e.g. VTEX `ft=<name>` instead of `fq=C:{id}`. Cheapest, but matching is fuzzier.

### File Layout

> **CRITICAL:** The entry point `index.tsx` **MUST** be at the component root folder. Components are discovered via the glob `*/index.tsx` — placing the entry point inside a subdirectory (e.g. `src/index.tsx`) will break discovery, builder validation, and production builds.

```
components/
└── my-component/
    ├── index.tsx                    # Entry point — default exports the CustomComponent
    ├── components/                  # Presentational sub-components (optional)
    │   └── SubComponent.tsx
    ├── hooks/                       # Data/logic hooks (optional)
    │   └── useMyComponent.ts
    ├── utils/                       # Pure helpers (optional)
    ├── mocks/                       # Preview fixtures + single MOCK_ENABLED toggle
    │   ├── index.ts                 #   export const MOCK_ENABLED = true; ...
    │   └── fixture.json
    ├── types.ts                     # Shared interfaces (optional)
    ├── index.module.css             # Component styles
    └── meta.json                    # Component metadata + slot assignment

commons/                            # Import-only / shared modules — NO meta.json,
└── shared-thing/                   # NOT discovered as slot components. Imported by
    └── index.tsx                   # real components via e.g. "../../commons/shared-thing".
```

> **`meta.json` decides registration.** A folder under `components/` with a `meta.json` is registered to the slot it names. A module WITHOUT a `meta.json` is not a slot component — keep those in a top-level `commons/` folder (sibling of `components/`) so the `components/*/index.tsx` glob never picks them up, and import them from the real component that uses them. This is the fix when two components would otherwise compete for the same slot: render one inside the other and move the inner one to `commons/`.

> **One manifest per component: `meta.json`.** Do NOT emit a per-component `ollie.json` — there is no documented schema for one and the CLI does not consume it. `meta.json` is the only canonical artifact at the component level. (The root-level `ollie.json` exists at the project root and carries `storeId` / `platformStoreId`, but that is a different file with a different scope.)

### meta.json Schema

```json
{
  "name": "free-shipping-bar",
  "version": "1.0.0",
  "slot": "header",
  "description": "Shows progress toward free shipping threshold",
  "props": {
    "target": { "type": "number", "default": 99, "required": false }
  }
}
```

Constraints:

- `name`: lowercase alphanumeric + hyphens, no leading/trailing hyphens, max 50 chars
- `version`: semver format (`1.0.0`, `2.1.0-beta.1`)
- `props`: max 20 props per component, each typed as `string | number | boolean | object | array`
- `tags`: max 5 tags, each max 20 chars
- Reserved names: `index`, `checkout`

Props declared in `meta.json` can be overridden via the Admin without code changes.

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
  totals: { total, items, shipping, discount, interest }
  locale: { language, currency }
  campaign?: { coupons[], giftCards[] }
  extensions?: Record<string, unknown>
  readOnly: boolean
}
```

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
| `platformStoreId` | `string \| undefined`                  | Platform-native store identifier. On VTEX this is the **account name** (e.g. `"olliepartnerus"`) used to build `https://{platformStoreId}.vtexcommercestable.com.br/...` URLs. Sourced from `ollie.json` at the store root. Prefer this over exposing a `vtexAccount` prop on components — every platform-specific catalog call (best sellers, category tree, brand lookup, stock, etc.) should read it from here. |
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
  loginRef: React.RefObject<HTMLDialogElement | null>;
  openLogin: (options?: { isRequired?: boolean; title?: string }) => void;
  closeLogin: () => void;
  isLoginRequired: boolean;
  loginTitle: string | null;
};
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

CSS Modules are supported (`import styles from './styles.module.css'`). Runtime theme overrides are exposed via `useStoreInfo().theme`.

For the canonical design-token list, interaction-state rules, responsive rules, accessibility requirements, copywriting tone, and the canonical CSS Module example, see **`references/design-contract.md`**. Do not duplicate those rules here.

---

## Platform Abstraction

The SDK provides an **anti-corruption layer over checkout session state** — cart, customer, shipping, payment, totals — so components don't have to know whether the backend is VTEX, Shopify, etc.

**What is abstracted (read + mutate via SDK):**

- **Cart state** — `useCheckoutSession().session` provides a platform-agnostic `CheckoutSession` regardless of backend.
- **Mutations** — `useCheckoutAction()` translates typed actions (`UPDATE_COUPONS`, `ADD_ITEMS`, …) into platform-specific calls server-side.
- **Payment** — methods, installments, saved cards are normalized into a consistent schema.
- **Shipping** — addresses, packages, quotes follow a unified model.

**What is NOT abstracted (you hit the platform directly):**

- **Product catalog, categoryId/brandId lookups, stock, search, reviews, loyalty balances, etc.** — anything that isn't part of the checkout session. `useCheckoutAction("REQUEST", { url, … })` is a **pure HTTP proxy**: it forwards your call server-side (so you don't leak tokens to the browser) and returns the response **unchanged**. There is no normalization — read the platform's own docs to know the shape of what comes back, and do your own parsing. If you later switch platforms, you replace these calls yourself; the SDK won't cover for you.

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

Extensions are typically populated by Hub Functions that enrich the session server-side. See `references/functions-guide.md`.

### When to Use External APIs

Prefer using a Function to enrich the session server-side, then read the data via `useCheckoutSession()`. Only call external APIs directly from a component when the scope is purely UI or restricted to a single slot/component.

---

## Error Handling

### Action Errors

Every `useCheckoutAction` call provides structured error information:

```ts
const { execute, error } = useCheckoutAction("UPDATE_PAYMENT_METHODS", {
  onError: ({ serverError, validationErrors }) => {
    // serverError — structured error with `message` and optional `code`
    // validationErrors — Zod validation errors for the input
  },
});
```

- `error.serverError` — server-side error (network failure, platform rejection)
- `error.validationErrors` — client-side Zod validation failures on the input
- Session is **not** updated on error

### Fallback State

When the platform API fails, the session may be served from a stale cache:

```ts
const { session, isFallback, fallbackError, clearFallback } =
  useCheckoutSession();

if (isFallback) {
  // Show warning: data may be stale
  // fallbackError contains the error message
}
```

Call `revalidate()` or `revalidateAsync()` to force a fresh fetch and clear fallback state.

### Slot Error Boundaries

See `references/slots-reference.md` §Constraints — slot-level error isolation is documented there.

### Messages System

Use `useMessages()` to surface errors to the user:

```ts
const { addMessage } = useMessages();
addMessage({ type: "error", content: "Payment failed. Please try again." });
```

---

## Testing Components

<!-- TODO: Document testing setup -->
<!-- Topics: test utilities provided by SDK, mocking slot context -->

---

## Known-absent APIs

The SDK surface documented above is **exhaustive** for the current release. If a capability is not listed in §SDK Hooks, it does not exist. Do NOT stub fake helpers or invent namespaces to paper over a gap.

Two concrete pitfalls seen in the wild:

- **No imperative `sdk.cart.*` namespace.** Mutations go through `useCheckoutAction(...)` (e.g. `UPDATE_COUPONS`, `ADD_ITEMS`, `UPDATE_ITEMS_QUANTITY`). There is no `sdk.cart.applyCoupon()` / `sdk.cart.setMarketingTags()` / etc.
- **No shared React Context across components.** Each component is its own bundle; `React.createContext(...)` values do not cross bundle boundaries. Coordinate state via SDK actions, not context.

Anything else you reach for that isn't in §SDK Hooks — telemetry, toasts, imperative navigation, a11y announcers, persistent storage helpers, etc. — also does not exist. If your component genuinely needs one, surface the gap to the skill maintainer instead of stubbing `sdk.foo()` in shipped code.

---

## Best Practices

### Do

- Use SDK hooks for all data access — never fetch directly from platform APIs
- Handle loading and error states in every component
- Use design tokens for styling consistency
- Keep components focused on a single responsibility
- Test with multiple cart states (empty, single item, many items, edge cases)
- Keep the component's files within its own folder tree (subfolders like `components/`, `hooks/` are fine)

### Don't

- Don't hard-code platform-specific logic (e.g., VTEX API calls)
- Don't store sensitive data in component state
- Don't rely on global DOM manipulation — work within the slot boundary
- Don't assume a specific template layout — use slot constraints
- Don't skip accessibility (a11y) — checkout must be accessible
- Don't import files from *another* component's folder — keep imports within this component's own tree (subfolders are fine)
- Don't import `.svg`, `.png`, or `.jpg` files — use inline SVG React components or data URIs instead
- Don't share React Context across component bundles — it won't work (each bundle gets its own context object)
- Don't use slot values not in the ComponentSlot enum
- Don't place payment-tagged components on non-payment slots (`payment-method` or `checkout-summary` only)

### Asset Constraints

The build pipeline only supports `.ts`, `.tsx`, `.js`, `.jsx`, and `.css` files.

```tsx
// INVALID — image imports fail at build
import visaLogo from "../../assets/visa.svg";

// VALID — inline SVG component
export function VisaIcon(props: React.SVGProps<SVGSVGElement>) {
  return (
    <svg viewBox="0 0 48 32" {...props}>
      <path d="..." />
    </svg>
  );
}

// VALID — data URI for small assets
export const logo = `data:image/svg+xml,${encodeURIComponent("<svg>...</svg>")}`;
```
