# Container Generation Flow — Ollie versus VTEX

Step-by-step process for adapting an existing VTEX GTM container into a dual-mode container that supports A/B testing between Ollie checkout and native VTEX checkout, sharing the same GTM container ID.

---

## Step 0 — Repository Setup

Create or verify the `gtm/` directory structure at the repository root:

```
gtm/
├── vtex/          # Original VTEX container export (read-only reference)
├── ollie/         # Ollie-adapted container (working copy)
└── README.md      # Overview of the current VTEX container state
```

- **`vtex/`**: Contains the original GTM container JSON exported from the client's account. This is the baseline reference and should not be modified.
- **`ollie/`**: Contains the Ollie-adapted container. Initially a copy of the VTEX container. All adaptations are made here.
- **`README.md`**: General description of the client's current VTEX container — installed tags, key triggers, third-party integrations, and any known issues.

## Step 1 — Container Audit

Analyze the VTEX container to map everything that fires on checkout/cart pages.

### 1.1 — Map triggers fired by VTEX native events

Consult `references/vtex-analytics.md` for the full list of VTEX native dataLayer cart and checkout events. Scan all triggers in the container looking for those that fire on checkout/cart-scoped VTEX events.

For each matching trigger, record: trigger ID, name, type, event name, and any additional filter conditions.

### 1.2 — Map tags fired by checkout/cart triggers

For each trigger identified in 1.1, find all tags that use it as a firing trigger. For each tag, check if its script pushes new custom events to the dataLayer (e.g., `dataLayer.push({event: '...'})`). If it does, return to step 1.1 and trace those custom events as additional triggers — repeat until no new events are found.

### 1.3 — Map tags fired on All Pages

Identify all tags using the `All Pages` trigger. For each, document:

- Tag name, type, and purpose (analytics, pixel, anti-fraud, A/B testing, etc.)
- Whether it depends on jQuery, `vtexjs`, or VTEX DOM selectors
- Whether it's relevant to checkout/cart pages or only to storefront pages

### 1.4 — Map variables used

For each tag and trigger identified in 1.1–1.3, list all GTM variables referenced. For each variable, document:

- Variable name, type (DLV, CJS, JS, cookie, Lookup Table, etc.)
- What it reads (dataLayer path, cookie name, JS global, etc.)
- Whether it depends on VTEX-specific data (VTEX event names, `vtexjs`, VTEX DOM)

### 1.5 — Identify tags without native Ollie event coverage

Cross-reference the tags mapped in 1.2 and 1.3 against the events that Ollie fires natively (see SKILL.md "Event Mapping"). For each tag whose trigger depends on a VTEX event that Ollie does not fire, classify its relevance:

- **Critical**: Tag must fire on Ollie checkout for analytics/fraud/conversion accuracy (e.g., anti-fraud fingerprinting, purchase tracking). Requires an Ollie-specific adaptation.
- **Nice-to-have**: Tag adds value but checkout can operate without it (e.g., retargeting pixels on intermediate steps). Can be deferred.
- **Not applicable**: Tag is storefront-only and irrelevant to checkout pages (e.g., product impression tracking). No action needed.

### 1.6 — Document findings in README.md

Write the audit results into `gtm/README.md`. This serves as the human-readable summary of the container state and the basis for all subsequent adaptation steps.

## Step 2 — Resolve unsupported event dependencies

If Step 1.5 identified tags classified as **Critical** or **Nice-to-have** that depend on VTEX events not fired by Ollie, present each one to the user and ask how to proceed. Options per tag:

- **Adapt**: Create an `[Ollie]`-prefixed version using an available Ollie event or trigger (e.g., Page View with path filter, or a different GA4 event)
- **Skip**: Tag will not fire on Ollie checkout — accept the gap
- **Custom event**: Request Ollie to push a new custom event to cover this use case (requires Ollie SDK changes)

Document the user's decision for each tag before proceeding to the next steps.

## Step 3 — Create Ollie base infrastructure

Create the foundational entities that every Ollie-adapted container needs.

### 3.1 — Ollie Checkout Provider variable

Create a first-party cookie variable named `Ollie Checkout Provider` that reads the `checkout_provider` cookie. This variable is the routing mechanism for all Ollie triggers and is referenced by every guard filter.

### 3.2 — Ollie event triggers

Create one `[Ollie]` trigger per GA4 checkout event that the client's tags need. Each trigger is a `CUSTOM_EVENT` that listens for the GA4 event name and includes the mandatory guard filter: `{{Ollie Checkout Provider}}` contains `ollie`.

Only create triggers for events that are actually needed by tags identified in Steps 1–2. Common set:

- `[Ollie] purchase Trigger` — needed by conversion/ad tags
- `[Ollie] add_to_cart Trigger` — needed by ad pixel tags
- `[Ollie] add_payment_info Trigger` — needed by anti-fraud / payment tags

Additional triggers may be needed for client-specific tags (e.g., `page_view` with path filter for anti-fraud fingerprinting on the payment step).

### 3.3 — Ollie blocking trigger

Create a generic blocking trigger named `[Ollie] Blocking - Ollie Active`. This is a `CUSTOM_EVENT` with regex `.*` and the same guard filter (`checkout_provider` contains `ollie`). It is added as a **blocking trigger** to VTEX-specific tags that would break on Ollie checkout (e.g., tags using `vtexjs`, jQuery + VTEX DOM, or VTEX API calls).

## Step 4 — Adapt tags with VTEX-specific dependencies

For each tag classified as **Critical** in Step 1.5 that depends on `vtexjs`, jQuery, VTEX DOM selectors, or VTEX API endpoints:

1. **Create an `[Ollie]` version** of the tag with the same functionality but reading from Ollie data sources:
   - Replace `vtexjs.checkout.getOrderForm()` → `__RAW_CHECKOUT_SESSION__`
   - Replace `$(window).on('orderFormUpdated.vtex', ...)` → Ollie event trigger
   - Replace VTEX API calls (`/api/checkout/pub/...`) → equivalent Ollie mechanism or `__RAW_CHECKOUT_SESSION__` fields
   - Remove jQuery dependencies — use vanilla JS

2. **Assign the appropriate `[Ollie]` trigger** from Step 3.2.

3. **Add the blocking trigger** (from Step 3.3) to the **original** tag to prevent it from firing on Ollie checkout (where it would error).

Common example: anti-fraud fingerprinting tags (Signifyd, Konduto) that use `vtexjs` to get `orderFormId` and `clientProfileData.email` — adapt to read from `__RAW_CHECKOUT_SESSION__.orderFormId` and `__RAW_CHECKOUT_SESSION__.clientProfileData.email`.

## Step 5 — Adapt tags on VTEX-specific event names

For active tags that fire on VTEX event names (`addToCart`, `orderPlaced`, `cart`, etc.) rather than GA4 event names:

1. **Create an `[Ollie]` version** of the tag with the same type, static parameters (pixel IDs, conversion labels, etc.), and template configuration.

2. **Remap variable references** from VTEX-format DLVs to GA4-format DLVs:

   | VTEX DLV path | GA4 DLV path |
   |---|---|
   | `transactionId` | `ecommerce.transaction_id` |
   | `transactionTotal` | `ecommerce.value` |
   | `transactionProducts` | `ecommerce.items` |
   | `transactionCurrency` | `ecommerce.currency` |
   | `visitorContactInfo.0` | Use CJS reading `__RAW_CHECKOUT_SESSION__.clientProfileData.email` |

3. **Assign the appropriate `[Ollie]` trigger** from Step 3.2.

**Important**: Tags that already fire on GA4 event names (`view_cart`, `add_to_cart`, `begin_checkout`, `purchase`, etc.) and use `sendEcommerceData: true` with DLVs from `ecommerce.*` do **not** need `[Ollie]` versions. Both VTEX IO and Ollie push the same GA4 events in the same format. These tags work for both checkouts without modification.

## Step 6 — Add `checkout_provider` to shared GA4 event tags

For GA4 event tags that fire on checkout events and are shared between VTEX and Ollie (identified as "no changes needed" in Step 5):

1. Add `checkout_provider` as an **event parameter** in the tag's `eventSettingsTable`, reading from `{{Ollie Checkout Provider}}`.

2. This enables segmentation in GA4 reports (Ollie vs VTEX) without duplicating tags.

3. The value will be `"ollie"` when the Ollie checkout is active, or empty/undefined on VTEX checkout.

4. Instruct the client to register `checkout_provider` as a **custom dimension** in GA4 Admin (Admin → Custom definitions → Create custom dimension).

## Step 7 — Generate adapted container

1. **Create a Python script** in `gtm/scripts/` that:
   - Reads the original container from `gtm/vtex/`
   - Injects all new entities (variables, triggers, tags) with sequential IDs starting after the container's max existing IDs
   - Applies modifications to existing tags (blocking triggers, new parameters)
   - Writes the adapted container to `gtm/ollie/`

2. The script must be **idempotent** — running it again from the same input produces the same output. It always reads from `gtm/vtex/` (never from a previous `gtm/ollie/` output).

3. All new tags should be created **active** (no `"paused": true`) unless the user explicitly requests otherwise.

## Step 8 — Validate and document

### 8.1 — Validate the adapted container

Run validation checks on the generated JSON:

- **Entity counts**: expected totals match (original + new)
- **Guard filters**: every `[Ollie]` trigger has `checkout_provider` contains `ollie`
- **No VTEX references**: no `[Ollie]` tag contains `vtexjs`, `jQuery`, `$(document)`, `$(window)`, `orderFormUpdated.vtex`
- **Blocking triggers**: original VTEX-dependent tags have the blocking trigger assigned
- **Tag status**: all `[Ollie]` tags are active (or paused, per user preference)
- **Firing triggers**: every `[Ollie]` tag points to valid `[Ollie]` trigger IDs
- **Shared tags**: GA4 checkout event tags have `checkout_provider` parameter

### 8.2 — Update README.md

Update `gtm/README.md` with an **Adaptation Status** section documenting:

- Entities added (variables, triggers, tags) with IDs and names
- Modifications to existing tags
- Tags that required no changes and why
- Activation instructions (import JSON → publish workspace, register custom dimension in GA4)
