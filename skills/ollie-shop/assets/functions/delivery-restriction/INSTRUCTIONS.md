# Delivery Restriction

## What it is

A **response** function that gates per-item delivery options based on a customer-side condition — a missing attachment, an unverified document, an unmet profile attribute, an absent identifier. For each item where the condition fails, removes the disallowed delivery SLAs, refunds any cost those SLAs were already contributing to the totals, and writes per-item snapshot flags so the UI can distinguish "blocked by this rule" from "never deliverable for other reasons".

Pairs with the `package-unavailable` component (`assets/components/package-unavailable/INSTRUCTIONS.md`), which reads the snapshot flags and renders the gated UI. The function and the component talk to each other only through these flags — keep their names stable across deployments.

> **Platform note**: the handler example and the field paths in this doc (`items[]`, `shippingData.logisticsInfo[]`, `slas[].deliveryChannel`, `value`, `totalizers[id="Shipping"]`) are written against the VTEX orderForm shape, which is what Ollie's response functions currently see for VTEX-backed stores. The pattern — gate per item, snapshot before/after, refund consistently across total and shipping totalizer — is platform-agnostic, but on Shopify, VNDA, or a custom backend the field names and the navigation paths change. Adapt the predicates and the mutation paths to the target platform's session schema; the snapshot contract and the antipatterns stay valid.

## Trigger type

- **`response`** — runs on the checkout-session response and rewrites the delivery options before the client renders the cart.

## Where the function lives

```
./functions/<name>/
├── index.ts          # exports `handler: CustomFunction`
└── package.json
```

Deploy via the admin uploader.

## Handler shape

```ts
import type { CustomFunction } from "@ollie-shop/functions";

export const handler: CustomFunction = async ({ res }) => {
  let session: SessionShape;
  try {
    session = await res.clone().json();
  } catch {
    return res;
  }

  const items = session.items ?? [];
  const logisticsInfo = session.shippingData?.logisticsInfo ?? [];
  let refundCents = 0;

  for (const info of logisticsInfo) {
    const item = items[info.itemIndex];
    if (!item) continue;

    // Snapshot the "before" state BEFORE any mutation.
    const hadDeliveryAvailable = (info.slas ?? []).some(isDeliveryChannel);

    if (conditionFails(item)) {
      const previouslySelected = (info.slas ?? []).find((s) => s.id === info.selectedSla);
      const wasDeliverySelected =
        info.selectedSla != null && isDeliveryChannel(previouslySelected);

      info.slas = (info.slas ?? []).filter(isAllowedWhenBlocked); // e.g. pickup only

      if (wasDeliverySelected) {
        refundCents += previouslySelected?.price ?? 0;
        info.selectedSla = null;
        info.selectedDeliveryChannel = null;
      }
    }

    info.hadDeliveryAvailable = hadDeliveryAvailable;
    info.hasDeliveryAvailable = (info.slas ?? []).some(isDeliveryChannel);
  }

  if (refundCents > 0) refundShipping(session, refundCents);

  return new Response(JSON.stringify(session), {
    status: res.status,
    headers: res.headers,
  });
};
```

`conditionFails`, `isDeliveryChannel`, `isAllowedWhenBlocked`, and `refundShipping` are predicates and helpers you implement per store. Their meaning is captured in the "Specific questions" section below.

## Behavior

1. Parse `res.clone().json()`. If the body is not the session payload, return the original `res`.
2. For each entry in `shippingData.logisticsInfo`:
   - **Snapshot before**: record whether the item had at least one allowed delivery SLA in the input. This is the `hadDeliveryAvailable` flag.
   - Evaluate the condition for the item.
   - If the condition fails:
     - Capture the SLA the customer had previously selected (if any) **before** filtering — its price is the refund if it was a delivery SLA.
     - Filter `info.slas` down to the allowed-when-blocked options.
     - If the previously-selected SLA was just removed and was a delivery SLA, null `info.selectedSla` and `info.selectedDeliveryChannel`, and add its price to the refund pool.
   - **Snapshot after**: record whether the item still has at least one allowed delivery SLA. This is the `hasDeliveryAvailable` flag.
3. If the refund pool is positive:
   - Subtract from `session.value`.
   - Subtract from the shipping totalizer (and any nested subtotal that represents the delivery component).
   - Drop the shipping totalizer entry if its value ends at or below zero — inspect the **final value**, not the refund amount.
4. Return a new `Response` carrying the mutated session, preserving the original `status` and `headers`.

## The snapshot contract (the key piece)

The function writes **two per-item boolean flags** on each `logisticsInfo` entry:

| Flag | When it's true | Why the UI needs it |
|---|---|---|
| `hadDeliveryAvailable` | The item had at least one allowed delivery SLA in the input. | Distinguishes "blocked by this rule" from "never deliverable" (no carrier coverage, no stock, etc.). |
| `hasDeliveryAvailable` | The item still has at least one allowed delivery SLA after this function ran. | Tells the UI whether delivery is currently possible for this item. |

The UI's predicate `hadDeliveryAvailable && !hasDeliveryAvailable` identifies the items this rule blocked — and only those. Items where `!hadDeliveryAvailable` are truly undeliverable and belong to the default platform render.

Pick the flag names once and document them. The paired component reads exactly these names; renaming on one side without the other is the most common drift bug.

## States to cover

- **No items match the rule** — set the flags (they may both be `true` everywhere) and return the session without any SLA changes.
- **Items match, condition satisfied for all** — set the flags, no SLA changes, no refund.
- **Items match, condition fails for one or more** — strip allowed SLAs, refund any selected delivery SLA, set the flags.
- **Item had no delivery SLA to begin with** — set `hadDeliveryAvailable = false` and `hasDeliveryAvailable = false`, skip the SLA filter (nothing to remove).
- **Parse failure / non-session body** — return the original `res`.

## Trigger configuration

For VTEX-backed stores, point at the orderForm host with a wildcard path that matches every sub-endpoint, and include every mutating method so the rule re-runs after the cart changes shape:

```json
{
  "url": "https://{{account}}.vtexcommercestable.com.br/api/checkout/pub/orderForm/*url",
  "expression": "method in [\"GET\", \"POST\", \"PUT\", \"DELETE\", \"PATCH\"]"
}
```

The `{{account}}` placeholder is the store's `platformStoreId` (see the SDK guide's `useStoreInfo` notes). The host follows the same canonical pattern documented for client-side `REQUEST` calls — `vtexcommercestable.com.br` is the fastest path and the same one the rest of the checkout uses, so other hub functions matched to the same URL fire too.

The expanded method list matters because the rule must re-evaluate on **every** orderForm-shape mutation, not just reads. Concrete example: the customer removes a blocked item via `DELETE /api/checkout/pub/orderForm/{id}/items/{index}`. The function has to run on the response of that DELETE so the freshly-shaped session keeps the right delivery options for the items that remain. Restricting the trigger to `GET` + `POST` means the customer's removal silently restores delivery for the others on the next render — a regression that is easy to miss in QA.

For non-VTEX backends, replace the URL with the equivalent session-endpoint family on the target platform and keep the broad method set.

## Specific questions to ask before coding

1. **What's the condition?** Where does the signal live — on the item (an attachment, a flag in customData), on the customer (a verified document, a profile attribute), or in a session-level enrichment field? Identify the canonical source before writing the predicate.
2. **What does "allowed when blocked" mean?** Usually pickup, but could be a specific carrier (e.g. "only same-day express"), a click-and-collect channel, or even "no delivery at all". This shapes the filter.

## Antipatterns

- **Don't snapshot the "before" flag after you filter.** Take it before any mutation. Reading the slas after filtering makes every blocked item look like it never had delivery, which collapses the UI's predicates.
- **Don't refund from `value` only or the totalizer only.** Refund both, always — touching one without the other leaves the UI inconsistent (the cart shows R$ 0 shipping but the total still includes it, or vice versa). This is not a choice the function author makes per store; it is a single correctness rule.
- **Don't keep the shipping totalizer at exactly zero.** Drop the entry. Inspect the **final value** after subtraction, not the refund amount — the entry may have residual cost from other items that weren't blocked.
- **Don't filter the SLA list without also clearing a now-invalid `selectedSla` pointer.** A selected id that no longer exists in `slas` confuses every downstream consumer.
- **Don't return a mutated response when the parse fails.** If `res.clone().json()` throws or the body is not the session, return the original `res` unchanged so the cart still renders.
- **Don't write the snapshot flags only on blocked items.** Set both flags on every item the function loops over, even the ones the condition didn't block. The UI then reads the flags as a contract: "flag present → the function ran on this item; flag absent → the function never reached this item". If you only write flags on blocked items, the UI cannot tell apart "function ran and didn't block" from "function never inspected this item" — both look like `undefined && undefined`. Always-writing keeps the contract observable and makes debugging trivial.
- **Don't bury the condition predicate inside the SLA loop.** Keep `conditionFails(item)` as a standalone, pure function — that's the part the next maintainer will want to read first.
