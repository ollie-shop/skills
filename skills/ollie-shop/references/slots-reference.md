# Slots Reference

How the Ollie Shop slot system works: concepts, lifecycle, visibility rules, constraints, and the canonical slot registry.

## Table of Contents
1. [How Slots Work](#how-slots-work)
2. [CustomComponent Schema](#customcomponent-schema)
3. [Slot Registry](#slot-registry)
4. [Slot Lifecycle](#slot-lifecycle)
5. [Visibility Rules](#visibility-rules)
6. [Constraints and Notes](#constraints-and-notes)

---

## How Slots Work

The `<Slot>` component is the universal UI injection point in Ollie Shop templates.

**Default behavior**: if no custom component is registered for a slot, it renders its `children` prop (the default template UI).

**Custom component behavior**: if `storeInfo.components[]` contains an entry with `slot === slotId`, the default `children` are replaced with the remote component.

Remote components are loaded as ESM modules and have access to:
```ts
import * as React from "react";
import * as OllieShopReact from "@ollie-shop/sdk";
import * as NextNavigation from "next/navigation";
import * as NextLink from "next/link";
import * as NextImage from "next/image";
import * as NextItl from "next-intl";
```

---

## CustomComponent Schema

```ts
{
  id: string            // unique component instance ID
  name: string          // human-readable name
  props: Record<string, unknown>  // static props merged with slot runtime props
  slot: string          // must match the Slot id
  url: {
    js: string          // URL to the compiled ESM bundle
    css?: string        // optional stylesheet URL
  }
}
```

### Props Passing

Props passed to `<Slot id="..." extra={value}>` are forwarded directly to the remote component. The slot's `children` are also forwarded as `children`.

### Children: Replace vs Augment

A custom component always **receives** the slot's default UI as `children`. What it does with them is a deliberate design choice:

| Strategy | What the component does | When to use |
|---|---|---|
| **Replace** | Ignores `children` and renders its own UI from scratch | The component owns the entire slot concern (e.g. a coupon input that fully substitutes the native coupon form). |
| **Augment** | Renders `{children}` plus extra UI around/below | You want to keep the native behavior and add on top (e.g. a free-shipping progress badge above the native totalizer). **Default choice** when in doubt — preserves native UX. |

Rule of thumb: if omitting the custom component would leave a gap in the UX, render `{children}`. If the custom component is the UX, don't.

### Only one custom component per slot

`storeInfo.components[]` is resolved by finding a single entry with `slot === slotId`. There is no stacking, ordering, or fallback chain. If two requirements target the same slot, they must be **consolidated into a single component** — see `agents/oliver-slot-resolver.md` for the canonical consolidation rules:

1. Both requirements must share the same `mount_phase` (cross-phase consolidation is forbidden).
2. The UIs must be complementary, not mutually exclusive — if both want to fully replace the slot with incompatible UIs, surface the conflict for user decision instead of silently merging.
3. If consolidation isn't possible, move one requirement to a narrower or adjacent slot (see the YAML for alternatives).

---

## Slot Registry

All slot IDs, lifecycle phases, visibility behavior, context props, and shared contexts are defined in:

> **`assets/checkout-slots-data.yaml`**

Load this file to validate slot IDs and understand rendering behavior. **Do not resolve slots from memory** — every slot ID you emit must exist as an `id` entry in that file.

### Registry Schema

Each entry in the YAML has these fields:

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Unique slot identifier |
| `step` | `string` | Checkout phase: `cart`, `shipping`, `address`, `payment`, `confirmation`, `global`, `on-action` |
| `mount_phase` | `string` | When the slot enters the React tree (see Slot Lifecycle) |
| `visibility` | `string` | `always` or `conditional` |
| `visibility_type` | `string` | Fine-grained visibility category (see Visibility Rules) |
| `visibility_trigger` | `string \| null` | Exact source condition for conditional slots |
| `device_constraint` | `string \| null` | `mobile-only`, `desktop-only`, or `null` |
| `default_content` | `string` | What renders by default when no custom component is registered |
| `context_props` | `string[]` | Props passed to the slot at runtime |
| `shared_contexts` | `string[]` | Files/pages where this slot ID appears simultaneously |
| `notes` | `string \| null` | Additional architectural notes |

---

## Slot Lifecycle

### mount_phase

Each slot has a `mount_phase` that determines when its custom component enters the React tree. Use this to reason about execution order and component separation.

| `mount_phase` | Meaning |
|---|---|
| `global` | Mounts once at checkout init, before any step is rendered. Stays mounted across all steps. |
| `step:cart` | Mounts only when the cart page is active. |
| `step:shipping` | Mounts only when the shipping step is active. |
| `step:address` | Mounts only when the address step is active. |
| `step:payment` | Mounts only when the payment step is active. |
| `step:confirmation` | Mounts only on the order confirmation page. |
| `on-action` | Rendered conditionally in response to a user action or async event (modals, overlays). |

**Key rule:** Two rules targeting slots with different `mount_phase` values MUST produce separate components. Never consolidate across mount phases — a `global` component and a `step:payment` component have fundamentally different lifecycles and responsibilities.

---

## Visibility Rules

`mount_phase` tells you which step a slot belongs to. It does not guarantee the slot is always present when that step is active. Slots have a `visibility_type` field that captures sub-behavior:

| `visibility_type` | Meaning |
|---|---|
| `always` | Present as soon as the parent step/page is active. |
| `user-interaction` | Only mounts when the user performs an explicit action (e.g. clicking a tab). |
| `data-state` | Conditionally rendered based on runtime data (e.g. saved cards exist, packages loaded). |
| `device` | Always in the DOM but scoped to a specific breakpoint (mobile-only or desktop-only). |

Before selecting a slot, check its `visibility_type` in the YAML. If `user-interaction` or `data-state`, verify the component's availability requirements match the spec intent. Prefer an `always` slot when the component must be immediately visible.

### Payment Option Slots

All `payment_option_*` slots (except `payment_option_gift_card`) have `visibility_type: user-interaction`. They do not exist in the DOM until the user actively clicks the corresponding payment method tab. **Never place a primary CTA, confirm button, or any element that must be immediately visible at a `payment_option_*` slot.** Use `payment_method_options` instead when the component must be present as soon as the payment step renders.

---

## Constraints and Notes

1. **Slot IDs are global strings** — two `<Slot id="checkout_coupon_form">` in different components resolve to the same registered component from `storeInfo.components[]`. Check `shared_contexts` in the YAML for slots that render in multiple locations simultaneously.

2. **Error boundaries** — every `<Slot>` is wrapped in an `ErrorBoundary`. In Studio mode, errors show a detailed debug UI; in production, a generic fallback message is shown. Errors never crash the parent tree.

3. **Skeleton fallback** — remote components show a `skeleton h-auto min-h-8` div while loading (`Suspense` fallback in `RemoteComponent`).

4. **`hint` prop** — passing `hint={true}` wraps the slot content in `<script data-slot="slot:start:<id>">` / `<script data-slot="slot:end:<id>">` markers. Used by the Studio editor to locate slot boundaries in the DOM.

5. **Custom step pages** — when a custom step's `page` string is used as a slot id (e.g. `<Slot id="LoyaltyStep" prev={...} next={...} />`), the `prev` and `next` navigation props are passed directly to the remote component.

6. **Device-gated defaults vs slot rendering** — some slots always render but their default children carry responsive CSS (`md:hidden`, `lg:hidden`). A custom component injected into these slots does **not** inherit the responsive class. Check the `notes` field in the YAML for affected slots (e.g. `checkout_totalizer`, `cart_totalizer`).
