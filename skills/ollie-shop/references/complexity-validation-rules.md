# Complexity Validation Rules

Keep components, functions, and actions **simple and maintainable** by following these quantitative rules. Not prescriptive (no "thou shalt use hooks X way") — just bounds that prevent cognitive overload.

**How to use:**
1. **[Specs for Complexity Rules](specs-for-complexity-rules.md)** — automated testing patterns to validate these rules
2. **[Coding Standards](coding-standards.md)** — SDK-specific implementation patterns and safety nets
3. **Self-assessment checklist** (below) — use before every PR

---

## Components (React)

| Rule | Limit | Why |
|------|-------|-----|
| **Max `useState` count** | ≤ 2 | More than 2 = likely state duplication. Derive instead of sync manually. |
| **Max `useEffect` count** | ≤ 1 | Each effect = separate concern. >1 = mix concerns into custom hooks. |
| **Conditional nesting in useEffect** | ≤ 1 level | `if { if { ... } }` = hard to reason. Extract helpers. |
| **Derived booleans inline** | ≤ 2 | More = "mental stack" of conditions. Extract to helper or hook. |
| **Lines in render** | ≤ 100 | Longer = harder to test, reason about, refactor. Consider slot composition. |
| **Defensive parsing** | Extract to helper function | `(x as Type)?.prop?.nested` = confusing. Pull into `getX()` pure function. |

### Why this matters

```typescript
// ❌ Complex B2BPaymentTerms pattern
const [days, setDays] = useState<number | undefined>(undefined);      // state 1
const [synced, setSynced] = useState(false);                          // state 2
useEffect(() => {                                                      // effect 1
  if (!isVisible || synced) return;                                   // guard 1
  if (initialDays !== undefined) {                                    // guard 2
    setDays(initialDays);                                             // sync state 1
    setSynced(true);                                                  // sync state 2
    return;
  }
  if (session && firstOption !== undefined) {                         // guard 3
    setDays(firstOption);                                             // sync state 1 again
    setSynced(true);                                                  // sync state 2 again
    if (autoPersistDefault) void save(firstOption);
  }
}, [...8 deps]);

// ✅ Simple: extract sync logic to custom hook
const days = useSyncPaymentTerm(initialDays, session, firstOption, autoPersistDefault);

// Custom hook isolates the state sync:
function useSyncPaymentTerm(initialDays, session, firstOption, autoPersistDefault) {
  const [days, setDays] = useState(undefined);
  const [synced, setSynced] = useState(false);
  
  useEffect(() => {
    // one concern: sync state when deps change
    // easier to read, test, modify
  }, [initialDays, session, firstOption, autoPersistDefault]);
  
  return days;
}
```

---

## Hub Functions

| Rule | Limit | Why |
|------|-------|-----|
| **Try-catch blocks** | ≤ 1 envelope | Multiple = decoupled concerns. Extract to helpers. |
| **Guards before mutation** | Always | Validate shape first, mutate after. Prevents silent corruption. |
| **Explicit fallback on error** | Required | Never silently ignore. Fail-open or logged. |
| **Function body lines** | ≤ 50 | Logic only. Longer = extract phases to helpers. |

### Why this matters

```typescript
// ❌ Complex enforcement with nested error handling
async function handler({ req }) {
  try {
    const body = await req.json();
    try {
      const of = await fetchOrderForm(ofid);
      try {
        // enforce logic
      } catch {
        console.error("enforce failed");
      }
    } catch {
      console.error("orderForm fetch failed");
    }
  } catch {
    console.error("parse failed");
  }
}

// ✅ Simple: one envelope, guards first
async function handler({ req }) {
  try {
    const body = (await req.clone().json()) as UpdateBody;
    const orderItems = body.orderItems ?? [];
    if (orderItems.length === 0) return req; // guard first

    const of = await fetchOrderForm(ofid);
    if (!of.ok) return req; // guard on fetch too

    // THEN enforce logic
    let changed = false;
    for (const update of orderItems) {
      if (typeof update.index !== "number") continue; // guards in loop
      const item = of.items[update.index];
      if (!item) continue;
      
      // apply rule
      if (applyRule(update, item)) changed = true;
    }

    if (!changed) return req; // nothing to do
    return new Request(req.url, { method: req.method, headers: req.headers, body: JSON.stringify(body) });
  } catch (err) {
    console.error("[enforce] error", err);
    return req; // fail-open: always return something
  }
}
```

---

## Actions (Hooks & API Functions)

| Rule | Limit | Why |
|------|-------|-----|
| **Sensitive data handling** | Never inline | CNPJ, CPF, card data = extract to vault/API. Never stringify inline. |
| **Fetch without error boundary** | None | If you fetch, wrap in try-catch. Errors should fail gracefully. |
| **Log statements** | With `[prefix]` | `[action-name] message` = traceable. Avoid bare `console.log`. |
| **Cache miss fallback** | Required | If reading cache, define what happens on miss (null, live fetch, default). |

---

## Parsing & Type Safety

| Rule | Limit | Why |
|------|-------|-----|
| **Raw casting inline** | Avoid; extract | `(x as Type)?.prop` = confusing. Use `asType(x)` helper. |
| **Optional chaining chains** | ≤ 2 levels | `a?.b?.c?.d` = error-prone. Extract to `getNestedValue(a)`. |
| **Defensive parsing** | Always | External data (rawSession, response JSON) = always `.catch(() => null)` or `.catch(() => {})`. |

**See [Coding Standards § Unwrap progressively](coding-standards.md#unwrap-progressively-with-optional-chaining) for SDK-specific guidance on REQUEST responses.**

```typescript
// ❌ Confusing
const salesChannel = (rawSession as { salesChannel?: string } | undefined)?.salesChannel;
const isWholesale = salesChannel === "2";
const isCorporate = isCnpj(session?.customer?.document);
const isVisible = isWholesale || isCorporate;

// ✅ Clear
function isB2BEligible(rawSession: any, customer: any): boolean {
  const isWholesaleChannel = rawSession?.salesChannel === "2";
  const isCorporateCustomer = isCnpj(customer?.document);
  return isWholesaleChannel || isCorporateCustomer;
}
const isVisible = isB2BEligible(rawSession, session?.customer);
```

---

## Comments & Naming

| Rule | Limit | Why |
|------|-------|-----|
| **Comment block length** | ≤ 2 lines | Longer = code isn't clear. Refactor + rename instead. |
| **Variable names** | Self-documenting | `data` is bad. `paymentTermOptions`, `isB2BEligible` are good. |
| **Function names** | Express intent | `useSyncPaymentTerm` tells you what it does. `usePayment` does not. |

---

## PR Checklist

Before committing:

### Components
- [ ] `useState` count ≤ 2
- [ ] `useEffect` count ≤ 1 (or extracted to custom hooks)
- [ ] Max 1 level of nesting in any `if` block
- [ ] Derived booleans ≤ 2 inline
- [ ] Defensive parsing extracted to named functions
- [ ] Render body ≤ 100 lines
- [ ] No comment block > 2 lines

### Functions (Hub)
- [ ] Try-catch envelope ≤ 1
- [ ] Guards before mutation
- [ ] Fallback on error explicit (fail-open or logged)
- [ ] Logic body ≤ 50 lines

### Actions
- [ ] No sensitive data inline
- [ ] Fetch wrapped in try-catch
- [ ] Logs use `[prefix]`
- [ ] Cache miss fallback defined

### General
- [ ] Names are self-documenting
- [ ] No multi-level optional chaining (extract to helper)
- [ ] Defensive parsing always in place

---

## Rationale

These rules exist to keep code **maintainable** as the checkout grows. They're not about style — they're about cognitive load:

- **useState/useEffect limits**: Prevent "state synchronization chaos" which makes bugs hard to find.
- **Nesting limits**: Reduce "mental stack" needed to trace logic.
- **Line limits**: Make code reviewable and testable in isolation.
- **Error handling rules**: Ensure nothing silently fails; bugs are loud.
- **Naming rules**: Code is read 10x more than written. Name for the reader.

If code hits 2+ limits, it's a sign to refactor before merging.
