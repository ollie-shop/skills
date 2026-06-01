# External Validation

## What it is

A function that calls an **external API** to validate something about the current request or response, and acts on the result — either letting the flow continue, modifying the outgoing request, or rejecting with a custom response. Common use cases: customer eligibility checks, fraud screening, B2B credit limit checks, age verification, document checks.

## Trigger type

- **`request`** — when the validation must happen before the platform processes the call (e.g. block adding restricted items, attach signed auth headers to a backend call).
- **`response`** — when the validation reads the platform's response and acts on it (e.g. cross-check a created order against a compliance API).

Pick based on whether the validation has to run before or after the platform call.

## Where the function lives

```
./functions/<name>/
├── index.ts          # exports `handler: CustomFunction`
└── package.json
```

Deploy via the admin uploader as a zip with `./index.ts` + `./package.json` at the root.

## Handler shape — request flavor

```ts
import type { CustomFunction } from "@ollie-shop/functions";

export const handler: CustomFunction = async ({ req }) => {
  const url = new URL(req.url);
  const customerId = extractCustomerId(req);

  const verdict = await fetch("https://validator.example.com/check", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.VALIDATOR_TOKEN}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ customerId, url: url.pathname }),
  }).then((r) => r.json());

  if (!verdict.ok) {
    return new Response(
      JSON.stringify({ error: "Validation failed", details: verdict.reason }),
      {
        status: 422,
        headers: {
          "Content-Type": "application/json",
          // Picked up by the checkout action client to surface the message.
          "x-ollie-shop-custom-message": verdict.reason,
        },
      },
    );
  }

  // The validation passed — attach auth header for the upstream request
  return new Request(req.url, {
    method: req.method,
    headers: { ...Object.fromEntries(req.headers), "x-validated": "1" },
    body: req.method === "GET" || req.method === "HEAD" ? null : await req.text(),
  });
};
```

Three return contracts:
- **`Request`** — let the upstream call proceed, but with a modified `requestInit` (new headers, auth, signed body, etc.).
- **`Response`** — short-circuit and answer the client directly. See "Rejection response shape" below for the exact contract.
- **The original `req` (request flavor) or `res` (response flavor)** — proceed unchanged.

## Rejection response shape

When you return a `Response` to short-circuit, the checkout's action client picks up your message only if you follow this exact shape:

- **Status `422`.** Any other status (`400`, `403`, `409`, 5xx, etc.) makes the front fall back to a generic error and ignore your text.
- **Header `x-ollie-shop-custom-message: <your message>`.** This is what the user sees.
- **Body:** JSON with `error` and `details` fields. Useful for logs and downstream functions; the user-facing copy still comes from the header.

This matches the contract documented in `functions/reject-function/INSTRUCTIONS.md` — same hub, same client.

## Manipulating `requestInit`

The most under-appreciated capability of request functions: you can attach headers, cookies, or auth that the original client didn't include — for example, to authenticate against a downstream API that you proxy through the same hub.

```ts
return new Request(req.url, {
  method: req.method,
  headers: {
    ...Object.fromEntries(req.headers),
    "x-store-key": process.env.STORE_KEY!,
    "cookie": [req.headers.get("cookie"), `myToken=${token}`].filter(Boolean).join("; "),
  },
  body: req.method === "GET" || req.method === "HEAD" ? null : await req.text(),
});
```

Cookies, signed auth, request IDs, idempotency keys — all fair game.

## Behavior

1. Extract the inputs the external API needs (customer id, product ids, address, etc.) from `req` or `res`.
2. Call the external API. Use a short timeout (2-3s).
3. Decide what to do with the verdict. Possible outcomes:
   - Pass and proceed unchanged → return the original `req` (request flavor) or `res` (response flavor).
   - Pass but adjust the upstream call → return a modified `Request` (new headers, signed body, etc.).
   - Fail → return a `Response` following the "Rejection response shape" above (status `422` + `x-ollie-shop-custom-message` header).
   - External API errored or timed out → up to the function author; common choices are returning the original `req`/`res` unchanged (proceed as if the validator wasn't there) or returning a specific error `Response`. Pick one explicitly and document the choice in a business rule (`ollieshop business-rule update ...`).

## States to cover

Treat the validator's possible outcomes as a decision tree rather than a fixed prescription. At minimum, decide upfront how the function reacts to:

- **Validation passes** — proceed (with or without rewriting the request).
- **Validation fails with a clear reason** — short-circuit with a `Response` following the rejection shape above so the client renders your message instead of the generic fallback.
- **External API errors or times out** — author's call. The choice between fail-open, fail-closed, retry, or returning the original `req`/`res` unchanged shapes the customer experience and should be made explicit before coding.

## Trigger configuration

```json
{
  "url": "https://<store>.<platform-host>/api/<endpoint-to-validate>",
  "expression": "method in [\"POST\"]"
}
```

## Specific questions to ask before coding

> Cross-cutting integration questions (HAR, payload, auth scheme) live in the design flow — see `references/component-design-flow.md`. Ask the questions below only after the design flow.

1. **What's the source of truth for the verdict?** A REST endpoint? GraphQL? A signed webhook from the partner? The shape determines how the function calls it.
2. **Does the verdict need to be cached?** Repeated calls for the same input (same customer, same SKU) hammer the external API and add latency. A cache header read or short in-process TTL may be appropriate.

## Antipatterns

- **Don't put credentials in the function source.** Use `process.env.*`. Ship the secret separately to the Lambda config.
- **Don't call the external API on every request unconditionally.** Filter via the `expression` so the function only runs on the endpoints it's meant to validate.
- **Don't return a modified `Request` with a stale body.** If you `await req.text()` to read it, you must pass the text back; the original stream is consumed.
- **Don't assume the external API speaks the same currency / locale as the store.** Validate the units explicitly.
