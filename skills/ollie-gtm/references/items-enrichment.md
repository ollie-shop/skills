# Items Enrichment Pattern

Use this pattern **only when** a tag requires data that the standard `ecommerce.items` data layer doesn't provide (e.g., full category hierarchy, product reference IDs, promotion data).

## When to Use

- Tag needs `item_category2` through `item_category5` (hierarchical categories)
- Tag needs `productRefId` for specific platform integrations
- Tag needs promotion/creative data from `localStorage`

## When NOT to Use

- The standard `{{[Ollie] DLV - GA4 - items}}` already has all needed fields
- The tag only needs `item_id`, `item_name`, `item_brand`, `item_category`, `price`, `quantity`

## Enrichment Variable Pattern

Create a Custom JavaScript variable (`jsm` type) named `[Ollie] DLV - GA4 - items enriched`:

```js
function() {
  var items = {{[Ollie] DLV - GA4 - items}};
  if (!items || !items.length) return [];

  var raw = (typeof __RAW_CHECKOUT_SESSION__ !== 'undefined') ? __RAW_CHECKOUT_SESSION__ : null;
  var rawItems = (raw && raw.items) ? raw.items : [];

  return items.map(function(item, index) {
    var enriched = {};
    // Copy all existing GA4 fields
    for (var key in item) {
      if (item.hasOwnProperty(key)) {
        enriched[key] = item[key];
      }
    }

    // Find matching raw item by index or ID
    var rawItem = rawItems[index] || null;
    if (!rawItem) {
      for (var i = 0; i < rawItems.length; i++) {
        if (String(rawItems[i].id) === String(item.item_id) ||
            String(rawItems[i].productId) === String(item.item_id)) {
          rawItem = rawItems[i];
          break;
        }
      }
    }

    if (rawItem) {
      // Enrich with category hierarchy from productCategories
      if (rawItem.productCategories) {
        var cats = Object.values(rawItem.productCategories);
        if (cats[0]) enriched.item_category = cats[0];
        if (cats[1]) enriched.item_category2 = cats[1];
        if (cats[2]) enriched.item_category3 = cats[2];
        if (cats[3]) enriched.item_category4 = cats[3];
        if (cats[4]) enriched.item_category5 = cats[4];
      }

      // Enrich with product reference ID
      if (rawItem.productRefId) {
        enriched.product_id = rawItem.productRefId;
      }
    }

    // Enrich with promotion data from localStorage (if available)
    try {
      var promoData = localStorage.getItem("select_items_promotion");
      if (promoData) {
        var promos = JSON.parse(promoData);
        var promo = promos[item.item_id] || promos[index];
        if (promo) {
          enriched.promotion_id = promo.promotion_id || '';
          enriched.promotion_name = promo.promotion_name || '';
          enriched.creative_name = promo.creative_name || '';
          enriched.creative_slot = promo.creative_slot || '';
        }
      }
    } catch(e) {}

    return enriched;
  });
}
```

## Normalizations (apply only if client needs them)

- **item_variant**: replace ` - ` with `/` (e.g., `"P - Azul"` → `"P/Azul"`)
- **item_name**: strip variant suffix if appended (e.g., `"Camiseta - P"` → `"Camiseta"`)
- **item_brand**: proper case (e.g., `"RESERVA"` → `"Reserva"`)

These are client-specific. Only add if the original container applies similar normalizations.
