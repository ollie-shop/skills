# Design judgement for custom checkout work

The requirements a custom component or hub function is held to, and the
evaluation it passes before anyone calls it done. Load this **before proposing a
layout** and **again before reporting the work complete**.

You are working on **one merchant's checkout**, inside a hosted checkout that
Ollie owns. You build slot components and hub functions; you never edit the
native template. That sets the bar for everything below: room to be specific to
this store, no licence to break what the shopper already relies on.

These are **requirements, not suggestions**, and each states its exception in the
same breath. The default is good enough to follow without thinking; the way out
is named so you never have to break a rule quietly.

| You are... | Load |
| --- | --- |
| Specifying states, loading, failure, undo, motion | `states.md` |
| Deciding modal vs inline vs step; placing a control; naming scope | `hierarchy.md` |
| Reading a flow, a Figma file or a screen for problems | `screens.md` |
| Writing or reviewing any string the shopper reads | `copy.md` |
| Starting on a store with no `checkout-context.md` | `context-template.md` |

The pixel rules — tokens, CSS modules, a11y mechanics — live in
`../design-contract.md`. This folder governs behaviour and hierarchy; that one
governs pixels.

## Before starting: two questions

Answer both from the request and the repo, state the answers in one line, and
continue. Do not ask the user what the repo already answers.

**1. Is this store new to you?** If the project root has no
`checkout-context.md`, you are working blind on who the shopper is. Say so in one
line, and offer to create the file from `context-template.md` once the work is
done. Never invent a persona to fill the gap. Analytics say where shoppers leave,
never who they are.

**2. Is it a component or a function?**

- **A slot component** changes what appears on screen. Build it against this file.
- **A hub function** changes what the checkout asks, returns, or enforces. **It is
  design work too:** a response function that reshapes a payload changes what the
  shopper sees, and a request function that rejects changes what they can do.
  Before building one, write the shopper's situation in one line per branch
  ("a returning customer whose saved address is now outside the delivery area").
  Each line becomes a case the function must survive.

**The guardrail.** If the design needs native behaviour changed — a slot that
does not exist, a step in a different order, a rule the template enforces —
**stop and say so.** That is a request to Ollie, not a workaround to build. A
component that fights the template around it breaks on the next native release.

## The three principles, ranked

When two rules conflict, the lower-ranked principle yields.

### 1. Never surprise anyone about money or consequence

The person is deciding whether to hand over money. Every surprise is withdrawn
from the same account.

- **A control's appearance must predict its consequence.** Two controls that look
  alike must do alike. A text link that relocates an entire order is the most
  expensive kind of lie, because it costs nothing to click.
- **A cost that exists must be visible, or explicitly deferred.** "Calculated at
  the next step" is honest. A total that quietly includes a charge the page says
  is unknown is not.
- **Never make an irreversible action look like navigation.** *Exception:* an
  action that is genuinely one click to undo may skip a confirmation, but the
  undo must be visible in the same place, immediately.

### 2. Remove work, not information

Friction reduction that costs clarity is a trade, not a win. Say which side you
took and why.

- **Cut fields and steps; do not cut the facts someone needs to decide.** Fewer
  fields is usually right and always costs someone downstream data.
- **Only show a control when it can do something.** *Exception:* when its absence
  would read as an omission, replace it with a muted, non-interactive label naming
  the constraint, not a dead control.
- **Do not occlude the data a decision depends on.** If the decision is about a
  price, the price stays on screen while it is made. This is the most common
  reason a modal is the wrong container.

### 3. The happy path is a minority of sessions

- **Design every state, not the default one.** See `states.md`.
- **Walk every branch the slot can land in.** See *Branches* below.
- **Real content is not mock content.** Ask what the longest realistic value is —
  a four-line address complement, a 90-character product name, a cart of 40
  items — and check the layout survives it.

## Replacing a slot: you inherit nothing

A slot component receives the native default as `children`. **Render `children`
and you augment; ignore it and you replace.** Replacing is often right, and it
always means everything the native component handled is now yours. The mechanics
— the callback signature the runtime actually passes, and the legacy type that
does not match it — are in `../sdk-guide.md` §Component Interface.

Before replacing, open the native component in the running checkout and list
what it does in each of these, then build each one or say why it is not needed:

- **Loading** — what shows while its data is in flight.
- **Empty and error** — what shows when there is nothing, or the call failed.
- **Focus and keyboard** — tab order, visible focus, what Escape does.
- **Accessible names** — labels, roles, live announcements of changes.
- **Every branch it renders differently in** — see *Branches* below.
- **The props it received** — what the slot passes in, and which of those carry
  behaviour (`isPending`, `onToggle`, and the like) rather than data.

*Exception:* augmenting — rendering `children` and adding beside it — keeps all
of that for free. Prefer it whenever the design allows. A replacement that drops
the loading state looks finished in Studio and broken on a slow phone.

## Branches

The same slot renders in more situations than the one in the Figma file. Before
calling a component done, check it in each that applies to this store:

- Guest and returning (identified, with saved data pre-filled)
- Delivery, pickup, and a mix of both in one order
- One package and several
- An item unavailable or out of stock
- An empty cart, and a 40-item cart
- A coupon applied, and a coupon rejected
- Every payment method the store offers, including the slow ones
- Every viewport, and every locale the store sells in

If a branch cannot be reached in this store, say which and why rather than
skipping it silently.

## Where a string lives

Follow the copy rule in `../component-authoring-cheatsheet.md`: use the host's
translations when it provides them, otherwise expose each string as an
overridable prop with a pt-BR default. Never bury a shopper-facing string where
the merchant cannot change it without a code change.

If the store sells in more than one locale, every new string exists in every one
of them. Wording itself: `copy.md`.

## Decide, don't ask

Whoever reads your output is usually about to build, and an open question is a
block on their work.

- **Lead with one recommendation.** Where a choice is genuinely live, give at
  most three, labelled **Recommended** / Alternative / Rejected, with one line on
  why each alternative lost. Never a menu of equal weight.
- **Ask only when a default would be unsafe or unknowable:** money, legal copy, a
  merchant rule you cannot see, or a branch that changes the shape of the work.
  Everything else gets a default, marked as one, so it can be overruled on
  purpose.
- **A question you can answer by reading is not a question.** Read the code, the
  Figma file, `checkout-context.md`, and the running checkout first.
- *Exception — explore mode.* When someone is still forming the idea ("should
  we", "what if"), argue both sides and withhold the answer. Entered **on
  request**, never by default.

## When you are given a reference to copy

"Build it like <store>" is an instruction about **arrangement**, not a licence to
redesign. Reorder, re-token and re-word; keep the components.

- **Moving a control between steps is a redesign, not a copy.** It changes what
  someone knows at the moment they decide. Raise it; never do it silently.
- *Exception:* when the reference's arrangement is impossible with the slots that
  exist, name the slot that blocks it and stop. That is the guardrail above.

**A client's Figma is a proposal, not a specification.** It usually carries their
visual language and often surfaces the checkout does not have. Keep what is worth
keeping, adapted to the slots that exist; say plainly what is not buildable.

## The evaluation — before calling anything done

Run this before reporting any component or function complete, however small the
change. It is not a conversation: **ask the user nothing.** Check what was built
against this file and report what fails. The small changes are exactly where it
gets skipped.

1. **Look at it running.** In Studio or the live preview, not in the diff. Reading
   code is not verification. *Exception:* when there is nothing to run — no
   deployed version, a placeholder `storeId` — say so in the first line of the
   report, check what the code can tell you, and label the result unverified. A
   code-only pass is never reported as a verified one.
2. **Walk the branches** above that apply, and the states in `states.md`.
3. **Check it on the merchant's tokens, not Ollie's defaults** — especially the
   states that carry meaning (selected, disabled, error) against the merchant's
   colour ramp.
4. **Check it against the store's real catalogue**, not fixtures: real product
   names, real address shapes, real quantities.
5. **Keep a finding only if you can name how it costs completed orders, and would
   still raise it if you could only raise three.** Cite the rule it breaks by
   name. "Competes with the primary action" is a finding; "feels cluttered" is
   not.
6. **Report:** what fails, the fix, and what the fix costs, ranked, at most three.
   Fix what is small and inside the agreed scope; report the rest. If nothing
   fails, say so in one line and finish.

### Done when

- Walked on **both viewports**, and in **every locale the store sells in**, with
  every new string present in each.
- Every **branch** that applies exercised, not only the happy path.
- On the **merchant's tokens**, checked in the states that carry meaning.
- Against the **store's real catalogue**.
- **Degrades rather than breaks** when the slot gains props it does not use:
  ignoring the contract costs the feature, never the page.
- **Handles the platform's failures, not only its successes.** Hook results arrive
  undefined before they arrive populated, and an empty string is not an absent
  value. See `../sdk-guide.md`.
- **Asked whether it belongs in native.** If the fix would be true of every
  merchant, say so in the handoff. The third time the same custom fix gets built,
  it was a native gap.

## Every recommendation states its cost

Not hedging. It is what lets someone overrule the guidance on purpose rather than
by accident. If you cannot name the cost, you have not finished thinking.

| Change | Costs |
| --- | --- |
| Fewer form fields | Data someone downstream wanted |
| Pre-selecting the recommended option | Fewer people compare; some overpay |
| Commit-on-select instead of a save | No undo; every click is a round trip |
| Replacing a slot instead of augmenting | Every behaviour the native one had is now yours |
| Collapsing an entry point | The people who used it lose the fast path |

## What this folder does not cover

- **Tokens, CSS, a11y mechanics** — `../design-contract.md`.
- **SDK hooks and their gotchas** — `../sdk-guide.md`.
- **Which slot to use** — `../slots-catalog.md`.
