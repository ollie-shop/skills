# GTM JSON Structure Reference

JSON structures for creating GTM entities programmatically. Use these templates when generating container JSON.

---

## Tag Structure

```json
{
  "accountId": "...",
  "containerId": "...",
  "tagId": "...",
  "name": "[Ollie] GA4 - purchase",
  "type": "gaawe",
  "parameter": [...],
  "firingTriggerId": ["TRIGGER_ID"]
}
```

Tags are created **active** by default (no `"paused"` field). Only set `"paused": true` if the user explicitly requests it.

## Trigger Structure

```json
{
  "accountId": "...",
  "containerId": "...",
  "triggerId": "...",
  "name": "[Ollie] purchase Trigger",
  "type": "CUSTOM_EVENT",
  "customEventFilter": [
    {
      "type": "EQUALS",
      "parameter": [
        { "type": "TEMPLATE", "key": "arg0", "value": "{{_event}}" },
        { "type": "TEMPLATE", "key": "arg1", "value": "purchase" }
      ]
    }
  ],
  "filter": [
    {
      "type": "CONTAINS",
      "parameter": [
        { "type": "TEMPLATE", "key": "arg0", "value": "{{Ollie Checkout Provider}}" },
        { "type": "TEMPLATE", "key": "arg1", "value": "ollie" }
      ]
    }
  ]
}
```

## DLV Variable Structure

```json
{
  "accountId": "...",
  "containerId": "...",
  "variableId": "...",
  "name": "[Ollie] DLV - GA4 - transaction_id",
  "type": "v",
  "parameter": [
    { "type": "INTEGER", "key": "dataLayerVersion", "value": "2" },
    { "type": "BOOLEAN", "key": "setDefaultValue", "value": "false" },
    { "type": "TEMPLATE", "key": "name", "value": "ecommerce.transaction_id" }
  ]
}
```

## Cookie Variable Structure (Ollie Checkout Provider)

```json
{
  "name": "Ollie Checkout Provider",
  "type": "k",
  "parameter": [
    { "type": "BOOLEAN", "key": "decodeCookie", "value": "false" },
    { "type": "TEMPLATE", "key": "name", "value": "checkout_provider" }
  ]
}
```

## Negated Filter (does not contain)

GTM JSON does NOT have a `DOES_NOT_CONTAIN` type. Use `CONTAINS` with `"negate": true`:

```json
{
  "type": "CONTAINS",
  "negate": true,
  "parameter": [
    { "type": "TEMPLATE", "key": "arg0", "value": "{{Page Path}}" },
    { "type": "TEMPLATE", "key": "arg1", "value": "orderPlaced" }
  ]
}
```
