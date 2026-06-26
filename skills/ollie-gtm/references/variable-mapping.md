# Variable Mapping — Ollie GTM

Rules and patterns for creating GTM variables that read from Ollie's data sources.

---

## Data Layer Variables (DLV)

Ollie's data layer follows GA4 ecommerce standard. All DLV variables use `dataLayerVersion: 2` and read from `ecommerce.*`:

| DLV Path | Description |
|---|---|
| `ecommerce.transaction_id` | Order ID |
| `ecommerce.value` | Total order value |
| `ecommerce.currency` | Currency code (e.g., BRL) |
| `ecommerce.coupon` | Coupon code |
| `ecommerce.shipping` | Shipping cost |
| `ecommerce.tax` | Tax amount |
| `ecommerce.items` | Array of GA4 item objects |
| `ecommerce.payment_type` | Payment method |
| `ecommerce.shipping_tier` | Shipping method name |

**No transformation needed** — the data layer already comes in GA4 format.

## GA4 Item Fields

Items in `ecommerce.items` use GA4 field names:

| Field | Description |
|---|---|
| `item_id` | Product/SKU ID |
| `item_name` | Product name |
| `item_brand` | Brand |
| `item_category` | Primary category |
| `item_category2` ... `item_category5` | Category hierarchy |
| `item_variant` | Variant (size, color) |
| `price` | Unit price |
| `quantity` | Quantity |
| `discount` | Discount amount |

## JavaScript Variables

When the data layer doesn't provide a needed field, read from `__RAW_CHECKOUT_SESSION__`:

```js
// Pattern: always guard with existence check
function() {
  if (typeof __RAW_CHECKOUT_SESSION__ !== 'undefined' && __RAW_CHECKOUT_SESSION__) {
    return __RAW_CHECKOUT_SESSION__.clientProfileData.email;
  }
  return undefined;
}
```

Common fields from `__RAW_CHECKOUT_SESSION__`:

| Path | Usage |
|---|---|
| `.clientProfileData.email` | User email |
| `.clientProfileData.phone` | User phone |
| `.clientProfileData.userProfileId` | User ID |
| `.items` | Raw items array (for enrichment) |
| `.items[].productCategories` | Category hierarchy object |
| `.items[].productRefId` | Product reference ID |
| `.marketingData.coupon` | Coupon code |
| `.totalizers` | Array of totals (Items, Shipping, Discounts) |

## Custom JS Variables for Third-Party Platforms

Each ad/analytics platform expects a different data shape. Create dedicated CJS variables that map from the Ollie data layer items to the platform's format:

```js
// Example: mapping items to a comma-separated ID list
function() {
  var items = {{[Ollie] DLV - GA4 - items}};
  if (!items) return '';
  return items.map(function(item) { return item.item_id; }).join(',');
}
```

For complete platform-specific CJS patterns (AWIN, RTB House, Pinterest, etc.), see `third-party-cjs-patterns.md`.

## Variable Reference Mapping

When adapting parameters from an existing container, replace every `{{variable}}` reference:

| Pattern | Ollie equivalent |
|---|---|
| `{{dlv - transactionId}}` | `{{[Ollie] DLV - GA4 - transaction_id}}` |
| `{{dlv - transactionTotal}}` | `{{[Ollie] DLV - GA4 - value}}` |
| `{{dlv - currency}}` | `{{[Ollie] DLV - GA4 - currency}}` |
| `{{JS - items}}` | `{{[Ollie] DLV - GA4 - items}}` |
| `{{JS - coupon}}` | `{{[Ollie] DLV - GA4 - coupon}}` |
