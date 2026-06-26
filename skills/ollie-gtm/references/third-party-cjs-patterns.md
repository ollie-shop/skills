# Third-Party Platform CJS Variable Patterns

Each ad/analytics platform expects a specific data shape. Below are common patterns for creating CJS variables that map from Ollie's GA4 items.

## Comma-Separated Product IDs

Used by: **Blue**, **RTB House**, **generic retargeting**

```js
function() {
  var items = {{[Ollie] DLV - GA4 - items}};
  if (!items || !items.length) return '';
  return items.map(function(item) { return item.item_id; }).join(',');
}
```

## Product ID Array

Used by: **RTB House** (offerIds), **Pinterest**

```js
function() {
  var items = {{[Ollie] DLV - GA4 - items}};
  if (!items || !items.length) return [];
  return items.map(function(item) { return item.item_id; });
}
```

## AWIN Product Tracking Array

Used by: **AWIN** (AW:P format)

```js
function() {
  var items = {{[Ollie] DLV - GA4 - items}};
  if (!items || !items.length) return [];
  return items.map(function(item) {
    return {
      id: item.item_id,
      price: item.price,
      quantity: item.quantity,
      name: item.item_name,
      category: item.item_category || ''
    };
  });
}
```

## AdvCake Product Array

Used by: **AdvCake**

```js
function() {
  var items = {{[Ollie] DLV - GA4 - items}};
  if (!items || !items.length) return [];
  return items.map(function(item) {
    return {
      id: item.item_id,
      name: item.item_name,
      categoryName: item.item_category || ''
    };
  });
}
```

## Total Minus Shipping

Used by: **AWIN** (commission base), **affiliate platforms**

```js
function() {
  var value = {{[Ollie] DLV - GA4 - value}};
  if (!value) return 0;

  var raw = (typeof __RAW_CHECKOUT_SESSION__ !== 'undefined') ? __RAW_CHECKOUT_SESSION__ : null;
  if (!raw || !raw.totalizers) return value;

  var shippingTotal = 0;
  for (var i = 0; i < raw.totalizers.length; i++) {
    if (raw.totalizers[i].id === 'Shipping') {
      shippingTotal = raw.totalizers[i].value / 100;
      break;
    }
  }
  return value - shippingTotal;
}
```

## Deduplication Cookie Check

Used by: **RTB House**, **affiliate attribution**

```js
function() {
  var cookieValue = {{cookie - _source_cookie_}};
  // Returns true if NOT attributed to the platform (for deduplication)
  return cookieValue !== 'platform_name';
}
```

## Category Split (Comma-Separated)

Used by: **generic retargeting**, **audience segmentation**

```js
function() {
  var items = {{[Ollie] DLV - GA4 - items}};
  if (!items || !items.length) return '';
  var categories = [];
  items.forEach(function(item) {
    if (item.item_category && categories.indexOf(item.item_category) === -1) {
      categories.push(item.item_category);
    }
  });
  return categories.join(',');
}
```
