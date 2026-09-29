# Reject Function

## What it is

A **request** function that intercepts an outbound request on the hub and **responds directly** without proxying it to the upstream commerce platform. Useful for short-circuiting calls when a precondition fails — e.g. blocking an order finalization based on a custom rule, returning a maintenance message during a campaign window, or refusing requests from a banned customer.

## Trigger type

- **`request`** — runs before the hub forwards the request upstream. By returning a `Response` from the handler, you skip the upstream call entirely.

## Where the function lives

- Filesystem layout:
  ```
  ./functions/<name>/
  ├── index.ts          # exports `handler: CustomFunction`
  └── package.json
  ```
- Deploy via the admin uploader as a zip with `./index.ts` + `./package.json` at the root.

## Handler shape

```ts
import type { CustomFunction } from "@ollie-shop/functions";

export const handler: CustomFunction = async ({ req }) => {
  const url = new URL(req.url);
  if (shouldReject(url, req)) {
    return new Response(
      JSON.stringify({
        error: "Validation failed",
        details: "The quantity of this item is limited to 5.",
      }),
      {
        status: 422,
        headers: {
          "Content-Type": "application/json",
          // Picked up by the checkout action client to surface the message.
          "x-ollie-shop-custom-message": "Max quantity is 5",
        },
      },
    );
  }
  return req; // let the request go through unchanged
};
```

The contract:
- **Return a `Request`** → the hub replaces the outbound request with the one you return (useful when you want to *modify* before forwarding — that's a different pattern; see `external-validation`).
- **Return a `Response`** → the hub stops here and returns your response to the client. This is the "reject and answer" pattern.
- **Return the original `req`** → the request goes upstream unchanged.

## Rejection response — the contract the client expects

This is the part devs trip on. For the checkout's action client to surface your custom message to the user, the rejection `Response` must follow this exact shape:

- **Status: `422`** — this is the recommended status for a user-facing rejection. The client's action handler reads the message only for `422`. Any other status (4xx, 5xx) falls back to a generic error message.
- **Header: `x-ollie-shop-custom-message: <your message>`** — this is the string the client renders to the user. Required for the front to display your text.
- **Body:** JSON with `error` and `details` fields. The body is what your function's own consumers (other hub functions, logs) read; the user-facing text comes from the header.

If you return any other status or omit the header, the front will still show *something*, but it will be a generic "an error occurred" instead of your message.

## Behavior

1. Inspect `req.url`, `req.method`, headers, and (when needed) body. The body is a stream — clone first if you need to forward later.
2. Evaluate your business rule. Keep it cheap: this runs on the hot path.
3. If the rule fails: return a `Response` following the contract above (status `422` + `x-ollie-shop-custom-message` header).
4. If the rule passes: return `req` to let the request go through unchanged.

## States to cover

- **Rule passes** — return `req` so the upstream call proceeds.
- **Rule fails** — return a rejection `Response` with status `422` and the custom-message header.

## Trigger configuration

```json
{
  "url": "https://<store>.<platform-host>/api/<endpoint-to-guard>",
  "expression": "method in [\"POST\"]"
}
```

Narrow the `expression` aggressively. A reject function attached to GET traffic by mistake will break read paths.

## Specific questions to ask before coding

> The reject pattern is more dangerous than enrichment — it changes user-visible behavior.

1. **What's the exact trigger endpoint and method(s)?** Reject functions on the wrong endpoint cause silent breakage. Get the HAR or platform docs first.
2. **What's the user-facing message?** This goes in the `x-ollie-shop-custom-message` header. Confirm it's the final, customer-ready copy in the right language.

## Antipatterns

- **Don't read the request body more than once.** Clone first (`req.clone()`) before any read; the body is a stream.
- **Don't return a `Response` for read paths (GET).** You'll break the cart / shipping / payment screens. Use `expression: 'method in ["POST"]'` (or stricter) on the trigger.
- **Don't rely on a flaky external source inside a reject function.** If the external source is slow or down, you'll start rejecting on timeout. Cache the rule data or use a separate function for that lookup with its own retry/cache layer.
- **Don't return a non-`422` status with a custom message** and expect the front to render it. Any other status falls back to the generic error.
- **Don't put the user-facing copy only in the JSON body.** The client picks up the message from the `x-ollie-shop-custom-message` header, not from the body.
