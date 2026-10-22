# Per-screen question bank

Prompts for your attention while reading a flow. **Not an output template.** Most
answers here are minor and get discarded — only findings that survive step 5 of
the evaluation in `design-judgement.md` make it into the report.

**These are prompts, not rules.** The rules live in `design-judgement.md` and its siblings — this file
only points your attention at where they tend to break. Cite the rule, not the
prompt.

Where a well-known pattern is invoked, its cost is named alongside it. Anything
without a named cost is a question, not a recommendation.

Many of these are **behavioural** — when validation fires, what happens on failure,
whether data persists. Reviewing a Figma flow, they cannot be answered from the
frames. Carry them forward as questions for the build rather than guessing, and note
where a state simply hasn't been designed yet.

---

## Cart / Bag

- Is "Proceed to checkout" the single dominant action, or does it compete with
  "Continue shopping", coupon entry, and cross-sells at similar visual weight?
- Is the subtotal visible without scrolling?
- Are shipping and taxes shown, or clearly flagged as calculated later? Unexpected
  cost at the end is the most consistently documented abandonment cause (Baymard).
- Can quantity be changed and items removed inline, without leaving the page?
- If the cart can be empty, is there a recovery path?
- **Promo code field**: a prominent empty coupon box makes people without a code
  feel they're overpaying, and some leave to search for one. Collapsing it reduces
  that — at the cost of making legitimate coupon holders hunt, which generates
  support contacts. Which side costs more depends on how many codes are actually
  in circulation. Check before recommending.

## Identification

- Field count: what is genuinely required to create the order, versus wanted?
- For a returning or identified user, what is pre-filled — and is it obvious that it
  was recognised, or does the form just appear mysteriously full?
- Is account creation forced before purchase, or optional and deferred?
- Does validation fire on blur or only on submit? Are messages specific enough to
  act on ("CPF must have 11 digits") rather than generic ("invalid field")?
- Tax ID: correct local term and input mask for the store's market (CPF, RFC, DNI)?

## Address

- Is entry split postal-code-first, with the rest auto-filled? Reduces perceived
  effort; costs an extra round trip, so the lookup's latency and failure state
  matter as much as the pattern itself.
- What happens when the lookup fails or returns nothing — is there a manual path,
  or a dead end?
- Is undeliverable-to-this-postcode surfaced *before* the full address is typed?
- Do field labels and order match local convention? (Brazil: Rua, Número,
  Complemento, Bairro, Cidade, Estado — not "Address Line 1 / Line 2".)
- For returning users, can a saved address be chosen in one tap? What happens when
  two saved addresses are near-identical?

## Shipping / SLA options

- Is an option pre-selected? Pre-selecting the recommended one removes a decision —
  at the cost that fewer people compare, and some pay for speed they didn't need.
  Pre-selecting the cheapest inverts both.
- Are options scannable without expanding anything — carrier, window, price all
  visible at once?
- Are estimates shown as dates ("chega 12–14 de agosto") or durations ("5–8 dias
  úteis")? Dates are concrete and generally read faster; they also expose weekend
  and holiday maths, so they are only better if the underlying calculation is right.
- If a free-shipping threshold exists, is the remaining amount stated?
- Is pickup, where offered, legible as the free/fast option rather than buried?
- How many options are shown? Past roughly four, comparison cost rises faster than
  the value of the extra choice.

## Payment

- Is the locally dominant method first (Pix in Brazil, OXXO in Mexico, iDEAL in NL)?
- Are trust signals at the point of card entry, not only in the footer?
- Does the card field detect brand, format the number, advance focus?
- Where installments are expected, is per-installment pricing and interest clear?
- **Failure recovery**: on a declined card, is the message specific, and is every
  other field preserved? This is the single most expensive moment in the flow —
  the person has already decided to buy.
- Are wallet/express options offered before the long form, not after it?

## Review / Summary

- Can address, shipping, and payment each be changed without restarting?
- Is the cost breakdown fully itemised — subtotal, shipping, discounts, taxes, total?
- Are product thumbnails present, so the order can be visually confirmed?
- Does the final button state the amount ("Pagar R$ 189,90") or a generic word
  ("Confirmar")? Stating the amount removes ambiguity about what pressing it does;
  it also puts the number one last time in front of someone who might reconsider.
- Are required consents separated from marketing opt-in, and is marketing unchecked?

## Confirmation

- Is there an unmistakable success state with an order number?
- Does the person know when it arrives and how to track it?
- Is there one gentle next action, or a wall of competing ones?
- Is the confirmation email mentioned, with the address it went to?

---

## Cross-flow

- **Step count** — count the real branches, not the happy path. Fewer steps usually
  means less drop-off, but consolidating into one long screen trades step-count for
  cognitive load and a longer error surface. Neither is free.
- **Progress indication** — is position and remaining length visible, and does it
  persist across every step?
- **Layout consistency** — same structure and CTA placement on every step?
- **Persistent summary** — can the person see what they're paying for throughout?
- **Exit path** — can they get back to the store without losing the cart?
- **Mobile** — reasonable scroll depth per step, and the correct keyboard per input
  type (numeric for CEP and card). Thumb reach, tap-target size, and data
  persistence are requirements, not prompts — `design-judgement.md`.
