# Hierarchy, scope, and containers

Where a control sits is a claim about what it affects. Most confusion in a
checkout is a control whose position claims one scope and whose behaviour has
another.

## Scope

**Let position carry scope, so the label does not have to.** In a two-column
layout, the item column is *this thing* and the summary column is *the order*.
Once that holds, a control's reach is legible before it is read, and you stop
needing labels like "Change shipping options for all packages".

> *Cart SLA simulation — seven entry points reached shipping options, running
> three different behaviours behind three different labels. Resolved not by
> renaming them but by cutting to two, placed so position said which was which.*

**One scope, one control.** If two controls do the same thing at the same scope,
delete one. If they look the same and do different things, that is worse than
duplication — see the appearance rule below.

**Adjacency beats labelling.** A control that changes one group belongs inside
that group, not in a toolbar above all of them.

## Appearance and consequence

**Two controls that look alike must do alike.** This is principle 1 restated
because it is the rule most often broken by accident: a component reused in a
second position quietly acquires a second meaning.

*Exception:* none. If the consequences differ, the appearance must differ —
different weight, different container, or different colour role. "It's obvious
from context" is what everyone says before the support ticket.

**Destructive actions get a text label.** An icon-only trash is a permanent
action with no name on it. *Exception:* an icon-only control is fine when the
action is trivially reversible and the icon is conventional — a bookmark, a
collapse chevron.

**Colour must not be the only carrier of meaning.** Red-and-amber to distinguish
"you must delete this" from "you can fix this" fails for roughly 8% of men, and
fails for everyone scanning quickly. Put the distinction in the words and the
available actions.

> *Cart SLA simulation — the unavailable cell and the prescription cell were the
> same shape with the same chevron, differing only in border colour, while one
> meant "remove this to continue" and the other meant "upload this and it
> joins your order".*

## Choosing a container

Ranked by cost to the shopper. Take the cheapest one that works.

| Container | Costs | Use when |
| --- | --- | --- |
| In place | Layout shift | The choice is small and the context matters |
| Inline expansion | Shift, plus state to manage | A few options, and the surrounding data informs the choice |
| Modal | Full occlusion, a dismiss decision | The sub-decision is heavy enough to need its own context |
| New step | A step in the funnel | The decision is genuinely sequential and gates what follows |

**A modal earns its place when the sub-decision is heavy enough to need its own
context.** Picking a store from a map, choosing a delivery date, entering an
address. A radio list of three options is not.

**A modal that covers the data the decision depends on is the wrong container.**
If someone is choosing between shipping prices, the total must stay visible.

*Exception — and it is the common one:* on a phone there is no room for anything
else, so a modal is right for decisions that would be inline on a desktop. That
is a viewport constraint, not a pattern preference. Say so explicitly when you
diverge, or the next person will "unify" it.

## Disclosure

**Show the number of hidden things, or do not hide them.** A carousel with pager
dots hides how much is left and lets someone act having seen one of three.
Vertical stacking reveals length by scrolling; a pager does not.

**A blocker is not a page in a sequence.** Anything that stops checkout —
unavailable items, a failed validation — belongs in a persistent region, not
behind a swipe or a tab. *Exception:* none worth taking.

**Expanded state must be visible from the collapsed state.** If a group can be
opened, the collapsed version has to say what is inside it — the current value,
not just a chevron. The collapsed summary and the expanded selection must
agree; a collapsed row naming one option while the panel highlights another is
the defect that survives review because both halves look right alone.

## Primary actions

**One dominant action per screen.** Everything else is visibly secondary. If two
things look equally primary, neither is.

**The verb must match the destination.** A button that says "Finish" and leads to
another step spends trust exactly where trust is scarce. Where the amount is
settled, state it — "Pay R$ 189,90" removes the last ambiguity about what the
press does, at the cost of putting the number in front of someone one final
time. That trade is usually worth taking; a client who disagrees has a
legitimate position, not an error.

**Keep the primary action reachable.** On mobile that means thumb reach and a
minimum 44×44 px target. *Exception:* a visual target may be smaller if its hit
area is expanded to meet the minimum.

## Labels and fields

**Every field keeps a visible label.** Above the field, or a floating label that
shrinks and stays — the platform's own input is the floating kind
(`../design-contract.md`), so that is a default here, not an exception. What is
never a label is a placeholder alone: it vanishes on focus, breaks for screen
readers, and makes reviewing a filled form impossible.

**Say why you need anything unusual.** A phone field with no explanation reads as
marketing capture. "So the courier can call you" reads as logistics. The rule
applies to any field whose purpose is not self-evident from the transaction.

**Mark whichever produces fewer marks.** If most fields are required, mark the
optional ones; if most are optional, mark the required ones.

Wording, rewrites, and the pt-BR register live in `copy.md`.
