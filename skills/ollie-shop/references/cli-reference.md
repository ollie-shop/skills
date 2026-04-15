# Ollie Shop CLI Reference

> **PLACEHOLDER** -- Replace this content with the actual CLI documentation from the ollie-shop CLI.

## Table of Contents
1. [Authentication](#authentication)
2. [Store Management](#store-management)
3. [Component Management](#component-management)
4. [Version Management](#version-management)
5. [Development](#development)
6. [Deployment](#deployment)
7. [Business Rule Management](#business-rule-management)

---

## Authentication

```bash
# Login to Ollie Shop
ollie login

# Check current session
ollie whoami

# Logout
ollie logout
```

<!-- TODO: Document auth flow, token storage, session refresh -->

---

## Store Management

```bash
# List stores
ollie store list

# Create a store
ollie store create --name <store-name> --platform <platform>

# Get store details
ollie store get <store-id>

# Update store config
ollie store update <store-id> --config <key>=<value>

# Delete a store
ollie store delete <store-id>
```

<!-- TODO: Document store config options, platform-specific settings -->

---

## Component Management

```bash
# List components in a store
ollie component list --store <store-id>

# Create a new component
ollie component create --name <name> --slot <slot-name> --store <store-id>

# Get component details
ollie component get <component-id>

# Update component metadata
ollie component update <component-id> --name <new-name>

# Delete a component
ollie component delete <component-id>
```

<!-- TODO: Document component metadata, slot assignment rules -->

---

## Version Management

```bash
# List versions of a component
ollie component version list --component <component-id>

# Create a new version
ollie component version create --component <component-id> --path <source-path>

# Activate a version
ollie component version activate <version-id>

# Rollback to a previous version
ollie component version rollback <component-id> --to <version-id>
```

<!-- TODO: Document versioning strategy, activation flow, rollback behavior -->

---

## Development

```bash
# Start local development server
ollie dev --store <store-id>

# Bootstrap a new ollie-shop project
ollie init --template <template-name>

# Validate component against slot contract
ollie validate --component <path>
```

<!-- TODO: Document dev server features, hot reload, template selection -->

---

## Deployment

```bash
# Deploy a component version
ollie deploy --component <component-id> --version <version-id>

# Deploy all pending versions
ollie deploy --all --store <store-id>

# Check deployment status
ollie deploy status <deployment-id>
```

<!-- TODO: Document deployment pipeline, rollback, environment targeting -->

---

## Business Rule Management

Business rules are scoped to a **store** and can optionally reference specific versions via `versions_ids`.

```bash
# List all business rules for a store
ollieshop business-rule list --store-id <uuid>

# List business rules linked to a specific version
ollieshop business-rule list --store-id <uuid> --version-id <uuid>

# List only rules where code was updated
ollieshop business-rule list --store-id <uuid> --code-updated true

# Get a single business rule
ollieshop business-rule get --id <uuid>

# Update a business rule (flags)
ollieshop business-rule update --id <uuid> --content "new rule content"

# Update with linked versions, components, and functions
ollieshop business-rule update --id <uuid> --content "rule" \
  --versions-ids '["<uuid>"]' \
  --components-ids '["<uuid>"]' \
  --functions-ids '["<uuid>"]'

# Update via JSON
ollieshop business-rule update --data '{"id":"<uuid>","content":"new rule","versionsIds":["<uuid>"]}'
```
