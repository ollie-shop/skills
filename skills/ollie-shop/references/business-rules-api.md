# Business Rules API Reference

> **PLACEHOLDER** -- Replace with the actual internal API documentation for querying business rules.

## Overview

Ollie Shop maintains an internal API for documented business rules. These rules describe merchant-specific checkout behaviors that custom components may need to implement.

<!-- TODO: Document the following -->

## API Base URL
```
<!-- TODO: Add base URL -->
```

## Authentication
<!-- TODO: How to authenticate with the business rules API -->

## Endpoints

### List Rules for a Store
```
GET /api/stores/{storeId}/rules
```
<!-- TODO: Response schema, filtering, pagination -->

### Get Rule Details
```
GET /api/rules/{ruleId}
```
<!-- TODO: Response schema, including conditions, actions, and metadata -->

### Search Rules by Component/Slot
```
GET /api/stores/{storeId}/rules?slot={slotName}&component={componentName}
```
<!-- TODO: Document search/filter capabilities -->

## Rule Schema

```json
{
  "id": "rule-123",
  "name": "Free shipping threshold",
  "description": "Show free shipping badge when cart total exceeds threshold",
  "store_id": "store-456",
  "conditions": [
    { "field": "cart.total", "operator": "gte", "value": 100 }
  ],
  "actions": [
    { "type": "show_element", "target": "free_shipping_badge" }
  ],
  "slot": "cart_summary_footer",
  "platform": "vtex",
  "active": true
}
```

<!-- TODO: Document full schema, condition operators, action types -->
