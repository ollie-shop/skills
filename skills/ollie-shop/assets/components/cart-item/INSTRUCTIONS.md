# Cart Item

## What it is

A single row in the cart's item list. Renders one product with image, name, SKU info, quantity controls, price, and remove action. The template iterates the cart and renders this slot once per item.

## Props the slot injects (this is the entire contract)

> **The slot passes exactly these six props. Nothing else.** Everything else you need — price extras, prescription/subscription state, addons, platform-specific fields, pending-action flags, promotional labels, lab-discount eligibility — **must be derived inside the component** from `rawSession`, `useCheckoutSession()`, `usePendingActions()`, or other hooks. Adding props like `prescription`, `subscription`, `labDiscount`, `unitPriceCents`, or `pendingQuantity` to the component signature is the single most common mistake; the slot does not pass them and they arrive `undefined` in production.

| Prop | Type | Notes |
|---|---|---|
| `item` | `CartItem` | The parsed cart item: `id`, `uniqueId`, `index`, `name`, `quantity`, `price` (cents), `originalPrice` (cents), `available`, `image`, `url`. |
| `isUnavailable` | `boolean` | `true` when the host is rendering the row inside the unavailable group. Hide quantity controls and the price block; show a muted variant — or pass-through to `children` if you have no custom unavailable design. |
| `onQuantityChange` | `(quantity: number) => void` | Emit when the user changes quantity. The slot wires this to the parent's dispatch using `item.uniqueId`. |
| `onRemoveItem` | `(uniqueIdOrIndex: string \| number) => void` | Emit when the user clicks remove. The parent accepts either the unique line id or the cart index; passing `item.uniqueId` is the safest default. |
| `formatPrice` | `(priceCents: number) => string` | Currency formatter already wired to the session's locale and currency. Use it for **every** price you render in this row — do not build your own `Intl.NumberFormat`. |
| `children` | `ReactNode` | The platform's default render for the row. Use it as a fallback when you don't want to re-implement a particular state — most commonly the unavailable variant. |

## Slot(s)

- **Primary:** `cart_item` — the canonical per-item slot. The template iterates the cart and renders this slot once per item.

## Behavior

1. Render the product image (clickable, goes to PDP via `item.url`).
2. Render product name as a link to the PDP. Open in a new tab with `rel="noreferrer"`.
3. Render SKU info under the name (e.g. `SKU: <id>`).
4. Quantity controls (only when `!isUnavailable`): hold the typed value in optimistic local state so the input feels instant. On change and on blur, clamp to a minimum of `1` and call `onQuantityChange(value)`.
5. Price block (only when `!isUnavailable`): show unit price with strike-through original price when discounted, plus line total. Render every value through the `formatPrice` prop — never call `Intl.NumberFormat` directly.
6. Remove button: on click, call `onRemoveItem(item.uniqueId)`. The parent also accepts `item.index`, but `uniqueId` is stable across reorders and is the safer default.

## States to cover

- **Available** — full controls, normal price.
- **Unavailable** (`isUnavailable === true`) — host renders the row inside the unavailable group. Two acceptable shapes:
  - **No custom unavailable design** → `return <>{children}</>` and let the platform's default render handle this state. Choose this when the brief is about styling the available state only.
  - **Custom unavailable design** → hide quantity controls and the price block; show a muted variant of the row (image + name + SKU + remove only).
- **Discounted** — strike-through original price beside the discounted price.

## Visual structure (semantic only)

- Two-column row on desktop: image on the left, info block on the right.
- Inside the info block: name (link) on top, SKU below, controls + price aligned to the right edge.
- Quantity controls: `<button>` decrement, `<input type="number">` with `min="1"`, `<button>` increment. All same line.
- On mobile, the row collapses to: image at the top-left, name + SKU on the top-right, controls + price as a separate row below the info block.
- Remove control as a `<button>` (icon + text). Don't rely on icon alone.

## SDK touchpoints

- `useCheckoutSession()` — read `session.locale` for currency and locale formatting. Drop down to `rawSession` only when you need a platform-specific extra (e.g. attachments, fulfillment hints) that isn't on `session.cartItems[]`.
- Quantity and remove **do not** dispatch SDK actions from inside this component. The parent that uses `cart_item` is responsible for calling `useCheckoutAction("UPDATE_ITEMS_QUANTITY")` / `useCheckoutAction("REMOVE_ITEMS")` after receiving the callback.

## Without a design reference

- Two-column flex on desktop (`display: flex; gap: ...`); single column on mobile via media query.
- Image wrapped in a fixed-aspect container so layout doesn't shift while the image loads.
- Name uses `<a target="_blank" rel="noreferrer">`. Quantity controls are real `<button>` and `<input type="number">`.
- Strike-through for original price via `<s>` (semantic), not a `text-decoration: line-through` div.
- Use CSS variables for colors (`--color-fg`, `--color-fg-muted`, `--color-primary`). No hardcoded brand colors.
- Hover/focus visible states on every interactive element.

## Antipatterns

- **Don't dispatch checkout actions from inside the component.** Emit `onQuantityChange` / `onRemoveItem` and let the parent decide. Otherwise the row can't be reused in other lists (summaries, unavailable groups, etc.).
- **Don't read the cart from `session.cartItems` to render this row.** The slot passes you `item`; using the array directly causes double-render and bugs when items reorder.
- **Don't build your own `Intl.NumberFormat` for price display.** The slot already injects `formatPrice` wired to the session's locale and currency. Use it for every price you render. Re-implementing the formatter (especially with hardcoded `"en-US"` / `"USD"`) breaks multi-region stores and risks drifting from the host's formatting decisions (currency symbol position, decimals, grouping).
- **Don't write quantity directly to the SDK on every keystroke.** Keep an optimistic local value, debounce or commit on blur — otherwise the user can't type "12" without "1" firing first.

---

# Pharma extensions (optional)

When the store is a pharmacy, the cart row anchors several regulated or commercial features that the default cart doesn't have. Each one is independent — pick the ones the pharmacy supports and skip the rest. They stack inside the same `cart_item` slot, below the price/controls row.

This whole section is opt-in. If the request is for a generic cart row, ignore everything below the line.

## When to load this section

Load when the user's request, the store metadata, or the platform payload mentions any of: prescription requirement, controlled medication, lab discount program (any operator name), continuous-use medication, recurring subscription tied to a SKU, patient identification at checkout. If none of those are present, the base section above is enough.

## Setup applies to every extension

Each extension below is a custom client integration (Case B in the design flow). Before generating code for any of them, run **one batch** of clarifying questions covering only the extensions the user wants — do not ask them per extension:

1. **Where does the eligibility signal live for this SKU?** It is not always the same place even on the same platform. On VTEX it is *often* an `attachmentOfferings` entry, but it can also be a category, a SKU specification, a price-list teaser, a tag, a side API call, or store config. Ask the pharmacy explicitly which field/source carries the signal and how to read it. On other platforms (Shopify / VNDA / custom) the same question applies — get the exact key, source, and expected value.
2. **Where is the "applied" state stored?** Usually it mirrors the eligibility source — on VTEX, often a populated `attachments[name=…]` whose content shape you need to pin down. On other platforms, where does the per-line marker live?
3. **What endpoints back the mutation?** HAR, docs, auth scheme, identity model. The cross-cutting design-flow file owns the wider Case B checklist; don't duplicate it — just collect the specifics for this extension.
4. **Are there per-extension constraints we need to surface in UI?** Examples: a file-size or MIME-type rule for an upload, a CPF/RG/national-id mask, a cadence enum the catalog publishes, a copy variant the pharmacy's compliance team requires. Capture them as props/config — never bake them in as constants.

## State sources (general rule)

- `session.cartItems[index]` — the standardized snapshot. Use it for everything cross-platform: id, quantity, prices, availability.
- `rawSession.items[index]` — the platform-specific extras. Pharma extensions almost always need this for eligibility/applied predicates when the signal lives in the platform payload. Index by `item.index` and verify the resolved raw item's id matches `item.id` — the index can drift when items reorder mid-render, and a stale index leaks another line's state into this row.
- Same-SKU duplicates are real (two cart lines with the same medication and different prescriptions, for example). Key any per-line state (modals, drafts, polling loops) by a composite cart-line key (`${item.index}-${rawItem.uniqueId}` or the platform equivalent), not by SKU.

## Extension A — Prescription attach

### Experience

For SKUs the pharmacy flags as prescription-required, the row carries an inline bar below the price/controls block. Two states:

- **Pending** — alert-level bar telling the customer a prescription is required. CTA opens a modal that collects whatever the pharmacy's compliance flow needs (a file, optionally some metadata).
- **Attached** — info-level bar acknowledging the prescription is on file. CTA reopens the modal in edit mode so the customer can revise.

The bar itself does not call the prescription API. It opens a modal owned by the pharmacy's compliance flow and forwards `onAddPrescription` / `onEditPrescription` to the orchestrator.

### Eligibility & applied predicates

- **Eligible:** the line carries the pharmacy's prescription-required marker. The location of this marker varies — get it from the setup questions, don't assume it.
- **Applied:** the same line has the pharmacy's prescription-attached marker (often a non-empty attachment, sometimes a separate flag). Treat it as a boolean computed from whatever source the pharmacy declared.

### What to ask the pharmacy

- The exact source and key for the eligibility signal, and for the applied signal.
- If the attached state has a content payload (schema), the field keys and types. There may be no schema at all — the attachment might be just the file. Don't presume one shape.
- Whether any field in the attached payload can be surfaced on the bar to help the customer distinguish one attached prescription from another (a holder name, a prescription number, a date, anything). If the answer is "no field like that", the attached-state bar shows a generic acknowledgement without an identifier.
- The accepted document types and any size cap. Pass both as props or store config.
- Whether the prescription needs validation against an external compliance API before it's considered "attached," or whether file presence is enough. If validation is involved, treat it as part of the Case B integration checklist for this extension.

### States to cover

- **Pending attach** — alert bar with a primary CTA.
- **Attached** — info bar. If the schema has an identifier field, show it; otherwise show a generic acknowledgement.
- **Editing** — modal open; the bar's CTAs are disabled to prevent double-open.
- **Mutation pending** — disabled and `aria-busy`.

### Antipatterns

- **Don't gate visibility on `item.available`.** Prescription-required items are often the reason a pack is unavailable; the bar must still render so the customer can resolve the block. The pack-level unavailable component should defer to this row.
- **Don't hardcode the schema fields.** The pharmacy's compliance form is theirs. Take the schema as input and render the fields the schema lists; don't assume any specific field exists.
- **Don't bake the file-size or MIME-type rules into the component.** They are policy and they vary per pharmacy and per country. Take them as props.
- **Don't reset the modal's local form state on every parent re-render.** Mount the modal with a composite cart-line key so React unmounts/remounts it per cart line — otherwise editing line B inherits line A's typed values.
- **Don't autosubmit on close.** Closing without explicit save leaves the prior attachment intact. The file binary is only captured in this modal.

## Extension B — Lab discount program trigger

### Experience

Some SKUs are enrolled in a lab-funded discount program (the patient identifies themselves, the operator authorizes a price, the cart line is repriced). When the line is eligible and not yet applied, the row shows a small promotional bar below the price block — generic copy referencing "the program" rather than a specific operator, plus a primary CTA that opens the program's enrollment flow.

After the flow finishes, the line's unit price is replaced with the authorized price and an inline tag on the price block acknowledges the source of the discount. The trigger bar hides.

The orchestrator owns the flow — typically a patient-identifier modal, then a branching modal that walks the customer through terms acceptance or registration (often by deep-linking to the operator's site), then a polling loop that detects when enrollment completes, then a write that stamps the applied marker on the cart line.

### Eligibility & applied predicates

- **Eligible:** the SKU carries the pharmacy's lab-program marker. The location varies — it might be an attachment offering, a catalog flag, a tag, a teaser, or a side API result. Get it from the setup questions, treat it as a boolean computed from whatever source the pharmacy declared.
- **Applied:** the line has the pharmacy's "discount applied" marker. Often this is a populated attachment whose content names the authorized price, the operator transaction id, and any other fields the pharmacy declared.

### Price preview rule

The trigger label should show the best price the program could authorize, picking from these sources in priority order, falling back when one isn't available:

1. The authorized price from a real call to the operator (requires the patient identifier — only available after at least one program flow this session, or when the identifier is already on the customer record).
2. A catalog-driven estimate using whatever percentage hint the catalog publishes, clamped to the current selling price as a floor so the teaser never undercuts a promo that's already cheaper.
3. The selling price itself (no surprise discount shown until you know better).

### What to ask the pharmacy

- The exact source and key for the eligibility signal and the applied signal.
- If the catalog publishes a "minimum discount %" or equivalent hint that lets you preview a price without calling the operator, where it lives.
- The endpoint surface for the operator: discovery (does this patient need to accept terms / register / are they already enrolled?), price quote, and the write that stamps the applied marker.
- The patient identifier the operator uses (national tax id, customer id, anything) and where it can be reused from session state without re-prompting.
- The program identifier(s) and any per-store config (operator-specific store id, contract id, etc.). These belong in store config or props, never in the component.
- The expected state transitions for the discovery endpoint and the polling cadence the operator's team recommends.

### States to cover

- **Eligible, idle** — trigger bar visible, label uses the best price source available.
- **Eligible, pre-fetching** — trigger bar visible but disabled while the discovery or quote call is mid-flight.
- **Applied** — trigger bar hidden; price block shows the authorized price plus the inline tag.
- **Polling for enrollment** — the customer was taken to the operator's site in a new tab. The component polls discovery on a cadence the orchestrator owns; the trigger is disabled until polling resolves.
- **Polling cancelled** — modal closed, polling stopped; trigger returns to idle.

### Antipatterns

- **Don't bake operator names, program ids, or store contract ids into the component.** Take them as props or read them from store config. A second pharmacy will have different operators.
- **Don't trust marketing teasers as the eligibility signal unless the pharmacy explicitly says that's the source.** Teasers are copy; if a more authoritative source exists (attachment offering, catalog flag, API), prefer it.
- **Don't re-open the modal on every polling tick.** Once the customer is on the operator's site, polling runs in the background; only a state flip updates UI.
- **Don't re-ask the patient identifier when the cart already knows it.** Reuse it silently from the customer record or a prior flow's saved value.
- **Don't open the operator's deep link more than once per flow.** Multiple clicks should re-trigger discovery, not stack new tabs.

## Extension C — Recurring subscription bar

### Experience

For SKUs sold on a recurring cadence, the row shows a compact bar below the price block: a toggle to subscribe this line plus a cadence picker. Toggling on attaches the subscription to the cart line; toggling off removes it; changing the cadence updates the attachment in place.

The bar is presentation-only. The orchestrator dispatches the attach/remove/update calls and flips a `pending` prop while the call is in flight.

### Eligibility & applied predicates

- **Eligible:** the line carries the pharmacy's subscription marker, plus a non-empty cadence enum the customer can pick from. The marker's location varies — confirm in setup.
- **Active:** the line has the pharmacy's "subscription on" marker (often a populated attachment whose content names the chosen cadence). Mirror the cadence into the picker.

### Cadence rendering rule

- The picker's option values are exactly what the catalog enum publishes. Pass them through unchanged when writing back — whitespace, casing, and unit suffixes are part of the platform's enum.
- The picker's option labels are human-facing and should be humanized at render time. Take a `formatCadence(value) => label` function as a prop (or default to a passthrough) so the pharmacy can localise.

### What to ask the pharmacy

- The exact source and key for the eligibility and active signals, and where the cadence enum is published.
- The endpoint surface for attach / remove / update.
- Whether the pharmacy expects a default cadence when the customer toggles on without picking (usually the enum's first entry).

### States to cover

- **Eligible, off** — toggle off, picker shows the default cadence.
- **Eligible, on** — toggle on, picker shows the current cadence.
- **Pending** — disabled and `aria-busy`.
- **Empty cadence enum** — hide the bar entirely; there's nothing to attach.

### Antipatterns

- **Don't dispatch the attach/remove call from inside the bar.** Emit `onToggle(active)` / `onCadenceChange(next)` and let the orchestrator decide. Same rule as the base cart-item.
- **Don't normalize the raw cadence value before writing back.** Trimming, lowercasing, or reformatting a string the platform's enum expects byte-for-byte triggers a validation failure server-side.
- **Don't hardcode cadence labels.** They are localisation; pass them in.

## Composition: how the extensions stack inside one row

Place the extensions below the price/controls block, stacked top-down. Use this default ordering and only override it when the pharmacy's design says otherwise:

1. Lab discount trigger (when eligible and not applied).
2. Recurring subscription bar (when eligible).
3. Prescription bar (when required).

Rationale: prescription is the most regulatorily charged signal and sits closest to the boundary that gates checkout. Subscription is a commitment signal that benefits from sitting where the discount conversation just happened. The lab trigger is the most promotional and sits highest because it can change the price the customer is about to read.

When the row is unavailable (`isUnavailable === true`), render `children` and skip every extension. The unavailable variant is owned by a separate orchestrator (a pack-level slot, not the per-item slot).

## Antipatterns specific to pharma layering

- **Don't duplicate eligibility predicates across sub-components.** Compute each predicate once at the top of the row and pass booleans/objects down.
- **Don't render any extension on an unavailable row.** The bar UX assumes the line can still be acted on; the unavailable variant is owned at the pack level.
- **Don't bake operator, schema, copy, or limit values into the component.** Every value that varies between pharmacies is a prop or config input. Anything that varies between countries (mask, MIME types, allowed sizes) is also a prop.
- **Don't share modal state across same-SKU cart lines.** Key modals by the composite cart-line key. Two prescriptions for the same medication are a real scenario.
- **Don't introduce a fourth extension here without a real second pharmacy needing it.** Single-deployment patterns belong in the customer's project, not in the shared library.
