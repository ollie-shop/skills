# Hub Functions — Reference & Authoring Guide

A **hub function** is a server-side request/response interceptor (a Lambda) that sits in the proxy path between the storefront and the underlying platform. Use one to enforce business rules, enrich session data, rewrite a platform response, attach auth, or short-circuit a call — instead of patching behavior in the browser.

> Components (UI in slots) and Functions (server-side interceptors) are separate artifacts with separate layouts, build pipelines, and deploys. For components see `sdk-guide.md`.

---

## Invocation types

Every function picks one:

- **`request`** — runs **before** the hub forwards the call upstream. The `res` parameter is `null` at this point. Useful when the function needs to decide whether the call should proceed, rewrite the outgoing payload, attach auth, or short-circuit with a custom response.
- **`response`** — runs **after** the upstream returns, **before** the client receives the response. The `res` parameter is the upstream `Response`. Useful for enriching the payload, redacting fields, or rejecting based on what came back.

A single function picks one invocation type. If you need both (e.g. rewrite the request *and* mutate the response), write two separate functions.

---

## File layout

```
functions/<function-name>/
├── index.ts          # exports `handler: CustomFunction` — the build entry
├── index.test.ts     # optional, vitest
└── package.json      # see exact shape below
```

`<function-name>` is lowercase-kebab (`cart-enrichment`).

---

## package.json — get this exactly right

The build bundles `./index.ts` directly with **bun** (`bun build ... --target=node --format=cjs`). The package is **ESM source bundled in place**, NOT a tsc-compiled npm library. A `tsc`/`dist` style manifest **breaks the build**.

✅ **Correct shape:**

```json
{
  "name": "my-function",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./index.ts",
  "dependencies": {
    "node-cache": "^5.1.2"
  }
}
```

Rules:
- **`"type": "module"`** — functions are ESM.
- **`"main": "./index.ts"`** — points at the TypeScript **source**, not `dist/index.js`. bun bundles from source.
- **`dependencies`** are real npm packages — they get bundled into the output. Add only what `index.ts` imports at runtime.
- **Do NOT** add `@ollie-shop/functions` as a dependency — the builder injects it and marks it `--external`.
- **Do NOT** add `build`/`validate` scripts, a `types` field, or `event`/`timing`/`tags`/`priority` fields. Those came from an unrelated scaffold schema and do not drive the build.

---

## Handler signature

```ts
// `@ollie-shop/functions` is external — declare the type locally so the file type-checks standalone:
type CustomFunction = (context: {
  req: Request;
  res: Response | null;
}) => Promise<Request | Response | undefined>;

export const handler: CustomFunction = async ({ req, res }) => {
  // ...
  return undefined; // pass-through
};
```

The builder wraps it at deploy time:

```bash
import { createCustomFunctionHandler } from '@ollie-shop/functions';
import { handler as customFunction } from './index.ts';
export const handler = createCustomFunctionHandler(customFunction);

bun build _entry.ts --production --outfile=index.js \
  --target=node --format=cjs --external @ollie-shop/functions
zip output.zip index.js
```

---

## Runtime model

| `invocation` | `res` value | You return… |
|---|---|---|
| **request** (before upstream) | `null` | modified `Request` to rewrite what's sent upstream, a `Response` to short-circuit (skip upstream entirely), or `undefined` to pass through |
| **response** (after upstream) | upstream `Response` | modified `Response` to rewrite body/headers, or `undefined` to pass through |

- **Bodies are cached** — you may call `.json()` / `.text()` on `req`/`res` multiple times without "body already consumed". No need to `.clone()` before reading (but see [Reading without consuming](#reading-inputs-without-consuming-the-stream) if you need to forward the same body).
- **Narrow before acting.** Gate on the URL and phase so you only process what you intend.
- **When rewriting a response body, drop `content-length`** — the length changed:
  ```ts
  const headers = new Headers(res.headers);
  headers.delete("content-length");
  return new Response(JSON.stringify(body), { status: res.status, headers });
  ```
- **Session storage:** `globalThis.sessionStorage` is a real key→string store, hydrated from and serialized back to the checkout session each invocation. Use `getItem`/`setItem` to persist small values across the request/response pair or to the storefront.
- **Env:** `process.env` is populated from the function's configured env vars for the duration of the call.
- **Return `undefined` to pass through** — the default, fail-open behavior. Prefer it on unexpected errors so the checkout call is never broken by the function.

### Caching — use ONE request function, not two

Don't split a cache into a request reader + response writer — those are two separate containers and an in-memory `node-cache` is **not shared** between them (the reader never sees what the writer stored). Use ONE `invocation: "request"` function that does read-through:

```ts
const hit = cache.get(key);
if (hit) return toResponse(hit);          // HIT → short-circuits upstream
const upstream = await fetch(req.url, { method: req.method, headers });
if (upstream.ok) cache.set(key, ...);     // MISS → fetch, then store
return toResponse(upstream);
```

For a cache shared across instances, back it with an external store (Redis/ElastiCache) instead.

### Fail-open vs fail-closed

Decide deliberately and document it in the README:
- **fail-open** (`return undefined` on error) — safest for checkout availability; the platform's native response wins.
- **fail-closed** (return the restrictive outcome when eligibility is uncertain) — use only when showing something would be worse than a transient over-restriction.

---

## Rejection response contract

When returning a `Response` to block a call, the checkout's action client picks up your message **only** if the response follows this shape:

- **The actual trigger is the `x-ollie-shop-custom-message: <your message>` header** — this string is what the user sees. The action client surfaces it on the presence of the header alone; it does **not** check the status code (`error-handler.ts` keys off `error.response.headers.get("x-ollie-shop-custom-message")`).
- **Status: any error status (4xx/5xx).** The status is not inspected, but it **must** be non-2xx so the HTTP client throws an `HTTPError` — a 2xx response is treated as success and your message is ignored. **`422` is the recommended convention** (validation failure), but `400`/`409`/etc. work identically.
- **Body:** JSON with `error` and `details` fields (useful for logs; user-facing copy comes from the header).

```ts
return new Response(
  JSON.stringify({ error: "Validation failed", details: "Qty limited to 5." }),
  {
    status: 422,
    headers: {
      "Content-Type": "application/json",
      "x-ollie-shop-custom-message": "Max quantity is 5",
    },
  },
);
```

---

## Manipulating `requestInit`

By returning a new `Request`, you can attach headers, cookies, or signed payloads the original client didn't include:

```ts
return new Request(req.url, {
  method: req.method,
  headers: {
    ...Object.fromEntries(req.headers),
    "x-store-key": storeKey,
    "cookie": [req.headers.get("cookie"), `myToken=${token}`].filter(Boolean).join("; "),
  },
  body: req.method === "GET" || req.method === "HEAD"
    ? null
    : JSON.stringify(await req.json()),
});
```

Don't set `body` for `GET`/`HEAD` — those methods can't have a body.

---

## Priority and chaining

When multiple functions match the same call, the hub runs them in `priority` order (ascending):

- Each function receives the **previous function's output** as input.
- A function that returns a `Response` **stops the chain** — no later functions run in that direction.
- Errors are handled per each function's `on_error` setting independently.

Set priorities explicitly when order matters. Default `0` means unstable ordering when two functions share the same priority.

---

## Reading inputs without consuming the stream

`Request`/`Response` bodies are streams — they can only be read once. If you need to inspect the body and then forward it:

```ts
const body = await req.clone().json();
// ... use body ...
return new Request(req.url, { method: req.method, headers: req.headers, body: JSON.stringify(body) });
```

Use `req.clone()` to peek; calling `req.json()` directly consumes the stream and breaks later reads.

---

## Testing (vitest)

The handler is a plain function — test it directly, no platform needed:

```ts
const ctx = (body: unknown, url = ORDERFORM_URL) => ({
  req: { url } as Request,
  res: { status: 200, headers: new Headers(), json: async () => body } as unknown as Response,
});
const result = await handler(ctx(orderForm));
// undefined => passed through; Response => rewritten
```

Cover: the rewrite path, the pass-through path (eligible / not-applicable URL / target absent), and the fail behavior when config can't load.

---

## Trigger configuration

Each function record carries a `trigger` object set via the admin form or `ollieshop function create --trigger-url ... --trigger-expression ...`:

```json
{
  "url": "https://<store>.<platform-host>/api/checkout/pub/orderForm",
  "expression": "method in [\"POST\"]"
}
```

Plus fields on the function record:

| Field | Type | Purpose |
|---|---|---|
| `invocation` | `"request"` \| `"response"` | When the function runs in the proxy lifecycle. |
| `priority` | integer ≥ 0 | Execution order; lower runs first. Default `0`. |
| `on_error` | `"throw"` \| `"skip"` | What hub does if function throws. `throw` aborts; `skip` logs and continues. Default `throw`. |
| `active` | boolean | Whether hub considers this function for matching. Default `false`. |

### `trigger.url`

Absolute http(s) URL. Hub matches origin exactly + path via `path-to-regexp` (e.g. `/api/checkout/pub/orderForm/:id`). Query strings are ignored for matching.

### `trigger.expression`

A JSONata-like boolean expression evaluated against the in-flight request (and response, for response functions):

| Variable | Value |
|---|---|
| `method` | HTTP method uppercase (`"GET"`, `"POST"`, …) |
| `headers` | Object of request headers (lowercase keys) |
| `body` | Parsed JSON if `Content-Type: application/json`, else text |
| `status` | Upstream status code (response functions only; `undefined` on request triggers) |

Examples:
```
method in ["POST"]
method in ["POST", "PATCH"]
method in ["POST"] and headers."x-store" = "pharma"
status >= 200 and status < 300
method = "POST" and body.items[0].quantity > 5
```

Narrow aggressively — a function matched on `method in ["GET"]` fires on every read of that endpoint.

---

## Deploying

Recommended path: zip the function folder (`index.ts` + `package.json`), drop it on the function's create/edit form in the admin — the admin uploads and polls the build for you. See `references/cli-reference.md` for CLI options.

When a function replaces browser-side logic, keep the old client code until the function is **deployed and verified live**, then remove it — otherwise there's a window where the behavior is unguarded.
