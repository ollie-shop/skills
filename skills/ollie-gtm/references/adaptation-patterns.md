# Adaptation Patterns and Lessons Learned

Proven patterns and pitfalls discovered during Ollie GTM container adaptations.

---

## Adaptation Checklist

When adapting a tag from an existing container:

1. **Create new tag** with `[Ollie] ` prefix
2. **Keep** the same tag type and static parameters (conversion IDs, measurement IDs, etc.)
3. **Replace** all variable references using Ollie naming conventions (see `variable-mapping.md`)
4. **Create new trigger** with guard condition and GA4 event name
5. **Create new variables** if they don't exist yet (prefer DLV over JS variables)
6. **Tags are active by default** — do NOT set `"paused": true` unless the user explicitly requests it
7. **Do not modify** the original tag or trigger

## Minimum Setup for a New Client

To set up GTM for any new Ollie client:

1. **Variable**: `Ollie Checkout Provider` (cookie `checkout_provider`)
2. **DLV Variables**: `transaction_id`, `value`, `currency`, `coupon`, `items`, `shipping`, `tax`
3. **Triggers**: one per GA4 event, each with `Ollie Checkout Provider` guard
4. **GA4 Event Tags**: pointing to the DLV variables
5. **Platform-specific CJS variables**: only as needed per ad platform

---

## Lessons Learned

### Enrichment tag pattern (dataLayer.push override)

Some clients have a Custom HTML tag that overrides `dataLayer.push` to intercept events, enrich items with promotion/category data, and re-emit with an `_enriched` suffix. When this pattern exists, **do NOT create separate GA4 Ollie tags** — create an `[Ollie]` version of the enrichment tag instead. The GA4 tags already listen to `_enriched` events and will consume both flows.

### GA4 tags with `sendEcommerceData: true` don't need DLV variables

The Ollie dataLayer is already in GA4 format. Tags can use `sendEcommerceData: true` + `getEcommerceDataFrom: dataLayer` directly.

### Shared GA4 event tags work for both checkouts

GA4 tags that fire on standard GA4 event names (`view_cart`, `add_to_cart`, `begin_checkout`, `purchase`, etc.) with `sendEcommerceData: true` work for **both** VTEX and Ollie without modification — both platforms push the same GA4 events in the same format. Only tags that fire on VTEX-specific events (e.g., `orderPlaced`, `addToCart`) or depend on VTEX APIs (`vtexjs`, jQuery + VTEX DOM) need `[Ollie]` versions.

For these shared tags, add `checkout_provider` as a GA4 event parameter to enable segmentation in GA4 reports (see `container-generation-flow.md` Step 6).

### Purchase Defer pattern (sessionStorage)

Ollie fires `purchase` on the checkout page, but platforms expect it on the order confirmation page. Pattern: capture enriched payload to sessionStorage on checkout, re-push on orderPlaced via `WINDOW_LOADED` trigger, block the GA4 purchase tag from firing on checkout using a negated `CONTAINS` filter.

### Enriching items from `__RAW_CHECKOUT_SESSION__`

The Ollie SDK sends minimal item data. Cross-reference with `__RAW_CHECKOUT_SESSION__.items[]` matching by `rawItem.id === item.item_id` (skuId) to fill: `productId` as `item_id`, category hierarchy via `productCategoryIds` order + `productCategories` map, `additionalInfo.brandName`, `refId`, `skuName`, `additionalInfo.dimension`. See `items-enrichment.md` for the full CJS pattern.

### Copy root-level fields from original push

The Ollie SDK sends extra fields outside `ecommerce` (payment_method, installments, userId, etc.). The enrichment tag must iterate and copy these to the enriched push, skipping only `event`, `ecommerce`, `ecommerceV2`, `gtm.uniqueEventId`.

### Tags that break in Ollie checkout

Any tag using `vtexjs`, jQuery `$` without guard, VTEX DOM selectors, or loading VTEX-specific external scripts will error in Ollie checkout. Add blocking trigger to these. Also check tags on All Pages triggers — they fire on checkout too.
