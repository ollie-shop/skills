# Checkout copy

Checkout copy has one job: **remove hesitation.** Nobody is here to read. Every word
either helps someone finish, or it is in the way.

**The requirements live in `design-judgement.md` and `hierarchy.md`** — label
placement, error recovery, validation timing, verb-matches-destination. This file
is the craft: how to actually write the string, and how to review a screenful of
them.

This is not the place for brand personality. A clever error message at a declined
payment is worse than a plain one. Save voice for the store, the emails, the empty
cart — never for the moment money moves.

**The merchant's brand voice is a separate matter.** If `checkout-context.md`
records it, follow it in the store's own surfaces; never at the moment money moves.
The defaults below apply to checkout microcopy.

## The four rules

**1. Name the actual thing.**
Not "Continue" but "Ir para pagamento". Not "Erro" but "CEP não encontrado". A label
that could sit on any button in any product is telling the person nothing.

**2. Say what happens next.**
The anxiety in checkout is almost always about what pressing this will do. "Finalizar
compra" and "Pagar R$ 189,90" both submit the order; only one tells you the amount
you're about to be charged.

**3. Never blame the person.**
"Você digitou o CEP errado" → "Não encontramos esse CEP". The system failed to find
it; that framing is both kinder and more often true — the CEP may be valid and simply
missing from the database.

**4. Cut every word that isn't load-bearing.**
"Por favor, insira o seu endereço de e-mail para continuar" → "E-mail". The field is
already labelled, already required, already in front of them.

## Buttons

The verb must match the destination. A button that says "Finalizar" and leads to
another step is a small betrayal, and it costs trust exactly when trust is scarce.

| Situation | Write | Not |
| --- | --- | --- |
| Moves to the next step | `Continuar para entrega` | `Continuar`, `Próximo` |
| Opens payment | `Ir para pagamento` | `Avançar` |
| Actually charges | `Pagar R$ 189,90` | `Finalizar`, `Confirmar` |
| Applies a code | `Aplicar` | `Enviar`, `OK` |
| Secondary, non-destructive | `Voltar ao carrinho` | `Cancelar` |

State the amount on the final button when the amount is settled. It removes the last
ambiguity about what the press does — accepting that it also puts the number in front
of someone one final time, which a small number will reconsider. That trade is
usually worth it; if a client disagrees, that's a legitimate position, not an error.

## Errors

Two parts, always: **what happened**, then **what to do about it.** One sentence each
at most. Never lose what they already typed.

| Instead of | Write |
| --- | --- |
| `Erro ao processar pagamento` | `Cartão recusado pelo banco. Tente outro cartão ou escolha Pix.` |
| `Campo inválido` | `CPF precisa ter 11 dígitos` |
| `CEP inválido` | `Não encontramos esse CEP. Confira ou preencha o endereço manualmente.` |
| `Falha na requisição` | `Não conseguimos calcular o frete agora. Tentar de novo` |
| `Item indisponível` | `Sabonete Dove esgotou. Remova para continuar.` |

Rules that matter more than wording:

- **A declined payment is the most expensive error in the flow.** The person already
  decided to buy. Give a real reason where the gateway provides one, keep every field
  filled, and offer the next-best method by name.
- **Phrase it for someone mid-form, not someone being scolded at the end of one.**
  (That errors must carry an action, and that validation fires on blur, are
  requirements — `design-judgement.md`.)

## Field labels and helper text

- Use the local term: `CEP`, not `Código postal`. `CPF`, not `Documento`.
- Helper text only where the field genuinely surprises people — `Complemento`
  ("Apartamento, bloco, referência") earns it; `E-mail` does not.
- When a field needs a reason, write the reason well — "Para o entregador falar
  com você", not "Usamos para contato". *That* it needs one is a requirement;
  what it says is craft.

## Reassurance

Place it where the doubt is, not in the footer where nobody scrolls.

- Beside card fields, not at the bottom of the page.
- Concrete over vague: "Seus dados são criptografados" beats a padlock icon alone.
- Delivery promises must be dates the system can actually keep. Copy that overstates
  what shipping can do is a support cost, not a conversion win.
- Return and exchange terms belong near the final action for considered purchases,
  and are noise for a R$ 20 basket. Match the reassurance to the size of the risk.

## pt-BR specifics

- **Second person, informal `você`.** Consistent throughout — never drift into
  `senhor`, and never mix registers between steps.
- **Don't translate English patterns literally.** "Prossiga para o checkout" is a
  translation; "Ir para o pagamento" is Portuguese.
- Imperative for actions (`Escolha a entrega`), not gerund (`Escolhendo a entrega`).
- Local vocabulary: `frete` not `envio`, `carrinho` not `sacola` (unless the client's
  own brand says otherwise), `parcelas` not `prestações`.
- Currency always `R$ 1.299,90` — dot for thousands, comma for decimals, space after
  the symbol.

For any other market, use the terms, formats and register that market expects —
the store's own configuration is the source, not a translation of the pt-BR table.

## Reviewing existing copy

Read every string in the step aloud in order, as one sequence. Most checkout copy
problems are not bad sentences — they are five fine sentences that repeat each other,
or that shift register halfway through.

Report only strings worth changing, with the replacement written out. A list of
observations without rewrites is not useful; if you flag it, write the fix.

## Related

- `design-judgement.md` — the requirements these strings have to satisfy, and the
  evaluation that routes wording findings here
- `hierarchy.md` — labels, fields, and the verb-matches-destination rule
