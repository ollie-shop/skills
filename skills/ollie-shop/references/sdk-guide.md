# Ollie Shop SDK Guide

The SDK package is **`@ollie-shop/sdk`**. It exposes the React hooks a custom checkout component uses to read checkout state, dispatch actions, and react to UX events, plus the TypeScript types those hooks return.

Everything a custom component needs is available from the single `@ollie-shop/sdk` entry. Don't reach into `@ollie-shop/react` or `@libs/schemas` directly — those are implementation details that may change.

## Component contract

A custom component is a React function exported as `default` from `./components/<name>/index.tsx`. It takes its own props (configured via the component's `props` JSON in the database, surfaced by the slot) and renders into a template slot.

```tsx
// ./components/FreeShippingBar/index.tsx
import { useCheckoutSession } from "@ollie-shop/sdk";
import styles from "./styles.module.css";

export default function FreeShippingBar({ thresholdCents = 19900 }: { thresholdCents?: number }) {
  const { session } = useCheckoutSession();
  // session.totals.items is the items subtotal in minor units (cents).
  const remainingCents = Math.max(0, thresholdCents - session.totals.items);

  if (remainingCents === 0) {
    return <div className={styles.unlocked}>You unlocked free shipping!</div>;
  }

  const formatted = new Intl.NumberFormat(session.locale.language, {
    style: "currency",
    currency: session.locale.currency,
  }).format(remainingCents / 100);

  return (
    <div className={styles.bar}>Add {formatted} more for free shipping.</div>
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
  revalidate,     // () => void — re-fetch from the platform (rarely the right call — see below)
  revalidateAsync,// () => Promise<...>
  isFallback,     // true if session came from a cached fallback after an API failure
  fallbackError,  // string when isFallback
  clearFallback,  // () => void
} = useCheckoutSession();
```

The session is **generic** over extensions: `useCheckoutSession<MyExtensions>()` lets you read store-specific custom fields the platform attached. Default extensions type is an empty record.

Read state through `session.*` for the typed shape; only drop down to `rawSession` if you genuinely need a platform-specific field that's not on `CheckoutSession`.

**Do not call `revalidate()` after dispatching a checkout action.** `useCheckoutAction` already updates the session on success (except for `REQUEST`, `SIMULATE_SESSION`, and `CREATE_ORDER`, which return their own shape and don't mutate the session). Calling `revalidate()` triggers a redundant round-trip and can race with the action's own update. Reach for `revalidate()` only in the rare case where state could have drifted outside of any action you dispatched — e.g. after a background event you cannot observe — and even then, prefer dispatching a `REQUEST` action with `revalidate: true` so the refetch is sequenced with the rest of the action queue.

#### Authoritative shape of `session`

The schema below is what the SDK actually returns. Use these field names exactly; the d.ts in `@ollie-shop/sdk` is the source of truth if you need a field not listed here.

**`session.totals` — all amounts in minor units (cents).**

| Field | Type | Notes |
|---|---|---|
| `items` | `number` | Items subtotal (no shipping, no discounts). This is the field most "free shipping threshold" / "minimum order" rules want. |
| `total` | `number` | Grand total (items + shipping + tax + interest − discounts). |
| `shipping` | `number?` | Total shipping cost. |
| `tax` | `number?` | Total tax amount. |
| `discounts` | `number?` | Total discounts (positive number representing what was subtracted). |
| `giftCard` | `number?` | Amount paid with gift cards. |
| `interest` | `number?` | Interest from multi-installments. |
| `change` | `number?` | Change amount. |

**`session.locale`.**

| Field | Type | Example |
|---|---|---|
| `currency` | `string` | ISO 4217 — `"BRL"`, `"USD"`. |
| `language` | `string` | `"pt-BR"`, `"en"`, `"es"`. |
| `country` | `string` | ISO 3166-1 alpha-3 — `"BRA"`, `"USA"`. |

**`session.cartItems[]` — each amount in minor units (cents).**

| Field | Type | Notes |
|---|---|---|
| `id` | `string` | SKU code or platform item id. |
| `uniqueId` | `string` | Unique line id (platform-specific). |
| `index` | `number` | Position in the cart. |
| `name` | `string` | Display name. |
| `price` | `number` | Current sale price in cents. |
| `originalPrice` | `number` | Pre-sale price in cents. Use this as the base for promotional preview math; `price` already reflects applied discounts. |
| `quantity` | `number` | |
| `available` | `boolean` | |
| `image` | `string` | URL. |
| `url` | `string?` | PDP link. |
| `sellerId`, `variant`, `brand`, `category`, `variantDetails` | optional | Display extras when present. |

#### Typed extras (`session.extensions`) vs raw platform extras (`rawSession`)

Two different channels carry "extra" data on top of the parsed session — they look similar but are not the same:

- **`session.extensions`** is the **typed** channel. Reach for it via `useCheckoutSession<MyExtensions>()` and define `MyExtensions` as a `Record<string, unknown>` of fields you control. Use it when your own code (hub function, store config) attaches data to the session in a known shape.
- **`rawSession`** is the **unparsed** payload from the underlying commerce platform (VTEX orderForm, Shopify cart, etc.). It carries every platform-specific field — including fields a paired hub function may write under custom paths. Its shape varies per platform; treat any access as platform-coupled.

Default rule: prefer `session.*` (parsed, platform-agnostic) for everything that's there. Drop to `session.extensions` for your own typed extras. Drop to `rawSession` only for platform-specific data the parsed session doesn't expose. Don't reach into `rawSession` for fields that already live on `session`.

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
| `REQUEST` | `{ url, method?, headers?, body?, revalidate? }` | Escape hatch for arbitrary HTTP through the checkout backend. See the dedicated `REQUEST` subsection below for URL host, body, response envelope, and `revalidate` rules. |

`isPending` and the `usePendingActions` hook below both let you reflect the action's loading state.

#### `REQUEST` — calling external endpoints

`REQUEST` is the escape hatch for any HTTP call that isn't covered by a typed checkout action — a store-specific endpoint, a third-party validator, persisting custom data alongside the orderForm, etc. The hub forwards the call server-side, so it carries the checkout session and platform auth without the bundle ever touching them directly.

```ts
const { platformStoreId } = useStoreInfo();
const { executeAsync, isPending } = useCheckoutAction("REQUEST");

await executeAsync({
  url: `https://${platformStoreId}.vtexcommercestable.com.br/api/checkout/pub/orderForm/${id}/customData/recipe`,
  method: "PUT",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ item: JSON.stringify(recipes) }),
  revalidate: true,
});
```

Six details trip people up:

1. **URL must be absolute.** A relative path (e.g. `/api/checkout/...`) does not resolve against the store — the hub is a separate origin and has no implicit host. Always build the full `https://...`.

2. **VTEX-native routes go to `vtexcommercestable.com.br`, not `myvtex.com`.** Anything that starts with `/api/` (checkout-pub, master-data, orders, catalog) targets `https://{platformStoreId}.vtexcommercestable.com.br`. That host is the fastest path and the same one the rest of the checkout uses — hub functions matched to the same URL will fire. Routes that start with `/_v/` (VTEX IO custom services) live on the store's CNAME or `*.myvtex.com` instead. Read `platformStoreId` from `useStoreInfo()`:

   ```ts
   const { platformStoreId } = useStoreInfo();
   const VTEX_API = `https://${platformStoreId}.vtexcommercestable.com.br`;
   await executeAsync({ url: `${VTEX_API}/api/checkout/pub/orderForm/${id}/items`, ... });
   ```

3. **`body` must be a string** — typically `JSON.stringify(...)`. The hub does **not** serialize objects automatically. Passing `body: { foo: "bar" }` as a plain object results in an empty request payload (the field arrives `undefined` on the upstream); setting `Content-Type: application/json` in `headers` does not change that. Always stringify before passing. For multipart payloads (binary file uploads, etc.) `REQUEST` cannot help — use `fetch` directly with `credentials: "include"` when the bundle runs on the same origin as the store:

   ```ts
   const fd = new FormData();
   fd.append("file", file, file.name);
   const res = await fetch(`${VTEX_API}/_v/services/uploadImage`, {
     method: "POST",
     body: fd,
     credentials: "include",
   });
   const { data: imageUrl } = await res.json();
   ```

4. **Response envelope is axios-like.** The upstream body comes back inside `{ data, status, headers }`. In some hub paths it is nested one level deeper (`res.data.data`). A defensive extractor keeps consumers tidy:

   ```ts
   function extractResponseBody<T>(res: unknown): T | undefined {
     if (!res || typeof res !== "object") return undefined;
     const obj = res as Record<string, unknown>;
     if (obj.data && typeof obj.data === "object") {
       const inner = (obj.data as Record<string, unknown>).data;
       if (inner !== undefined) return inner as T;
       return obj.data as T;
     }
     return res as T;
   }

   const res = await executeAsync({ url, method: "GET" });
   const body = extractResponseBody<MyShape>(res);
   ```

5. **`revalidate: true` only on the LAST request of a multi-step flow.** Each `revalidate` re-pulls the session from the platform; chaining them means every step blocks on a round-trip and intermediate state can race with the next request. Set `revalidate: false` (the default) on intermediate steps, and `revalidate: true` only on the final mutation that the UI needs to reflect.

6. **Chained `REQUEST`s use a single hook with sequential `await`s.** When the flow needs more than one HTTP call in order (discover → authorize → apply), instantiate **one** `useCheckoutAction("REQUEST")` at the top of the component and call `await executeAsync(...)` in sequence inside the same async handler. Do not create a separate `useCheckoutAction("REQUEST")` per call and stitch them together via `onSuccess` callbacks — the second call may never fire (callbacks aren't a substitute for await), and you lose the ability to read the previous response inline.

   ```ts
   const { platformStoreId } = useStoreInfo();
   const { addMessage } = useMessages();
   const { executeAsync, isPending } = useCheckoutAction("REQUEST");

   async function applyDiscount(cpf: string, ean: string) {
     try {
       // Step 1 — discover. No revalidate yet (intermediate).
       const authRes = await executeAsync({
         url: "https://www.mystore.com.br/_v/app/authorize",
         method: "POST",
         headers: { "Content-Type": "application/json" },
         body: JSON.stringify({ cpf, ean }),
       });
       const auth = extractResponseBody<{ approved: boolean }>(authRes);
       if (!auth?.approved) {
         addMessage({ type: "error", content: "Not authorized." });
         return;
       }

       // Step 2 — apply. revalidate only on the last blocking step.
       await executeAsync({
         url: "https://www.mystore.com.br/_v/app/apply",
         method: "POST",
         headers: { "Content-Type": "application/json" },
         body: JSON.stringify([{ ean, document: cpf }]),
         revalidate: true,
       });

       addMessage({ type: "success", content: "Discount applied." });
     } catch (err) {
       addMessage({ type: "error", content: "Could not apply discount." });
     }
   }
   ```

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

#### Pairing `useMessages` with `useCheckoutAction`

Any async action that hits the network should pair a loading state with a `useMessages` push on both success and failure. This is the default pattern; it should not be optional.

```ts
const { platformStoreId } = useStoreInfo();
const { addMessage } = useMessages();
const { executeAsync, isPending } = useCheckoutAction("REQUEST");

async function applyCoupon(code: string) {
  try {
    await executeAsync({
      url: `https://${platformStoreId}.vtexcommercestable.com.br/api/checkout/pub/orderForm/${id}/coupon`,
      method: "POST",
      body: JSON.stringify({ text: code }),
      revalidate: true,
    });
    addMessage({ type: "success", content: "Coupon applied." });
  } catch (err) {
    const detail = err instanceof Error ? err.message : "";
    addMessage({
      type: "error",
      content: detail ? `Could not apply coupon. ${detail}` : "Could not apply coupon.",
    });
  }
}

// In the JSX:
<button onClick={() => applyCoupon(value)} disabled={isPending}>
  {isPending ? <Spinner /> : "Apply"}
</button>
```

Three things make this the default:

- `isPending` from `useCheckoutAction` (or a manual `useState` for non-action flows like a direct `fetch`) drives the button's disabled state and a spinner / skeleton. The user sees that something is happening.
- On success, `addMessage` confirms the action so the user is not left wondering if it worked.
- On error, `addMessage` surfaces the failure instead of leaving the UI silently broken. Logging to `console` does not count — the customer cannot see the console.

Without this pattern, the UI either looks frozen (no loading) or lies (silent failure). Both are real bugs reported in production pilots.

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
