---
name: ollie-gtm
description: "Configure Google Tag Manager containers for Ollie Shop headless checkout. Use when adapting GTM tags, triggers, and variables to work with Ollie checkout, creating Ollie-prefixed tags from existing ecommerce platform tags, mapping data layer variables to GA4 ecommerce standard, or setting up tracking for Ollie checkout events (view_cart, add_to_cart, begin_checkout, purchase, etc.). USE FOR: GTM adaptation, ecommerce tracking, data layer configuration, tag migration, Ollie checkout analytics. DO NOT USE FOR: general GTM questions unrelated to Ollie, frontend component development, Ollie SDK integration."
metadata:
  author: ollie-shop
  version: "2.0.0"
---

# Ollie GTM Configuration

Adapt any ecommerce platform's GTM container to work with Ollie Shop headless checkout. Ollie replaces native checkout with a headless one; original tags must not fire — dedicated `[Ollie]`-prefixed counterparts fire instead, reading from Ollie's GA4-format data layer and `__RAW_CHECKOUT_SESSION__`. The `checkout_provider` cookie (`"ollie"` when active) is the routing mechanism: original tags fire when absent, Ollie tags fire when present. **Never modify original tags** — Ollie tags are always separate entities.

## Routing table

Pick the reference file based on user intent. Load it before proceeding.

| Intent | Load |
|---|---|
| "adapt a container", "generate Ollie container", "migration steps" | `references/container-generation-flow.md` |
| "data layer", "how do Ollie events work", "variable mapping", "DLV" | `references/variable-mapping.md` |
| "GTM JSON format", "how to write tags/triggers/variables in JSON" | `references/gtm-json-structures.md` |
| "items enrichment", "category hierarchy", "enrich items" | `references/items-enrichment.md` |
| "third party pixels", "ad platform variables", "CJS patterns" | `references/third-party-cjs-patterns.md` |
| "VTEX events", "what events does VTEX fire", "VTEX dataLayer" | `references/vtex-analytics.md` |
| "debugging", "lessons learned", "common patterns", "checklist" | `references/adaptation-patterns.md` |

When a request straddles two modes, start with `container-generation-flow.md` (the procedural backbone) and pull in others as needed.

## Data sources

| Source | Type | When to use |
|---|---|---|
| `dataLayer` | GA4 Ecommerce | Primary source. Standard GA4 ecommerce events pushed by Ollie SDK. |
| `window.__RAW_CHECKOUT_SESSION__` | JS global | Fallback. Same structure as platform orderForm. Use only when data layer lacks a field. |
| `checkout_provider` cookie | First-party cookie | Mandatory guard for every Ollie trigger. Value is `"ollie"` when active. |

## Naming conventions

| Entity | Pattern | Example |
|---|---|---|
| Tag | `[Ollie] ` prefix (replace existing bracket prefixes) | `[Ollie] GA4 - Begin Checkout` |
| Trigger | `[Ollie] {ga4_event} Trigger` | `[Ollie] purchase Trigger` |
| DLV | `[Ollie] DLV - GA4 - {field}` | `[Ollie] DLV - GA4 - transaction_id` |
| JS Var | `[Ollie] JS - {field}` | `[Ollie] JS - user_id` |
| CJS Var | `[Ollie] cjs - {description}` | `[Ollie] cjs - basketProductsIdsWComma` |

## Mandatory guard rule

**Every** Ollie trigger must include this filter condition — no exceptions:

```
{{Ollie Checkout Provider}} contains "ollie"
```

`Ollie Checkout Provider` is a first-party cookie variable (type `k`) reading the `checkout_provider` cookie. See `references/gtm-json-structures.md` for JSON.

## Trigger rules

1. **Always create a new trigger** — never reuse originals
2. **Mandatory guard**: `{{Ollie Checkout Provider}} contains "ollie"` (see above)
3. **Optional page path filter**: add `{{Page Path}} contains "checkout"` for cart/bag-scoped events
4. **Remove platform-specific filters**: CSS selectors, DOM checks, platform conditions
5. **Event type**: `CUSTOM_EVENT` with GA4-standard event names

## Event mapping

Ollie fires GA4-standard ecommerce events:

| GA4 Event | When it fires |
|---|---|
| `view_cart` | Checkout loads with items |
| `add_to_cart` | Item added to cart |
| `remove_from_cart` | Item removed from cart |
| `begin_checkout` | Checkout flow starts |
| `add_shipping_info` | Shipping address selected |
| `add_payment_info` | Payment method selected |
| `purchase` | Order completed |

Common platform mappings: `updatecart` (VTEX) -> `view_cart`, `addToCart` (VTEX) -> `add_to_cart`, `orderPlaced` (VTEX) -> `purchase`.

## Shared GA4 tags — no duplication needed

GA4 tags with `sendEcommerceData: true` firing on standard GA4 events work for **both** VTEX and Ollie — the data layer format is identical. Only tags on VTEX-specific events or with VTEX API dependencies need `[Ollie]` versions. For shared tags, add `checkout_provider` as a GA4 event parameter for segmentation (see `container-generation-flow.md` Step 6).

## Tags are active by default

New `[Ollie]` tags are created **active** (no `"paused": true`). Only pause if the user explicitly requests it.
