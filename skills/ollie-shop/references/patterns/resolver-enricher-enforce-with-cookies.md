# Pattern: Resolver → Enricher → Enforce with Session Storage & Cookies

When a customization needs **per-item data the platform doesn't carry natively** and that data is **expensive to fetch**, split the work across three hub functions that share state through `globalThis.sessionStorage` and use **cookies to pass context** between them.

---

## Why this pattern

- **`sessionStorage` is shared per session** — data cached in the resolver is instantly available to enricher + enforce without re-fetching.
- **Three separate roles** reduce latency:
  - **Resolver**: fetches once, off the critical path (storefront initiates it).
  - **Enricher**: runs on every orderForm response, but only reads from warm cache (cheap).
  - **Enforce**: runs on cart mutation, reads cache + applies rules before the write.
- **Cookies carry request context** — like the orderForm ID — without polluting the request body.

---

## Architecture: 3-function chain

```
┌────────────────────────────────────────┐
│ Storefront component fires ONE request │
│ to `/api/_v/merchant/thing` (batch)    │
│ Guarded by persisted cart signature    │
└────────────────────────────────────────┘
           ↓
    [RESOLVER]
  (request phase)
           ↓
   Fetch external API
   Cache in sessionStorage
   Short-circuit response
           ↓
┌────────────────────────────────────────┐
│ Every orderForm response → ENRICHER    │
│ (response phase)                       │
│ Read warm cache, write onto items      │
└────────────────────────────────────────┘
           ↓
┌────────────────────────────────────────┐
│ Cart mutation → ENFORCE                │
│ (request phase)                        │
│ Extract cookie, fetch orderForm        │
│ Read cache, apply rule                 │
└────────────────────────────────────────┘
```

---

## Part 1: RESOLVER — fetch once, cache it

**Invocation:** `"request"`

**What it does:**
1. The storefront fires a single **batched** request carrying all keys to enrich.
2. Resolver fans out to the external API (one call per key, or batch if possible).
3. Caches each result in `sessionStorage` with a `<prefix>:<key>` naming scheme.
4. Short-circuits with a Response — the custom URL never goes upstream.

**Guards:**

| Guard | Why |
|-------|-----|
| `items.filter(it => it.id)` | Only process items with a valid key (non-null SKU). |
| `items.filter(it => !sessionStorage.getItem(prefix + it.id))` | Skip keys already cached this session. Fan-out only for uncached keys. |
| Parse request body defensively: `(await req.json().catch(() => ({})))` | Malformed JSON → empty body → safe fallback. |
| `try/catch` around external fetches. | Network errors should not crash the function. |

**Code shape:**

```ts
const SS_PREFIX = "data:"; // key prefix for sessionStorage

export const handler: CustomFunction = async ({ req }) => {
  // Guard: parse defensively
  const body = (await req.json().catch(() => ({}))) as { ids?: string[] };
  const ids = (body.ids ?? []).filter(Boolean);
  
  if (ids.length === 0) return new Response("{}", { status: 200 });

  // Guard: only fetch uncached
  const uncached = ids.filter((id) => !sessionStorage.getItem(SS_PREFIX + id));

  try {
    // Fetch from external API
    const data = await fetchData(uncached);
    
    // Cache results
    for (const [id, value] of Object.entries(data)) {
      sessionStorage.setItem(SS_PREFIX + id, JSON.stringify(value));
    }
  } catch (err) {
    console.error("fetch failed", err);
  }

  return new Response(JSON.stringify({ success: true }), {
    status: 200,
    headers: { "content-type": "application/json" },
  });
};
```

**Storefront guard (component-side):**

Fire the resolver only when cart changes. Use `localStorage` to guard on cart signature:

```tsx
const useResolveData = () => {
  const { session } = useCheckoutSession();
  const ids = session?.cart?.items?.map((it) => it.id) ?? [];
  const signature = ids.sort().join(",");
  const lastSignature = localStorage.getItem("resolve:signature");

  useEffect(() => {
    if (signature && signature !== lastSignature) {
      // Cart changed → fire resolver once
      fetch("/api/_v/merchant/data", {
        method: "POST",
        body: JSON.stringify({ ids }),
      });
      localStorage.setItem("resolve:signature", signature);
    }
  }, [signature, lastSignature]);
};
```

---

## Part 2: ENRICHER — read cache, write onto items

**Invocation:** `"response"` on `/api/checkout/pub/orderForm…`

**What it does:**
1. Reads the orderForm JSON from the response.
2. For each `items[i]`, reads `sessionStorage[<prefix>:<key>]`.
3. Writes the cached data onto each item.
4. **Always** returns a Response (function chaining — never `undefined`).

**Guards:**

| Guard | Why |
|-------|-----|
| `res.clone().json()` with `.catch()` → pass through on error | If response is not valid JSON, don't mutate it. |
| `items.length === 0` → return `res` early | No items → nothing to enrich. |
| Safe default on cache miss | Return `null` or `undefined` for missing cache. Never throw. |
| **Always return a Response** | Function chaining — never `undefined`. |

**Code shape:**

```ts
const SS_PREFIX = "data:";

function getCachedValue(id?: string): any {
  if (!id) return null;
  const raw = sessionStorage.getItem(SS_PREFIX + id);
  return raw ? JSON.parse(raw) : null;
}

export const handler: CustomFunction = async ({ res }) => {
  // Guard: parse JSON safely
  let body: any = await res.clone().json().catch(() => null);
  if (!body?.items?.length) return res; // guard: nothing to enrich

  // Write cached data onto each item
  for (const item of body.items) {
    item.customData = getCachedValue(item.id);
  }

  return new Response(JSON.stringify(body), { status: res.status, headers: res.headers });
};
```

---

## Part 3: ENFORCE — read context from cookie, apply rule

**Invocation:** `"request"` on `/api/checkout/pub/orderForm/{id}/items/update`

**What it does:**
1. Parses the update request.
2. **Extracts context from the `checkout.vtex.com` cookie** — e.g., the orderForm ID.
3. Fetches the current orderForm to map indices to items.
4. Reads cached data from `sessionStorage` (or fetches live on cache miss).
5. Applies the business rule (validate, modify, enforce).
6. Returns the rewritten Request (or original on any error — fail-open).

**Guards:**

| Guard | Why |
|-------|-----|
| `items?.length === 0` → return `req` | No items to update → nothing to enforce. |
| `typeof index !== "number"` | Malformed update request. Skip + continue. |
| `!item` (item not at index) | Index mismatch. Skip + continue. |
| Try-catch around entire flow | Any error → return original request, never break the update. |
| Extract context from cookie | Avoid polluting request body. Use `ofidFromCookie()` pattern. |
| Forward `cookie` header to internal fetch | VTEX API needs auth; the cookie carries the session. |

**Code shape:**

```ts
const SS_PREFIX = "data:";

function ofidFromCookie(cookie: string): string {
  for (const part of cookie.split(";")) {
    const [key, value] = part.trim().split("=");
    if (key === "checkout.vtex.com" && value?.startsWith("__ofid=")) {
      return value.slice("__ofid=".length);
    }
  }
  return "";
}

async function getCachedData(id: string): Promise<any> {
  const cached = sessionStorage.getItem(SS_PREFIX + id);
  if (cached) return JSON.parse(cached);
  
  // Cache miss: fetch live (expensive, only on miss)
  try {
    const res = await fetch(`/api/_v/merchant/data/${id}`);
    return res.ok ? await res.json() : null;
  } catch {
    return null;
  }
}

export const handler: CustomFunction = async ({ req }) => {
  try {
    const body = (await req.clone().json()) as any;
    const items = body.items ?? [];
    if (!items.length) return req; // guard: nothing to do

    const origin = new URL(req.url).origin;
    const cookie = req.headers.get("cookie") ?? "";
    const ofid = ofidFromCookie(cookie);

    // Fetch current orderForm to map indices
    const ofRes = await fetch(
      `${origin}/api/checkout/pub/orderForm/${ofid}`,
      { headers: { cookie } },
    );
    const of = (await ofRes.json().catch(() => ({}))).items ?? [];

    let changed = false;
    for (const update of items) {
      if (typeof update.index !== "number") continue; // guard: invalid index

      const cartItem = of[update.index];
      if (!cartItem?.id) continue; // guard: item not found at index

      const data = await getCachedData(cartItem.id);
      // Apply your business rule here
      if (applyRule(update, data)) {
        changed = true;
      }
    }

    if (!changed) return req; // nothing changed → pass through

    return new Request(req.url, {
      method: req.method,
      headers: req.headers,
      body: JSON.stringify(body),
    });
  } catch (err) {
    console.error("enforce error", err);
    return req; // fail-open on any error
  }
};

function applyRule(update: any, data: any): boolean {
  // Your custom validation/enforcement logic here
  // Return true if update was modified
  return false;
}
```

---

## Priority & chaining

Register the functions with explicit priorities (lower = earlier):

| Function | Invocation | Trigger URL | Priority | Notes |
|----------|------------|-------------|----------|-------|
| **enricher** | response | `/api/checkout/pub/orderForm` | `0` | Runs first; gets the warm cache when resolver fires. |
| **resolver** | request | `/api/_v/merchant/data` | — | Custom URL, fires on demand from component. |
| **enforce** | request | `/api/checkout/pub/orderForm/{id}/items/update` | `1` | Reads cached data, applies rules before cart write. |

---

## Guard checklist

### Resolver
- [ ] Parse request body defensively (`.catch(() => {})`)
- [ ] Filter to valid keys (non-null SKU)
- [ ] Skip already-cached keys
- [ ] Try-catch around external API calls
- [ ] Store results with consistent prefix

### Enricher
- [ ] Parse response JSON defensively
- [ ] Skip if no items
- [ ] Safe default on cache miss (null / undefined)
- [ ] Always return a Response

### Enforce
- [ ] Guard early: empty items → return req
- [ ] Extract context from cookie (not body)
- [ ] Try-catch the entire flow (fail-open)
- [ ] Validate update shape (index is number)
- [ ] Skip items not found at index
- [ ] Forward auth (cookie) to internal fetch
- [ ] Only rewrite if something changed

---

## Example file layout

When applying this pattern to a real customization (e.g. a sell-by-box multiplier), the three functions live as siblings under `./functions/`:

- Resolver: `functions/<name>-resolver/index.ts`
- Enricher: `functions/<name>-enricher/index.ts`
- Enforce: `functions/<name>-enforce/index.ts`
