# Fraud Detector

## What it is

A **request** function that gates a server-side action (typically the order-finalization call) and picks one of several outcomes — *allow*, *block*, or **route the customer somewhere else first** with a 303 redirect — based on a verdict produced out-of-band. The defining feature is the **multi-outcome decision tree**: request functions usually only allow or reject; this one adds at least one third branch where the verdict says "the customer can recover from this, send them through that flow first".

The canonical use case is antifraud screening, hence the name. The same shape applies to any gate with more than two outcomes: KYC checks, B2B credit holds, age gates that can be cleared with a verification step, regional compliance walls, regulated-category warnings. The example below shows one concrete implementation (gift-card screening on the VTEX transaction endpoint, driven by a fraud-verdict cookie, with a block branch and a login-redirect branch). **Treat the concrete choices as illustrative** — the verdict source, the redirect targets, and the rule predicate are all merchant-specific; the pattern is the request gate + multi-outcome tree.

## Trigger type

- **`request`** — must run **before** the upstream call so a flagged action is stopped (or rerouted) before the platform processes it. Returning a `Response` short-circuits the upstream call.

## Where the function lives

```
./functions/<name>/
├── index.ts          # exports `handler: CustomFunction`
└── package.json
```

Deploy via the admin uploader as a zip with `./index.ts` + `./package.json` at the root.

## Handler shape — a worked example

The handler below is a complete, copy-able implementation for one scenario: screen high-value pure-gift-card transactions on a VTEX-backed checkout. It demonstrates every piece of the pattern (endpoint guard, context fetch, rule predicate, verdict read, three outcomes). The **shape** is the pattern; the cookie, the URLs, the threshold, and the rule predicate are choices you swap for your own.

```ts
import type { CustomFunction } from "@ollie-shop/functions";
import { FLAG_COOKIE } from "./cookie";

/**
 * RECOMMENDED TRIGGER:
 *   url:        https://{{account}}.vtexcommercestable.com.br/api/checkout/pub/orderForm/*orderFormId/transaction
 *   expression: method = "POST"
 *
 * RULE — only screen high-value pure gift-card orders (a common laundering shape).
 * Replace the predicate, the verdict source, and the redirect targets to fit
 * the merchant's actual antifraud (or KYC, age-gate, credit-hold, …) rule.
 *
 * Fail-open on every error path: never break checkout because our pipeline tripped.
 */
type OrderForm = {
  items: { name: string; price: number; quantity: number }[];
};

async function getOrderForm(orderFormId: string, origin: string): Promise<OrderForm> {
  const res = await fetch(
    new URL(`/api/checkout/pub/orderForm/${orderFormId}`, origin),
  );
  if (!res.ok) {
    throw new Error(`Failed to fetch orderForm ${orderFormId}: ${res.status}`);
  }
  return res.json() as Promise<OrderForm>;
}

export const handler: CustomFunction = async ({ req }) => {
  const url = new URL(req.url);

  // 1. Endpoint guard. The trigger already narrows, but stay safe if it widens later.
  const orderFormId = /\/api\/checkout\/pub\/orderForm\/([^\/]+)\/transaction/
    .exec(url.pathname)?.[1];
  if (!orderFormId) return;

  // 2. The transaction body carries payment/address but NOT the items — fetch the
  //    orderForm so the rule has the cart to inspect. (Other gates may need to
  //    fetch a customer profile, a session enrichment, etc. instead.)
  let orderForm: OrderForm;
  try {
    orderForm = await getOrderForm(orderFormId, url.origin);
  } catch (error) {
    console.error(`fraud-detector: orderForm fetch failed for ${orderFormId}`, error);
    return; // fail-open.
  }

  // 3. Rule predicate — keep it pure: cart in, boolean out. Swap for whatever
  //    the merchant's actual "is this in scope" test is.
  const isPureGiftCardOrder = orderForm.items.every((item) =>
    item.name.toLowerCase().startsWith("gift card"),
  );
  if (!isPureGiftCardOrder) return;

  const totalAmount = orderForm.items.reduce(
    (sum, it) => sum + it.price * it.quantity,
    0,
  );
  if (totalAmount <= 500_00) return; // R$ 500 threshold — pull from env vars in real use.

  // 4. Read the verdict. This example reads a cookie set by the antifraud service;
  //    other implementations may read a request header, a sessionStorage key
  //    written by a sibling resolver, or a field on the orderForm.
  const cookies = req.headers.get("cookie") ?? "";

  // 5. Pick the outcome. This example shows one block branch + one redirect branch;
  //    keep, drop, or add branches based on the merchant's verdict values.
  //    The block response sets FLAG_COOKIE so the device stays flagged for the
  //    cookie's TTL (see "Persist the block on the device" below).
  if (cookies.includes("fraud=true")) {
    return Response.json(
      { success: false, message: "Order flagged as fraudulent" },
      {
        status: 403,
        headers: {
          "x-ollie-shop-custom-message":
            "Detectamos um comportamento suspeito neste pedido. Por favor, revise os detalhes antes de prosseguir com a compra.",
          "set-cookie": FLAG_COOKIE,
        },
      },
    );
  }

  if (cookies.includes("fraud=login")) {
    // Example redirect: send the customer to authenticate before retrying.
    // Swap the Location for whatever follow-up flow the merchant wants
    // (verification, captcha, regulated-purchase confirmation, queue, etc.).
    return new Response("This order requires login.", {
      status: 303,
      headers: { Location: "https://<store>.com.br/login?returnUrl=/checkout" },
    });
  }
};
```

## The outcomes

| Outcome | Status | Required headers | What the client does |
|---|---|---|---|
| **Allow** | — (return `undefined`) | — | Action goes upstream unchanged. |
| **Block** | `403` / `422` (any non-2xx) | `x-ollie-shop-custom-message: <copy>` | Surfaces the message to the customer; action is cancelled. See the rejection contract in `hub-functions.md`. |
| **Redirect** | `303` | `Location: <url>` | Browser follows the redirect — customer lands on whatever flow the merchant wants them to complete before retrying. |

You can have **multiple distinct redirect outcomes** in a single function — each driven by a different verdict value, each pointing at a different `Location`. The example uses one (login), but the same shape extends to N branches. The `Location` URL is fully merchant-controlled: a sign-in page, a verification step, a captcha, a 3DS challenge, a regulated-purchase confirmation, a queue page, a merchant-built remediation flow — anything the browser can navigate to. The function only decides *whether* to redirect; the URL is the merchant's call.

## Where the verdict comes from

The verdict is **out-of-band** — produced by something other than the function itself, then read by the function. The example reads a cookie; common alternatives:

- **Cookie** set by an external service (most decoupled — any service that can set a cookie on the merchant domain can write the verdict, and the function only needs a string match). *Used by the example.*
- **Request header** injected by an earlier hub function or by the storefront.
- **`globalThis.sessionStorage` key** written by a sibling resolver/enricher function (see the resolver→enricher→enforce pattern).
- **A field on the orderForm / customer profile / session enrichment** — when the rule can be evaluated from data already in the cart (no external verdict, the function is the verdict). *The gift-card-threshold check in the example is this kind of inline rule.*
- **A synchronous external API call** — works, but every call you add costs latency on the gated action's hot path. Prefer to cache the verdict via a sibling resolver before this function runs.

Pick a source and document it. The function only depends on the contract (a value the function can read), not on who produced it — so the verdict mechanism can change without rewriting the handler.

## Persist the block on the device (default)

The block response above sets a signed cookie (`FLAG_COOKIE`) so the device stays flagged for the cookie's TTL. Bundle a `cookie.ts` helper alongside `index.ts`:

```ts
// ./functions/fraud-detector/cookie.ts
import { createHmac, randomBytes } from "node:crypto";

// Process-local HMAC key. Flags persist for the lifetime of this
// function instance; when the instance recycles, old flags stop
// verifying (and act as unflagged). Acceptable trade-off for a
// soft signal — do NOT use this alone for hard blocks.

export const COOKIE_NAME = "__Host-fraud";
const TTL_SECONDS = 10 * 60;
const PAYLOAD = "flagged";
const SECRET = randomBytes(32);

function sign(val: string) {
  return `${val}.${createHmac("sha256", SECRET).update(val).digest("base64url")}`;
}

function readCookie(req: Request) {
  return req.headers
    .get("cookie")
    ?.split(";")
    .map((p) => p.trim())
    .find((p) => p.startsWith(`${COOKIE_NAME}=`))
    ?.slice(COOKIE_NAME.length + 1);
}

export function isDeviceFlagged(req: Request): boolean {
  const value = readCookie(req);
  if (!value) return false;
  const dot = value.lastIndexOf(".");
  if (dot <= 0) return false;
  return sign(value.slice(0, dot)) === value;
}

export const FLAG_COOKIE = `${COOKIE_NAME}=${sign(PAYLOAD)}; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=${TTL_SECONDS}`;
```

The cookie is HMAC-signed (client can't forge it), `HttpOnly` (browser JS can't touch it), `Secure` + `SameSite=Lax` (HTTPS only, checkout-flow same-site), and TTL-bounded so a false positive decays automatically. The HMAC secret is generated once per function instance — when the Lambda recycles, old flags stop verifying and act as unflagged. That bounds max persistence.

### Optional fast-path

Once flagged, the default handler still runs the full verdict pipeline (orderForm fetch, rule check, verdict read) on the next request from the same device. If that lookup is expensive and you want to skip it while the flag is active, add `isDeviceFlagged` to the import and short-circuit right after the endpoint guard:

```ts
if (isDeviceFlagged(req)) {
  // return the same block response as the main branch below
}
```

Only apply the fast-path to the *block* outcome. Never short-circuit a redirect branch (see antipatterns).

### When to skip this pattern

Drop the cookie import and the `set-cookie` header entirely if any of these apply:

- **Hard blocks** — KYC / regulatory / age-gate that must persist across sessions. The cookie is a soft signal any user can clear; combine with server-side state (customer profile flag, session enrichment) for hard requirements.
- **Idempotent, cheap-to-re-evaluate rules** — if the rule can be checked from data already in the request (no external fetch) and the customer's *situation* may have changed since the last block, forcing a re-evaluation each time is cheaper than caching a stale verdict.
- **Only redirect outcomes** — the redirect branch assumes the customer will complete the remediation flow and retry. Caching a "redirect" state defeats the retry loop; only cache the terminal *block* decision.

## Behavior

1. **Match the gated action defensively.** The trigger filters the URL, but parse `req.url` and bail out cheaply if the request doesn't match — the function stays safe if a future trigger gets widened by mistake.
2. **Load extra context if the request doesn't carry it.** Transactions carry payment, not items — fetch the orderForm. Other gates may need a customer profile or a session enrichment. Use the same origin and forward the original `cookie` header.
3. **Apply the rule predicate.** Keep it pure: inputs in, boolean out. Most calls — the 99% case — return `undefined` here and do nothing else.
4. **Read the verdict.** From whichever source the merchant uses (cookie in the example). Absence of a verdict is "verdict not made yet" — pass through, never block by default.
5. **Pick the outcome** — allow, block, or one of the redirect branches. On block, set `FLAG_COOKIE` in the response headers so the device stays flagged for the cookie's TTL.
6. **Fail-open on every error path.** Context fetch failed, parse error, network blip — return `undefined`. Breaking the action because our own pipeline tripped is a worse outcome than a missed catch.

## States to cover

- **Rule doesn't apply** — pass through immediately, no fetch, no work.
- **Rule applies, verdict absent** — pass through.
- **Rule applies, verdict says "block"** — return the rejection shape (4xx + `x-ollie-shop-custom-message` + `set-cookie: FLAG_COOKIE`).
- **Rule applies, verdict says "redirect"** — return `303` + `Location: <merchant-defined target>`. Do NOT set the flag cookie on this branch.
- **Rule applies, multiple distinct redirect verdicts** — route each to its own `Location`.
- **Context fetch fails** — log and pass through.
- **URL doesn't match the gated action** — guard returns early.

## Trigger configuration

```json
{
  "url": "https://<store>.<platform-host>/<gated-action-path>",
  "expression": "method = \"POST\""
}
```

Document this trigger as a JSDoc block at the top of `index.ts` (see the handler shape above and `references/hub-functions.md` §"Document the recommended trigger inline"). Narrow `expression` aggressively — a gate on a read path is wasted latency and risks breaking the read if the function errors. The transaction endpoint specifically only takes `POST`.

## Specific questions to ask before coding

> Cross-cutting integration questions (HAR, payload, auth scheme) live in `references/component-design-flow.md`.

1. **What action is being gated?** Transaction, item-add, a custom merchant endpoint? The trigger URL drops out of this answer.
2. **Where does the verdict come from?** Cookie, header, session storage, an inline rule on existing context, external API? Pick one and document the contract.
3. **What are the verdict values, and what does each one do?** The pattern supports allow/block plus N distinct redirect targets — most merchants need two or three. Drop branches that don't apply, add ones that do.
4. **Where do the redirect targets live?** Each one is merchant-specific. Hardcoding them in source is OK to start; move to function env vars when the merchant runs more than one storefront or the URL becomes part of an A/B test.
5. **What's the block message?** Goes in the `x-ollie-shop-custom-message` header on the 4xx. Confirm the final copy in the right language with the customer team.

## Antipatterns

- **Don't fail-closed when the verdict is missing.** Absence means "we don't know yet" — passing through is correct. Blocking by default punishes every customer when the verdict-producer has a hiccup.
- **Don't put the gate in a `response`-flavored function.** By the response phase the platform has already processed the action; reversing it is asynchronous and lossy. Always gate at the request phase.
- **Don't run on `GET`s.** A gate on a read path is wasted latency and risks breaking the read if the function errors.
- **Don't omit the `Location` header on a 303.** The browser needs it to follow the redirect.
- **Don't put the user-facing copy only in the JSON body.** The action client reads the message from the `x-ollie-shop-custom-message` header.
- **Don't tie the function to one verdict source.** Read the verdict through a single helper so swapping cookie → header → session storage is a one-line change.
- **Don't call a slow external service synchronously inside the gate.** Either cache the verdict via a sibling function and read from the cache here, or accept a tight timeout and a fail-open on miss.
