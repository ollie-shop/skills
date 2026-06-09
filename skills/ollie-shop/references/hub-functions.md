# Hub Functions

A **hub function** is a piece of code that runs on Ollie's hub as middleware on the request or response of a checkout endpoint. Functions let a store enforce business rules, enrich data, attach auth, or intercept calls without modifying the checkout UI or the underlying commerce platform.

This file is the agnostic reference for how to write, configure, and deploy any function. For opinionated patterns (cart enrichment, payment restriction, reject, external validation), see the entries in `assets/functions/`.

## Invocation types

Every function picks one:

- **`request`** — runs **before** the hub forwards the call upstream. Useful when the function needs to decide whether the call should proceed, rewrite the outgoing payload, attach auth, or short-circuit with a custom response.
- **`response`** — runs **after** the upstream returns, **before** the client receives the response. Useful for enriching the payload, redacting fields, or rejecting based on what came back.

A single function picks one invocation type per record. If a customization needs both (e.g. rewrite the request *and* mutate the response), write two functions.

## Filesystem layout

```
./functions/<name>/
├── index.ts          # exports `handler: CustomFunction`
└── package.json      # used by the builder to install runtime deps
```

The `index.ts` filename and the named export `handler` are required — the build step writes a thin wrapper that imports `./index.ts` and calls `createCustomFunctionHandler(handler)`. Other files (helpers, types, etc.) are fine alongside `index.ts`.

The same source layout is used regardless of invocation type — the difference between `request` and `response` is stored in the function record, not in the file structure.

## Handler signature

```ts
import type { CustomFunction } from "@ollie-shop/functions";

export const handler: CustomFunction = async ({ req, res }) => {
  // ... your logic
};
```

The type is:

```ts
type CustomFunction = (context: {
  req: Request;
  res: Response | null;
}) => Promise<Request | Response>;
```

### The context object

- `req` — the inbound `Request`. Always present, regardless of invocation type.
- `res` — the `Response` from the upstream call.
  - For an **`invocation: "response"`** function: `res` is the upstream response and is always present.
  - For an **`invocation: "request"`** function: `res` is `null` (the upstream hasn't been called yet).

The TS type widens `res` to `Response | null` for both flavors, but the schema discriminator guarantees the presence rule above at runtime. Inside a response function, write against `res` without defensive null guards; inside a request function, write against `req`.

### Return contracts

Pick exactly one of these per invocation:

| Return value | Effect |
|---|---|
| The **original** `req` (request flavor) or `res` (response flavor) | Pass-through. The upstream call (request flavor) or the client response (response flavor) is unaffected. |
| A new `Request` | The hub replaces the outbound request with the one you return. Only meaningful for request functions — use it to attach auth headers, sign the body, rewrite the URL, etc. |
| A new `Response` | The hub stops here and returns this `Response` to the client. Short-circuits the upstream call (request flavor) or replaces the upstream payload (response flavor). See "Rejection response contract" below for the shape the checkout client expects. |

Always return one of these three. There is no implicit "do nothing" — the handler must return.

## Trigger configuration

Each function record carries a `trigger` object in the database (set via the admin form or `ollieshop function create --trigger-url ... --trigger-expression ...`):

```json
{
  "url": "https://<store>.<platform-host>/api/checkout/pub/orderForm",
  "expression": "method in [\"POST\"]"
}
```

Plus, separately on the function record:

| Field | Type | Purpose |
|---|---|---|
| `invocation` | `"request"` \| `"response"` | When the function runs in the proxy lifecycle. |
| `priority` | integer ≥ 0 | Execution order among functions that match the same call. Lower priority runs first. Defaults to `0`. |
| `on_error` | `"throw"` \| `"skip"` | What the hub does if the function throws. `throw` aborts the proxy with the function's error; `skip` logs and continues as if the function returned the original `req`/`res`. Default `throw`. |
| `active` | boolean | Whether the hub considers this function for matching. Default `false`. |

### `trigger.url`

An absolute http(s) URL that identifies the endpoint the function attaches to. The hub matches against the request's origin and path; query string is ignored for matching. The URL must be reachable from the hub — relative paths (`/api/...`) are accepted by the CLI as a shortcut but resolve against the store's platform host at deploy time.

### `trigger.expression`

A JSONata-like boolean expression evaluated against the in-flight request (and the response, for response functions). When it returns `true`, the function runs.

Available variables inside the expression:

- `method` — HTTP method, uppercase (`"GET"`, `"POST"`, etc.).
- `headers` — object of request headers (lowercase keys).
- `body` — the request/response body. JSON bodies are pre-parsed when `Content-Type: application/json`; otherwise it's the text string.
- `status` — the upstream response status code. Only meaningful for response functions; `undefined` on request triggers.

Examples:

```
method in ["POST"]
method in ["POST", "PATCH"]
method in ["POST"] and headers."x-store" = "pharma"
status >= 200 and status < 300
method = "POST" and body.items[0].quantity > 5
```

Narrow the expression aggressively. A function matched against `method in ["GET"]` will fire on every read of the endpoint — that's almost never what you want for mutations.

## Rejection response contract

When a function returns a `Response` to short-circuit (block the call, refuse with a message, enforce a precondition), the checkout's action client picks up your message only if the `Response` follows this exact shape:

- **Status `422`.** Any other status — `400`, `403`, `409`, 5xx — falls back to a generic error message in the UI.
- **Header `x-ollie-shop-custom-message: <your message>`.** This string is what the user sees.
- **Body:** JSON with `error` and `details` fields. Useful for downstream consumers and logs; the user-facing copy still comes from the header.

```ts
return new Response(
  JSON.stringify({
    error: "Validation failed",
    details: "The quantity of this item is limited to 5.",
  }),
  {
    status: 422,
    headers: {
      "Content-Type": "application/json",
      "x-ollie-shop-custom-message": "Max quantity is 5",
    },
  },
);
```

If you return a different status or omit the header, the front will still show *something* — but it will be a generic "an error occurred" instead of your text.

## Manipulating `requestInit`

The most under-appreciated capability of **request functions**: by returning a `Request` instead of the original, you can attach headers, cookies, or signed payloads the original client didn't include — typically to authenticate against a downstream API that the hub proxies on the customer's behalf.

```ts
return new Request(req.url, {
  method: req.method,
  headers: {
    ...Object.fromEntries(req.headers),
    "x-store-key": storeKey,
    "cookie": [req.headers.get("cookie"), `myToken=${token}`].filter(Boolean).join("; "),
  },
  body:
    req.method === "GET" || req.method === "HEAD"
      ? null
      : JSON.stringify(await req.json()),
});
```

Cookies, signed auth, request IDs, idempotency keys, modified bodies — all fair game inside the returned `Request`.

Two gotchas:

- The body is a stream. If you `await req.json()` to read it, you must pass the value back in the new `Request` (e.g. `JSON.stringify(await req.json())`), otherwise the upstream gets an empty body.
- Don't set `body` for `GET` or `HEAD` — those methods are not allowed to have a body and the runtime will reject the `Request`.

## Priority and chaining

When multiple functions match the same call, the hub runs them in `priority` order (ascending). Within the chain:

- Each function receives the **previous function's output** as input. A request function that returns a modified `Request` hands that modified request to the next request function in line.
- A function that returns a `Response` **stops the chain** — no later functions in that direction run, and the response is returned to the client (or replaces the upstream payload).
- Errors are handled per the `on_error` setting on each function independently.

When two functions need to run in a specific order, set their priorities explicitly. Default priority `0` means "no preference" and ordering becomes unstable in that case.

## Reading inputs without consuming the stream

`Request` and `Response` bodies are streams — they can only be read once. If you need to inspect the body and then forward it:

```ts
const body = await req.clone().json();
// ... use body ...
return new Request(req.url, { method: req.method, headers: req.headers, body: JSON.stringify(body) });
```

`req.clone()` is the safe way to peek; `req.json()` directly will consume the stream and break later reads.

## Deploying

The recommended path is the admin uploader — see the **Deploying** section of `cli-reference.md`. Briefly: zip the function folder (`./index.ts` + `./package.json`), drop it on the function's create/edit form in the admin, and the admin uploads + polls the build for you.
