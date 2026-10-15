# States and time

A design shows arrangement. Almost everything that goes wrong in checkout lives
in the seconds between arrangements.

## Every interactive thing needs five states

Default, hover, focus, active, disabled. Non-negotiable, and cheap to specify —
the expensive part is discovering at build time that nobody decided.

**Focus is a state, not an afterthought.** `:focus-visible` on everything
interactive, never `:focus` (it fires on mouse click and looks like a bug).
Never `outline: none` without a visible replacement. Someone navigating by
keyboard has no other way to know where they are.

**Disabled must say why, or not be disabled.** A greyed control with no
explanation is a dead end that looks like a bug. *Exception:* when the reason is
obvious from immediate context — a Save that enables once a field changes needs
no explanation, because the shopper just watched it happen.

## Every data-bearing thing needs four more

| State | When | What it must not do |
| --- | --- | --- |
| Loading | Data is in flight | Collapse the layout, or leave the old value looking current |
| Empty | Legitimately nothing | Look like an error, or offer no way forward |
| Error | The request failed | Blame the person, or lose what they typed |
| Partial | Some of it arrived | Present incomplete data as complete |

**A design with no loading state has an undesigned second or two in it.** That
second is where people leave. Postal-code lookup, shipping quotes, payment
authorisation — each is a real wait somebody has to look at.

**Skeleton the thing that will change, not the whole region.** A skeleton that
replaces more than the incoming value makes the page feel broken rather than
busy. That is the scope; the mechanics of building one — DOM budget, keyframes
outside CSS Modules, class naming — are in `../component-authoring-cheatsheet.md`
§6.

## Commit, undo, and failure

The three ways to handle a change, and what each obliges you to build:

| Pattern | Obliges you to | Good when |
| --- | --- | --- |
| Explicit save | A dirty state, a disabled-until-changed primary, a discard path | The change is multi-field, or expensive to get wrong |
| Commit on select | A pending state, a revert path, a visible undo | The change is one value and the result is immediately visible |
| Optimistic | All of the above plus a reconciliation story | Latency would otherwise dominate a frequent action |

**If you commit without a save, you owe a revert.** Specify it in the design, not
in the ticket. The failure case is: the UI already moved, then the server said
no. What the shopper sees at that moment is a design decision, and leaving it to
the builder means it becomes a toast that says "Error".

> *Cart SLA simulation — selecting a shipping option collapses the panel
> instantly while the save is still in flight. Without a specified revert, a
> failed quote leaves the row showing a choice the order does not have.*

**An error must contain a way forward.** If the message has no action in it, the
screen needs a button that does. An error that only describes the problem is a
dead end with punctuation.

**Never blame the person.** "We could not find that postcode", not "you entered
the wrong postcode" — the second is both unkinder and more often false, since the
postcode may be valid and simply missing from the database. Rewrites in
`copy.md`.

**Validate on blur, not only on submit.** Someone who fills six fields and learns
at the end that the second one was wrong has to rebuild their place in the form.
*Exception:* fields whose validity depends on another field can only be checked
once both are filled — validate those on the second blur, not the first.

**Never lose what someone typed.** Not on error, not on back-navigation, not on
refresh. This is broken more often than anything else in checkout and noticed
internally almost never, because internal testers do not make mistakes on forms
they wrote.

**A declined payment is the most expensive error in the flow.** The person has
already decided to buy. Give the real reason where the gateway provides one,
keep every field filled, and name the next-best method.

## Latency

Numbers so the rule is checkable, not a vibe:

- **Under ~100 ms** — feels instant. No indicator; adding one makes it feel slower.
- **100 ms – 1 s** — show something immediately. A skeleton or an inline spinner
  on the affected element, not a page-level overlay.
- **Over ~1 s** — say what is happening in words. "Calculating shipping" beats a
  spinner with no subject.
- **Over ~10 s** — you need a way out, and a way back in.

**Give a spinner a show-delay of ~150–300 ms and a minimum visible time of
~300–500 ms.** Without the delay, fast responses flash. Without the minimum, the
flash happens anyway when the response lands mid-render.

## Motion

**Animate the transition, never the data.** Prices, totals, quantities, and
validation errors change because a fact changed. A number that slides or fades is
a number people re-read to check it did not change again.

**Duration by job:** ~100 ms for hover and collapse, ~200 ms for state changes,
~300 ms for a modal. Easing: `ease-out` entering, `ease-in` leaving,
`ease-in-out` for a toggle.

**Every animation respects `prefers-reduced-motion`.** Not a nicety —
vestibular disorders are real and checkout is not the place to trigger one.

**Motion must not move the target.** If collapsing one region shifts the region
someone just tapped, correct the scroll in the same frame as the layout change.
On a phone this is the difference between "it opened" and "something else
happened".

> *Cart SLA simulation — opening one package group collapses another. When the
> collapsing group sits above, the row the shopper just touched jumps upward and
> can leave the viewport entirely.*
