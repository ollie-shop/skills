# Payment Restriction

## What it is

A **response** function that removes (or adds) payment methods from the checkout session based on the cart's contents, the customer, or any other business rule. Examples: only allow boleto when total is below a threshold; hide credit-card installments above a certain amount; allow promissory only for B2B customers; block PIX for a specific product category. The same response phase is also where you **tag** a method with a discount badge — see "Tagging a payment method" below.

## Trigger type

- **`response`** — runs on the checkout-session response and edits the list of allowed payment methods before the client renders the payment step.

## Where the function lives

```
./functions/<name>/
├── index.ts          # exports `handler: CustomFunction`
└── package.json
```

Deploy via the admin uploader as a zip with `./index.ts` + `./package.json` at the root.

## Handler shape

```ts
import type { CustomFunction } from "@ollie-shop/functions";

export const handler: CustomFunction = async ({ res }) => {
  const body = await res.clone().json();
  const items = body.items ?? [];
  const total = body.value ?? body.totalizers?.find((t: any) => t.id === "Total")?.value ?? 0;

  // Example: hide promissory if there's any item with the "restricted" category
  const hasRestricted = items.some((i: any) => i.productCategoryIds?.includes("/123/"));
  if (!hasRestricted) {
    return res;
  }

  body.paymentData.paymentSystems = body.paymentData.paymentSystems.filter(
    (p: any) => p.groupName !== "promissoryPaymentGroup",
  );

  return new Response(JSON.stringify(body), {
    status: res.status,
    headers: res.headers,
  });
};
```

Exact field names depend on the platform (`paymentSystems`, `groupName`, `name`, `id` for VTEX). Read the platform's schema to know what's available.

## Behavior

1. Parse `res.body` and validate it's the checkout-session payload (right shape, status 2xx).
2. Compute the decision inputs from `body`: items, customer, total, address, custom session extensions.
3. Run the rule. The rule should be **pure** — same inputs always produce the same restriction.
4. Mutate the platform-specific payment list (`body.paymentData.paymentSystems` on VTEX; equivalent fields on other platforms).
5. Return a new `Response` with the mutated body, preserving `status` and `headers`.
6. If the rule doesn't apply (no restrictions today), return the original `res` to skip — save the serialization cost.

## Hide payment methods (unconditional drop)

Sometimes the "restriction" is a fixed list — a duplicate/invalid method that should never render, independent of cart or customer. Keep the drop separate from conditional restriction so operations can flip the list per store without touching the rule engine.

- Drop entries by `id` or `stringId` from `body.paymentData.paymentSystems` **and** the matching entries in `body.paymentData.installmentOptions`.
- **Leave `body.paymentData.payments` alone** — that's the customer's current selection, not the catalog.
- Wire the hide step as a separate numbered enricher step **with its own `try/catch`** — fail-open so a bug in the hide list can't take down the entire payment step.
- Expose the list as an env var (convention: `HIDDEN_PAYMENT_SYSTEM_IDS`, comma-separated). Fall back to a compiled-in default if unset; the env var **replaces** the default when set (per-store override).

```ts
const hiddenIds = (process.env.HIDDEN_PAYMENT_SYSTEM_IDS ?? "125")
  .split(",")
  .map((s) => s.trim())
  .filter(Boolean);

try {
  body.paymentData.paymentSystems = body.paymentData.paymentSystems.filter(
    (p: any) => !hiddenIds.includes(String(p.id)) && !hiddenIds.includes(p.stringId),
  );
  body.paymentData.installmentOptions = (body.paymentData.installmentOptions ?? []).filter(
    (o: any) => !hiddenIds.includes(String(o.paymentSystem)),
  );
} catch (err) {
  console.error("[hide-payment-methods] fail-open", { errorMessage: (err as Error)?.message });
}
```

## Tagging a payment method (discount badge / description)

To surface a **flag on the payment-method selection** — e.g. a green "5% OFF" pill next to PIX, or "em até 8x sem juros" under a card — you do **not** add a field to the payment method object. The VTEX session mapper keeps only `{ id, type, name, installments }` from each `paymentSystem`, so any extra property you stamp on `paymentData.paymentSystems[]` is dropped before the front sees it.

The default template reads the badge from **session `extensions`**, keyed by the method's **name**:

```ts
// default template — Payment step (PaymentMethodOption.tsx)
badges: session.extensions?.badges?.payment_method_options?.[option.name]
```

- **Where:** `extensions.badges.payment_method_options` — an object keyed by the payment method's **display name** (`paymentSystem.name` on VTEX: `"Pix"`, `"PayPal"`, `"Cartão de crédito"`), NOT the id or the type.
- **Value:** a string rendered as **HTML** inside a success-colored pill (`"5% OFF"`, or richer markup). Compute it in the function if it's dynamic (e.g. the real PIX discount %).
- **How to set it (response function):** merge onto `body.extensions` — never clobber the rest of `extensions` (the step engine keeps `extensions.checkoutSteps` there):

```ts
const ext = (body.extensions ??= {});
const badges = (ext.badges ??= {});
badges.payment_method_options = {
  ...(badges.payment_method_options ?? {}),
  Pix: "5% OFF",
  PayPal: "5% OFF",
};
// return a new Response with the mutated body (same as the restriction flow)
```

Related option fields the same template maps: `description` (a plain-text subtitle, built by the template from the method data) and `tags` (a node on the right of the row). For full control over the badge markup/behavior, override the nested **`payment_selector_badge`** slot instead — it receives the payment `option` as a prop and replaces the default pill.

> On non-VTEX platforms the `extensions.badges.payment_method_options` convention is the same (it's a template contract, not a platform one); only the source you read the % from differs.

## States to cover

- **Rule applies, restrict** — payment list shrinks; previously-selected method may no longer be allowed (the platform will surface that error on the next action — let it).
- **Rule applies, allow** — return the original `res` unchanged.
- **Body shape mismatch** — return the original `res`. Don't try to mutate something you don't recognize.
- **Customer not identified yet** — most rules need the customer to be identified; gate on `body.clientProfileData` or equivalent and bail out early by returning the original `res`.

## Trigger configuration

```json
{
  "url": "https://<store>.<platform-host>/api/checkout/pub/orderForm",
  "expression": "method in [\"POST\", \"GET\"]"
}
```

## Specific questions to ask before coding

> The cross-cutting design-flow questions (custom integration, payload sample) cover the *what*. These are the *which* questions for this function.

1. **What's the rule's input?** Item category, item SKU, cart total, customer attribute, address state? The answer determines which fields you read.
2. **What's the platform's payment list field path?** On VTEX it's `paymentData.paymentSystems[].groupName`. On other platforms it'll be different.
3. **Is the restriction additive (hide methods) or substitutive (replace with a different list)?** Additive is the safe default; substitutive can break stores using methods not in your allowlist.
4. **Should the restriction apply on every session read or only after specific changes?** Decides the `expression` narrowing.

## Antipatterns

- **Don't mutate the response in place.** Clone, modify, and return a new `Response`. The runtime expects an immutable original.
- **Don't filter the payment list using string matching on user-visible names.** Use stable ids (`groupName`, `paymentSystem.id`). Names get translated.
- **Don't remove every method when the rule is uncertain.** Empty payment lists leave the customer stuck with no way to pay. If the rule data is unavailable, return the original `res`.
- **Don't drop the platform's existing `paymentData` fields you didn't intend to change.** Only touch `paymentSystems` (or whichever field you're restricting); preserve the rest.
- **Don't run the restriction on the order-create endpoint.** By that point the customer has picked a method; restricting at that stage rejects the order. Run on the session endpoint, before the payment step renders.
- **Don't call an external service synchronously inside this function.** Restriction rules should be local; if external data is required, cache it via a separate enrichment function.
