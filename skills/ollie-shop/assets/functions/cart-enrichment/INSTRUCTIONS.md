# Cart Enrichment

## What it is

A **response** function that mutates the checkout session payload before the browser sees it, adding fields to cart items from an external source. Use it when items need extra data (custom labels, compliance flags, fulfillment info, content) that the commerce platform doesn't carry natively.

## Trigger type

- **`response`** — runs on the response of the checkout session endpoint, before the response reaches the client. Mutating the response body is the whole point of this function.

## Where the function lives

- Filesystem layout (when the deploy bundler is split per resource type):
  ```
  ./functions/<name>/
  ├── index.ts          # exports `handler: CustomFunction`
  └── package.json
  ```
- For now, deploy via the admin uploader as a zip with `./index.ts` + `./package.json` at the root.

## Handler shape

```ts
import type { CustomFunction } from "@ollie-shop/functions";

export const handler: CustomFunction = async ({ res }) => {
  const body = await res.clone().json();
  // ... fetch external data keyed by item SKUs
  // ... merge into body.items[].myCustomField
  return new Response(JSON.stringify(body), {
    status: res.status,
    headers: res.headers,
  });
};
```

Return the original `res` to let the response pass through unchanged. Return a new `Response` to replace the body the client receives.

## Behavior

1. Inspect `res.body`. If the response is not the checkout-session payload (wrong shape or non-2xx), return the original `res` to skip.
2. Collect the keys you need to look up (SKUs, item ids, customer email, etc.).
3. Call the external source. Use `fetch` directly.
4. Merge the external data into the response body. Keep the data inside item objects (e.g. `body.items[i].labels = […]`) so the consuming component can read it through `useCheckoutSession`'s extensions type.
5. Reconstruct a `Response` with the mutated body, preserving `status` and `headers`. Set `Content-Type: application/json` if you re-serialize.

## States to cover

- **Match present** — enrichment applies, body grows.
- **No matches** — return the original `res` to skip; never spend latency on a no-op.
- **External source fails** — fall back to returning the original `res`. Never throw on transient errors when `on_error: skip` is the trigger's setting.
- **External source slow** — short timeout (e.g. 2-3s) and a fallback. The function runs inline; if it stalls, the entire checkout response stalls.

## Trigger configuration

The trigger object stored in `functions.trigger`:

```json
{
  "url": "https://<store>.<platform-host>/api/checkout/pub/orderForm",
  "expression": "method in [\"POST\", \"GET\"]"
}
```

Refine the `expression` to narrow the trigger to specific endpoints if you only want to enrich after specific events.

## Without a clear contract from the external source

If the dev doesn't have an integration spec for the external source:

- Ask for a HAR file capturing a successful call to the external source on a working flow.
- Ask for a payload sample (request + response).
- Ask for the auth scheme (bearer token, signed header, mTLS, etc.).
- Ask whether the external source is rate-limited and what backoff it expects.

This question set is the cross-cutting "custom integration" path of the design flow; refer to `references/component-design-flow.md` and don't repeat it here.

## Antipatterns

- **Don't return a mutated body that breaks the schema the SDK parses.** Add fields under existing item objects (or a clearly-named top-level key); don't reshape `items[]` or rename existing fields.
- **Don't block the response on a slow external source without a timeout.** The checkout response delivers the cart UI — every second of stall is a perceived broken page.
- **Don't enrich every response.** Filter on `expression` so the function only runs on the endpoints it needs to mutate. Enriching unrelated traffic burns latency.
