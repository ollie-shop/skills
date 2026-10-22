# `checkout-context.md` — the store's memory

A file at the project root that carries **the facts about this store that stay
true across sessions**: who buys here, what data exists, what the store already
uses, and what has already been decided. The project's `AGENTS.md` tells every
agent to read it before proposing anything.

Research that stays in one chat dies with it. The next person on this store —
or you, next month — starts from nothing. This file is the cheapest fix.

## Rules that keep it trustworthy

- **Date what you write.** Every authored section says when, and based on what.
- **A guess is labelled as a guess.** "Shoppers here prefer pickup" is either a
  finding with a date and a source, or a hypothesis. Write it as one or the other.
- **No pipeline stage.** Never "we are in exploration". A stale stage makes the
  next agent confidently do the wrong thing.
- **No secrets.** It lives in the merchant's repository: no credentials, tokens or
  internal URLs. The merchant's own business rules belong here.

A context file nobody trusts is worse than none, because it gets read anyway.

## Skeleton

Copy this into `checkout-context.md` at the project root and fill it in. Leave a
section as "Unknown" rather than inventing it.

```markdown
# <Store> — checkout context

## The store

Platform: <VTEX / Shopify / VNDA / custom>. Store ID: <id>.
Sells in: <countries and locales>. Currency: <code>.

## Who we are building for

**This is a hypothesis, not research.** Based on: <what>. Written <date>.

<Two or three short paragraphs: who they are, what they buy, what they are
anxious about when paying, what device they are on.>

### How to prove this wrong

- <the measurement, recording or conversation that would confirm or kill it>

## What data we have

| Source | Covers | Last refreshed |
| --- | --- | --- |
|  |  |  |

**What we do not have:** <name it plainly>

## Brand and tokens

<Where the merchant's tokens come from; the colour roles that carry meaning
(selected, error, disabled) and anything known to be weak in them.>
Voice: <how the store speaks, if it has a defined voice>.

## What this store already uses

| Slot | Component | Notes |
| --- | --- | --- |
|  |  |  |

## Business rules that shape the checkout

- <rule, and where it is enforced — a hub function, the platform, the admin>

## Decisions of record

| Date | Decision | Why the alternative lost | Link (PR or thread) |
| --- | --- | --- | --- |
|  |  |  |  |
```

## When to update it

- After any research: a persona, a funnel reading, a session review.
- When a decision is made that the next person should not re-litigate.
- When a slot gets a component, or loses one.

Move a hypothesis to a finding only with a date and a source. Never delete a
decision; supersede it with a new row that says why.
