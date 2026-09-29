# Pattern: Keeping the Resolver → Enricher cache under the storage limit

The [resolver → enricher chain](resolver-enricher-enforce-with-cookies.md) shares state
through `globalThis.sessionStorage`. That storage has a **~2 KB ceiling**, and blowing it
is the worst kind of bug: **`setItem` fails without throwing**. The enricher then reads an
empty cache, the component renders nothing, and there is no error anywhere — not in the
logs, not in the response, not in the browser.

The numbers below come from a real chain whose cached list silently rendered empty for hours.

---

## The three rules

### 1. Store only what the consumer destructures

The failure starts with a spread:

```ts
// ✘ 3918 bytes — the WHOLE upstream response
const payload = { ...providerResponse, tier, entitlements };
```

The enricher read exactly three fields. The other ~1.5 KB was upstream noise — echoed
line items, rule-evaluation traces, provider status and message fields, a discount
breakdown nobody read — plus the customer's first and last name, personal data persisted
with no consumer at all.

```ts
// ✔ 1478 bytes — only what `readCache()` destructures
const payload = { balance, tier, entitlements };
```

**Spreading an upstream response is how the payload grows without anyone deciding it
should.** One of those noise fields echoed every cart line, so the payload scaled with cart
size: a big enough cart blew the limit on its own, with no change on our side.

Grep the consumer before you write. If a field is not destructured there, it does not
travel.

### 2. Compress — do not encrypt — when the goal is size

Measured on the real 1478-byte payload:

| Approach | bytes | vs JSON |
|---|---|---|
| plain JSON | 1478 | — |
| AES-GCM + base64 | 2008 | **+36%** |
| **deflate + base64url** | **308** | **−84%** |
| deflate + AES-GCM + base64 | 344 | −77% |

Encryption **grows** the payload: a 12-byte IV, a 16-byte auth tag, then base64 multiplies
by 1.33. And encrypted bytes are incompressible, so the order can only ever be
compress → encrypt, never the reverse.

Cache payloads compress extremely well because they are repetitive by construction — the
same keys once per array element, the same identifiers repeated across groups.

```ts
import { deflateSync, inflateSync } from "node:zlib";

export const encodeCache = (v: unknown): string =>
  deflateSync(Buffer.from(JSON.stringify(v), "utf8")).toString("base64url");

export function decodeCache<T>(raw: string | null): T | null {
  if (!raw) return null;
  try {
    // Accept plain JSON too: during the deploy window the storage still holds the
    // previous format, and a shopper mid-session would otherwise read garbage.
    // `{` is the discriminator — base64url never produces it.
    if (raw.startsWith("{")) return JSON.parse(raw) as T;
    return JSON.parse(
      inflateSync(Buffer.from(raw, "base64url")).toString("utf8"),
    ) as T;
  } catch {
    return null; // corrupt cache === missing cache; never throw in the enricher
  }
}
```

**On encrypting anyway:** ask who the attacker is first. For shopper-scoped data the
content is already on their screen, so confidentiality buys nothing. Tampering is usually
rejected upstream — a forged identifier still has to survive the provider that issued it. If
you genuinely need integrity, an HMAC is cheaper than a cipher; if you need both, use
AES-GCM (authenticated encryption), never CBC without a MAC.

### 3. Log both sizes at the write

The write is the last unobserved step of the invocation. Instrument it or you will be
guessing:

```ts
const json = JSON.stringify(payload);
const stored = encodeCache(payload);
console.log(`${LOG} write:start`, JSON.stringify({
  json: json.length,      // how big the data really is
  stored: stored.length,  // what actually has to fit the ceiling
  entries: list.length,   // what drives the growth
}));
sessionStorage.setItem(KEY, stored);
console.log(`${LOG} write:done`);
```

`write:done` missing while `write:start` is present localises the failure to the write in
one query. And `entries` is what tells you the payload scales with something you do not
control — the size of a per-shopper collection, not the cart.

---

## Budget the growth, not the current size

The number that matters is not today's payload — it is what the payload becomes for the
worst shopper. Ask what multiplies it:

| Grows with | Example | Consequence |
|---|---|---|
| cart lines | echoing the item array | a big cart breaks checkout for that shopper only |
| a per-shopper collection | 52 entries across 8 groups | breaks for your best customers first |
| upstream response shape | `{ ...response }` | breaks when the provider adds a field |

The last one is the meanest: nothing in your code changed.

Measured on the same payload, that array of 52 identifiers cost ~900 bytes on its own,
while the UI ever used at most one per group. Sending one identifier plus a `count` instead
of the full array removed the growth term entirely. **Check whether the consumer needs the
collection or just one member and a count.**

---

## Checklist

- [ ] Every field in the payload is destructured by the consumer — verified by grep, not memory
- [ ] No `{ ...upstreamResponse }` spread
- [ ] No PII that no consumer reads
- [ ] Arrays checked: does the consumer need all members, or one plus a count?
- [ ] `deflate` + `base64url`, not encryption, when the goal is size
- [ ] `decodeCache` returns `null` on corruption and accepts the previous format
- [ ] `write:start` logs `json` and `stored` lengths; `write:done` follows it
- [ ] Growth term identified and bounded

---

## Related

- [resolver-enricher-enforce-with-cookies.md](resolver-enricher-enforce-with-cookies.md) — the chain this budget applies to
- [hub-functions.md](../hub-functions.md) — invocation phases and function chaining
