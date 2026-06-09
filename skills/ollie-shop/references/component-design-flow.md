# Component & Function Design Flow

Cross-cutting workflow the agent runs **before** loading the entry-specific `INSTRUCTIONS.md`. Covers the questions and decisions that apply to any custom component or hub function, regardless of which one is being built.

## When to use this file

Load this file when the user wants to **create or change** a checkout component or a hub function. Examples that should trigger it:

- "I want to customize the cart item row."
- "Create a component from this Figma."
- "Write a function that validates the prescription before adding to cart."
- "Build a free shipping bar."

Do **not** load this file when the user is asking:

- "Which hooks does the SDK expose?" → `sdk-guide.md`.
- "What CLI command lists business rules?" → `cli-reference.md`.
- "Which slot is `payment_option_pix`?" → `slots-catalog.md`.

## Question hierarchy — read this first

Three rules govern the order of questions:

1. **Cross-cutting first** — Figma MCP availability, native SDK capability vs. custom integration, design reference quality. All resolved in this file.
2. **Entry-specific second** — load the target entry's `INSTRUCTIONS.md` (`assets/components/<id>/INSTRUCTIONS.md` or `assets/functions/<id>/INSTRUCTIONS.md`) for questions that are specific to that pattern.
3. **Never repeat** — skip any question whose answer is already in the user's initial prompt or in a previous step of this flow.

The goal is to produce one short batch of clarifying questions and then commit to the implementation, not a back-and-forth interrogation.

---

## Step 0: Context inventory

Before anything else — design references, library lookups, slot resolution — take stock of what the user actually gave you. The single most common failure mode is generating code with inferences when the user could have answered in one message. If critical context is missing, **stop and ask once, in batch**, before writing anything.

### What to check

Run through this checklist mentally on the first message of the conversation:

| Artifact | Why it matters | What "missing" looks like |
|---|---|---|
| **Design reference** | Pixel/spacing/color decisions | No Figma link, no screenshot, no "use the default neutral style" instruction. |
| **HAR file** (Case B integrations) | Real headers, payloads, status codes, sequencing | Only "we call our backend" without a recording. |
| **API docs or payload sample** (Case B) | Field names, shape, success/error variants | "Just call the endpoint" without showing the request/response. |
| **Auth scheme** (Case B) | Bearer? Signed header? Cookie? mTLS? | Not mentioned, or "use my token" without showing where the token lives. |
| **Business rule source** | Resolves ambiguity when a single signal can mean two things | E.g. "show the lab-discount when eligible" — but the eligibility flag could live in the catalog property, the orderForm, or a custom API. |
| **Failure / fallback policy** | Decides fail-open vs fail-closed for any external call | Not addressed when the request can time out. |

### The pause-and-ask rule

If anything **critical for the work** is missing, do not start coding. Ask in **one** batch, plain and specific:

> "Before I start, I need to confirm a few things:
> 1. Do you have a Figma file or a screenshot for this component?
> 2. The flow that adds the prescription — do you have a HAR of it on the existing site?
> 3. What's the source of truth for whether a product needs a prescription — a catalog property, a field on the orderForm, or an external API?"

After the user answers, commit and execute. **Do not loop**. If you find a new ambiguity mid-implementation, that is its own one-off question — but the upfront batch should cover 80%+ of cases.

### The no-deduction rule

If the user gave you an artifact (HAR, payload sample, source file) and you find yourself uncertain about a specific field, **re-consult the artifact** before guessing. The pattern that costs the most iterations is: agent skim-reads, picks the first interpretation that fits, hard-codes it. Examples observed in practice:

- Reading `openTextField.value` from a sample rawSession when the same store actually uses `customData.customApps` — both shapes exist in VTEX, only one is populated per store.
- Assuming `modalType === "CHEMICALS"` means "PBM-eligible" when the real signal is the `ProgramaPBM` catalog property.
- Replicating an HTTP call client-side because the final state in rawSession implies it was made — when in reality a server-side hub function applied it.

If the artifact does not answer the question definitively, **that is also a question** — ask the user before assuming.

### When the user clearly gave you everything

If the inventory passes (Figma is attached, the HAR is there, the business rule has a canonical source), skip the pause-and-ask and move on to Step 1. The proactive ask is for the case where the user said "build me X" with no supporting material — not for every conversation.

---

## Step 1: Identify the design reference

How visual fidelity will be achieved. Pick whichever the user has and adapt — do not block the work on a missing reference.

### Figma + Figma MCP (recommended)

If the user mentions Figma at any point, **recommend installing the Figma MCP** before going further. With MCP the agent reads the design structure directly (frames, auto-layout, components, tokens), which is dramatically more reliable than screenshots — colors, spacing, and typography come from the design rather than visual estimation.

Brief setup hint to give the user:

- Install the official Figma Dev Mode MCP server. The Figma desktop app must be running with the file open. Add the MCP to the agent's tool config.
- Once installed, the agent can call Figma MCP tools to inspect frames by URL or selection.

**Once MCP is installed, prefer `get_design_context` over `get_screenshot` for layout work.** `get_design_context` returns the structured tree — frames, auto-layout rules, components, design tokens — which is what you actually need to build markup. `get_screenshot` (or downloading PNGs) only gives a flat image and forces you to eyeball colors and spacing. Use the screenshot tool only for a visual diff or to confirm a hover/focus/empty state that's not in the inspected frame.

If the user doesn't want to install the MCP, fall back to screenshots — but flag that the agent will be guessing at hover/focus/responsive variants.

### Screenshots

Acceptable but with three caveats the agent must surface back to the user:

- Hover, focus, active, and disabled states are usually not in screenshots — ask explicitly.
- Error and empty states are usually not in screenshots — ask explicitly.
- Mobile vs. desktop layouts may not both be present — ask which viewports matter and whether either has a screenshot.

### Description-only

The user describes the component in prose. Workable, but the agent has to be more aggressive about asking for state coverage (idle, loading, error, success, empty). Use the matching `INSTRUCTIONS.md`'s "States to cover" section as the checklist.

### No reference at all

Fall through to the entry's `INSTRUCTIONS.md` section called **"Without a design reference"**. That section ships with a neutral semantic markup default that uses the host's CSS variables (`--color-primary`, `--surface`, etc.), so the component works visually without committing to a brand identity. The user can paint over it later.

---

## Step 2: Native SDK capability or custom client integration?

Almost every customization request falls into one of two cases. Determining which one upfront prevents the agent from either inventing fake SDK hooks (Case A) or trying to solve a problem without enough information (Case B).

### Case A — Native SDK capability

The behavior the user wants is reachable from the Ollie SDK alone — one of the 8 hooks or 14 `useCheckoutAction` action names (see `sdk-guide.md`).

Signals:

- "Add product to cart" → `useCheckoutAction("ADD_ITEMS")`.
- "Show the customer email" → `useCheckoutSession().session.customer.email`.
- "Open the login modal" → `useLogin().openLogin()`.
- "Apply a coupon" → `useCheckoutAction("UPDATE_COUPONS")`.
- "React to the order being created" → `useCheckoutOrder()`.

What to do: **no extra cross-cutting questions**. Move straight to the entry's `INSTRUCTIONS.md` and only ask what that file calls out as specific.

### Case B — Custom client integration

The behavior touches a system the Ollie SDK doesn't expose: the client's own backend, a third-party API, a webhook, an internal database, a SaaS the merchant pays for separately. The Ollie SDK has no hook for this.

Signals:

- "Upload an image to **my** S3 bucket."
- "Validate the prescription against **our** compliance API."
- "Pull store credit balance from **our** loyalty system."
- "Send a webhook to **our** ERP when the order is placed."
- "Call **our** authentication endpoint before the customer can checkout."

What to ask before coding (one batch — do not loop):

1. **Documentation of the route or API.** A link to the partner's docs, OpenAPI spec, internal wiki page, or a description of the contract.
2. **HAR file** capturing the working flow on the client's existing site (or in Postman/Insomnia). This is the single most useful artifact because it carries headers, payloads, status codes, and timing in one place.
3. **Payload sample** — at least one successful request body and the corresponding response body.
4. **Auth scheme** — bearer token, signed header (which algorithm), cookie, mTLS, basic auth. If credentials are needed at runtime, where do they live (env var, secret manager, hub-config)?
5. **Failure behavior the partner expects** — retry policy, idempotency, rate limits, what status codes mean what. Spending one question here saves an outage later.

If the user can only provide some of these, proceed with what they have and flag the gaps explicitly. Don't block on perfection.

---

## Step 3: Template gap analysis

Before committing to a custom component, check whether the default template already does what the user wants. Load `references/templates/default.md` and compare:

| Outcome | Action |
|---|---|
| Template already does it natively | Push back gently: "the default template already handles X — do you want to *change* the default behavior or are you OK with what's there?" |
| Template does it partially | The custom component **extends** the default; identify which behavior you're keeping vs. replacing. |
| Template does nothing | Full custom component. Confirm the target slot from `slots-catalog.md`. |

For hub functions, the equivalent question is: does the platform (VTEX/Shopify/etc.) already enforce this rule out of the box? If yes, skip the function — let the platform handle it.

---

## Step 4: Match against the library

Once the design reference and the integration mode are settled, look up the library:

- Components: `assets/components.csv`.
- Functions: `assets/functions.csv`.

Match by description, then by slot/trigger. Three possible outcomes:

- **Exact match** — load that entry's `INSTRUCTIONS.md` and follow it. The library entry already encodes the opinionated default, the antipatterns, and (when applicable) any specific questions.
- **Partial match** — closest entry is similar but not identical. Read its `INSTRUCTIONS.md` as a starting point; deviate from the parts that don't apply.
- **No match** — build from scratch using the patterns documented in `component-anatomy.md` (components) or `hub-functions.md` (functions). Surface this to the user: "no existing pattern fits — I'll build from scratch, here's the plan."

---

## Step 5: Resolve the slot or trigger

For components, resolve the target slot from `references/slots-catalog.md`. If the slot id has a `{{ variable }}` segment, resolve the variable to the concrete value before passing to `ollieshop component create --slot <id>` (e.g. `payment_option_pix`, never `payment_option_{{ paymentMethodName }}`).

For functions, decide the `trigger.url` and `trigger.expression`:

- `trigger.url`: the exact endpoint that should fire the function. Get it from the HAR or platform docs.
- `trigger.expression`: a JSONata-like match expression, typically narrowing by method. Example: `method in ["POST"]`. Chain with `and` to add header/body/status filters.

If you can't pin down a stable slot or trigger, that's a blocker — surface it before continuing.

---

## Step 6: Hand off to the entry's `INSTRUCTIONS.md`

Final step before generating code. With the cross-cutting context resolved, load the target entry's `INSTRUCTIONS.md` and follow it:

- Component: `assets/components/<id>/INSTRUCTIONS.md`.
- Function: `assets/functions/<id>/INSTRUCTIONS.md`.

The entry tells you the behavior contract, states to cover, semantic structure, SDK touchpoints, the "without a design reference" fallback, and the antipatterns specific to that pattern. **Skip any question whose answer is already known from this flow** (Figma availability, Case A vs B, slot id, etc.) — those don't get re-asked at the entry level.

If there's no matching entry, fall back to:

- `references/component-anatomy.md` for component layout and dev loop.
- `references/sdk-guide.md` for the hooks.
- `references/hub-functions.md` for handler shape and trigger configuration.

---

## Common cross-cutting pitfalls

- **Suggesting the Figma MCP only after the agent has already produced code from a screenshot.** Ask up front, when the user first mentions Figma.
- **Confusing Case A with Case B.** "Show free shipping when cart > $X" sounds custom but is Case A (`useCheckoutSession` + a threshold prop). Re-check signals before asking for a HAR.
- **Asking for design specifics on a component the user wants to ship with a neutral default.** If they say "I just want it to work, I'll style it later", go straight to the "Without a design reference" path in the entry's `INSTRUCTIONS.md`.
- **Forgetting Step 3.** Building a custom component for behavior the default template already does is the most common waste of effort.
- **Replicating an HTTP call that does not appear in the HAR.** If the final rawSession shows a state transition (price change, attachment added, customData populated) and no client-side call in the HAR explains it, the change was made by a server-side hub function. Document the inference but do **not** add the call to the component "just to be safe" — it will be redundant when the hook works and harmful when the orderForm config rejects it (403). Trust `revalidate: true` on the last blocking request to pick up the side effects.
- **Confusing fields with similar names without re-reading the sample.** A single store usually uses **one** of `customData.customApps` vs `openTextField.value` (not both); the eligibility signal for PBM is `ProgramaPBM` (catalog property), not `modalType` (orderForm field). When two field names sound plausible for the same data, **re-read the sample** before picking — the cost of one extra glance is much less than the cost of an iteration of correction.
