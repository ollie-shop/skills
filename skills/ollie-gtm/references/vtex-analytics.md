# VTEX Analytics

Centralized knowledge about VTEX analytics, dataLayer events, and GTM integration.

---

## List of native cart/checkout event types

Events pushed automatically by the VTEX IO GTM app (`vtex.google-tag-manager`).
Source: [vtex-apps/google-tag-manager](https://github.com/vtex-apps/google-tag-manager/blob/master/react/modules/enhancedEcommerceEvents.ts).

### Action-scoped events (Enhanced Ecommerce)

Fired on user actions. Format: UA Enhanced Ecommerce (`ecommerce.*`).

| Event name | Trigger | EE path |
|---|---|---|
| `productDetail` | PDP loads (from `vtex:productView`) | `ecommerce.detail.products[]` |
| `productClick` | User clicks product in list | `ecommerce.click.products[]` |
| `productImpression` | Products displayed in listing | `ecommerce.impressions[]` |
| `addToCart` | Item added to cart | `ecommerce.add.products[]` |
| `removeFromCart` | Item removed from cart | `ecommerce.remove.products[]` |
| `checkout` | Cart page loads (from `vtex:cartLoaded`) | `ecommerce.checkout.products[]` (step: 1) |
| `orderPlaced` | Purchase completed | `ecommerce.purchase.actionField` + `products[]` |
| `promoView` | Promotional content displayed | `ecommerce.promoView.promotions[]` |
| `promotionClick` | User clicks promotion | `ecommerce.promoClick.promotions[]` |

### Checkout step events

Fired during VTEX checkout flow progression.

| Event name | When |
|---|---|
| `cart` | Cart loaded |
| `profile` | Profile/email step |
| `email` | Email identification |
| `shipping` | Shipping step |
| `payment` | Payment step |

### Product data fields (UA Enhanced Ecommerce)

Fields available in `products[]` arrays:

| Field | Description |
|---|---|
| `id` | SKU ID |
| `name` | Product name |
| `brand` | Brand name |
| `category` | Category path |
| `variant` | SKU variant |
| `price` | Unit price |
| `quantity` | Quantity (where applicable) |
| `position` | List position (impressions/clicks) |
| `dimension1` | Product reference ID |
| `dimension2` | SKU reference ID |
| `dimension3` | SKU name |
| `dimension4` | Product availability (on productDetail) |

### orderPlaced extra fields

The `orderPlaced` event pushes additional root-level fields:

| Field | Description |
|---|---|
| `transactionId` | Order ID |
| `transactionTotal` | Order total |
| `transactionShipping` | Shipping cost |
| `transactionTax` | Tax amount |
| `transactionCurrency` | Currency code |
| `transactionPaymentType` | Payment method array |
| `transactionShippingMethod` | Shipping method array |
| `transactionProducts` | Products array |
| `transactionPayment.installments` | Installment count |
| `visitorContactInfo` | Array: [email, firstName, lastName, phone] |
| `visitorAddressDetail` | Shipping address fields |

### Notes

- VTEX uses **UA Enhanced Ecommerce** format, not GA4. The `ecommerceV2` duplicate object is also pushed on some events.
- The `orderPlaced` event fires on the order confirmation page (`/checkout/orderPlaced`), not during checkout.
- Checkout step events (`cart`, `profile`, `shipping`, `payment`) are used by tags like Konduto to identify the current checkout phase.

---

## VTEX Analytics Module (FastStore / VTEX IO)

### List of native event types (GA4 model)

The module provides built-in types based on [GA4 ecommerce events](https://support.google.com/analytics/answer/14434488?hl=en):

| Event type | Parameters |
|---|---|
| `add_payment_info` | `currency`, `value`, `coupon`, `payment_type`, `items` |
| `add_shipping_info` | `currency`, `value`, `coupon`, `shipping_tier`, `items` |
| `add_to_cart` | `currency`, `value`, `items` |
| `add_to_wishlist` | `currency`, `value`, `items` |
| `begin_checkout` | `currency`, `value`, `coupon`, `items` |
| `login` | `method` |
| `page_view` | `page_title`, `page_location`, `send_page_view` |
| `purchase` | `currency`, `transaction_id`, `value`, `affiliation`, `coupon`, `shipping`, `tax`, `items` |
| `refund` | `currency`, `transaction_id`, `value`, `affiliation`, `coupon`, `shipping`, `tax`, `items` |
| `remove_from_cart` | `currency`, `value`, `items` |
| `search` | `search_term` |
| `select_item` | `item_list_id`, `item_list_name`, `items` |
| `select_promotion` | `item_list_id`, `item_list_name`, `items` |
| `share` | `method`, `content_type`, `item_id` |
| `signup` | `method`, `content_type`, `item_id` |
| `view_cart` | `currency`, `value`, `items` |
| `view_item` | `currency`, `value`, `items` |
| `view_item_list` | `item_list_id`, `item_list_name`, `items` |
| `view_promotion` | `items` |
