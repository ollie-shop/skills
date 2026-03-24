---
name: ollie-shop
description: >
  Build, manage, and deploy custom checkout components for the Ollie Shop e-commerce platform.
  Use this skill whenever the user is working with ollie-shop, ollie checkout, ollie CLI,
  custom checkout components, checkout slots, checkout templates, or mentions deploying/managing
  components on the Ollie Shop platform. Also trigger when the user asks about building checkout
  UIs, integrating with e-commerce checkout flows (especially VTEX), or needs help understanding
  how to implement business rules in a custom checkout component. If the user mentions figma designs,
  screenshots, or business rules for a checkout flow, this skill helps translate those into
  ollie-shop components.
---

# Ollie Shop - Custom Checkout Components

You are helping a developer build, manage, and deploy custom checkout components for the **Ollie Shop** platform -- an agnostic, extensible checkout system that connects with multiple e-commerce platforms (currently VTEX).

Ollie Shop allows developers to:
- Select checkout **templates** (pre-built checkout layouts)
- Upload custom **components** that plug into specific **slots** within those templates
- Manage stores, components, and versions via a **CLI** and **SDK**

Your job is to guide developers through the full lifecycle: from understanding what they need to build, to coding it with the SDK, to deploying it via the CLI.

---

## How This Skill Is Organized

This skill has four modes of operation. Identify which mode the developer needs based on their request:

### 1. CLI Operations
When the developer needs to **manage resources** (login, deploy, CRUD for stores/components/versions).
- Read `references/cli-reference.md` for the full command reference.

### 2. SDK & Coding Guidance
When the developer needs to **write component code** using the Ollie Shop SDK.
- Read `references/sdk-guide.md` for SDK API, patterns, and best practices.

### 3. Component Library
When the developer needs **inspiration or reference implementations** for common checkout components.
- Read `assets/component-library.csv` for the catalog of reusable patterns.
- Each entry includes: component name, description, suggested slot, file path, tags, and complexity.
- Use this to suggest existing patterns before building from scratch.

### 4. Component Design Flow
When the developer needs help **going from a design/requirement to an implementation plan**.
- Read `references/component-design-flow.md` for the guided workflow.
- This flow helps translate Figma designs, screenshots, documented business rules, or existing website behavior into a structured implementation plan.

---

## General Principles

### Architecture
- Components are **self-contained UI units** that render in specific template slots.
- Each component receives context from the checkout SDK (cart data, user session, store config).
- Components communicate with the checkout orchestrator via SDK hooks -- never directly with the e-commerce platform API.
- The SDK provides an **anti-corruption layer** between your component and the underlying platform (VTEX, etc.).

### Coding Standards
- Use **TypeScript** for all components.
- Follow the SDK's component interface -- every component exports a default function matching the slot contract.
- Keep components **stateless where possible**; use SDK-provided state management for checkout state.
- Handle loading and error states explicitly -- checkout UX demands resilience.
- Respect the template's design tokens for consistent styling.

### Slot System
Templates define **named slots** where custom components can be placed. Each slot has:
- A **name** (e.g., `cart_coupon`, `payment_method`, `order_summary_footer`)
- A **contract** (the props/context the slot provides to the component)
- **Constraints** (size limits, required behaviors, accessibility requirements)

When helping a developer choose a slot, consider:
1. What the component does functionally
2. Where it makes sense in the checkout flow (cart, shipping, payment, confirmation)
3. What data the component needs (available via the slot contract)

### Deployment Flow
1. Developer writes component code locally
2. Tests with `ollie dev` (local development server)
3. Creates a new version with `ollie component version create`
4. Deploys with `ollie deploy`
5. Activates in the target store

---

## Working with Business Rules

Checkout components often encode **business rules** specific to the merchant. When helping a developer implement business rules:

1. **Identify the rule source** -- Is it from a Figma file, a screenshot, a documented spec, or observed behavior on a live website?
2. **Decompose the rule** into conditions and actions (e.g., "if cart total > $100, show free shipping badge")
3. **Map to SDK capabilities** -- Which SDK hooks and data sources support this rule?
4. **Consider edge cases** -- Empty cart, logged-out user, unavailable product, currency formatting, mobile viewport
5. **Validate against the slot contract** -- Can this slot's context provide the data the rule needs?

For accessing documented business rules, read `references/business-rules-api.md` for the internal API reference.

---

## Figma & Design Integration

When the developer provides Figma files or screenshots as reference:

1. Use the **Figma MCP** (if available) to read design files and extract component structure, spacing, colors, and layout.
2. Map visual elements to **Ollie Shop slots** -- identify which parts of the design correspond to which template slots.
3. Identify **design tokens** from the template that should be used instead of hard-coded values.
4. Flag any design elements that **fall outside** what the slot contract supports -- these need discussion with the team.
5. Generate a component skeleton that matches the design's structure.

---

## Quick Reference

| Task | Where to Look |
|------|---------------|
| CLI commands (login, deploy, CRUD) | `references/cli-reference.md` |
| SDK API, hooks, component interface | `references/sdk-guide.md` |
| Reusable component patterns | `assets/component-library.csv` |
| Design-to-code workflow | `references/component-design-flow.md` |
| Business rules API | `references/business-rules-api.md` |
