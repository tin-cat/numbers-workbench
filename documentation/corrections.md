# Cancellations and refunds

How an invoice is corrected once it exists. Decided 2026-09-06.

Nothing is ever edited and nothing is ever deleted. Every correction is a new record.

## Two different things that are easy to conflate

| | Annulment | Rectificativa |
|---|---|---|
| When | The invoice should never have existed: issued in error, duplicate, wrong customer | A real sale that is being corrected: a refund, a price correction, a returned service |
| Verifactu mechanism | A *registro de anulación*, which annuls the record | A new invoice of type R1 to R5, referencing the invoice it corrects |
| Money | Usually none moved, or it is being fully returned because the sale was never valid | Money is returned, wholly or in part |
| Numbers appear | The annulled invoice keeps its number; the annulment is a record about it | The rectificativa takes the next number in the rectificativa series |

The January 2026 incident in Litmind is the annulment case: a PayPal IPN retrying against a 500
created invoices for payments that had already been invoiced. Those invoices should never have
existed. See [[source-data-findings#The repeated invoices incident, January 2026]].

A member who cancels a membership and is refunded is the rectificativa case.

**The flows must ask which one it is** rather than inferring it from whether money moved. Getting it
wrong is not correctable by anything except another record.

## Rectificativa shape

Two axes, both required:

- **`TipoFactura`**, R1 to R5, which says on what legal basis the correction is made. **CONFIRM**
  the mapping of our cases with the gestor: a refunded service is most likely R1 or R4.
- **`TipoRectificativa`**, `S` or `I`:
  - **`I`, por diferencias**, states only the delta. This is the natural shape for a **partial
    refund**.
  - **`S`, por sustitución**, restates the corrected invoice in full with its new figures. This is
    the natural shape for a **full reversal** and for a corrected price.

Every rectificativa carries `FacturasRectificadas`, the reference to the invoice or invoices it
corrects. **This is the field Litmind holds and old Numbers does not**, which is why the migration
has to join the two. See [[migration#The source of record is old Numbers]].

Today Litmind's `Invoice::refund()` produces a plain negative invoice in the `WEBANULACION` series
with no type, no rectificativa axis and no structured reference to the original. That is what has to
change.

## Who triggers the Stripe refund

**Litmind, not Numbers v2.** The source application moves the money; Numbers v2 records the fiscal
consequence.

This follows the ownership boundary the whole architecture rests on: the source owns the commercial
fact, Numbers owns the fiscal document. A refund is a commercial act. The rectificativa is the
fiscal document about it.

Concretely, the reasons:

- **A refund in Litmind is never only a refund.** It cancels the noticeboard ad, revokes the
  membership, sends the email, updates `NoticeboardAd::setPayment`. Numbers has no business knowing
  any of that, and could not orchestrate it if it did.
- **Blast radius.** Numbers v2 already holds one crown jewel, the signing certificate. Giving it the
  Stripe secret keys of every source application, with refund permission, means a Numbers compromise
  can move money. That is a large and unnecessary expansion.
- **N sets of credentials.** Each SaaS product has its own Stripe account. Numbers would have to
  hold all of them, and manual invoices have no Stripe at all, so it would need a second path
  regardless.
- **Atomicity is not available anyway.** Stripe is an external system; there is no transaction that
  spans it and our database whichever side calls it. The problem is a saga either way, so put the
  saga where the other consequences already live.

## Ordering, which is the part that matters

Refund **first**, rectify **second**. The two failure modes are not symmetrical:

- Refund succeeds, rectificativa not yet issued: a temporary fiscal gap, fully recoverable, because
  the rectificativa command is queued, durable and idempotent and will land on retry.
- Rectificativa issued, refund fails: a permanent, immutable record in the chain asserting a refund
  that never happened, correctable only by yet another record.

So the rectificativa is requested from a durable queue **keyed on the Stripe refund id**, exactly as
issuance is keyed on the charge id. See [[api-contract#Idempotency]].

## Trigger the rectificativa from the webhook, not from the UI action

Litmind already handles `charge.refunded` and creates the refund invoice there. **Keep it there.**

That placement is not incidental: it means a refund issued by hand from the Stripe dashboard
produces a rectificativa exactly like one issued from the admin UI. If the trigger moved to the
button, dashboard refunds would silently produce no fiscal record, and that is a gap nobody would
notice until an inspection.

The Stripe refund id from the webhook is also the natural idempotency key.

## Foreign-currency corrections

A rectificativa or annulment of a USD invoice is issued in USD, and uses **the exchange rate stored
on the original invoice**, never the rate of the day it is corrected. A refund converted at a
different rate from the sale does not net against it, and the difference is not an FX gain, it is a
broken correction.

See [[money#Corrections carry the original rate]].

## Reconciliation

A periodic job that lists Stripe refunds for a period and asserts that each has exactly one
rectificativa, and that each rectificativa has a matching refund. Cheap, and it is the only thing
that catches a whole class of silent divergence.

## The partial refund flow

1. An operator (or an automatic path) decides on a partial refund of invoice `X`, amount `A`.
2. Litmind validates `A` against the invoice total, refunds `A` through Stripe, applies its own
   consequences.
3. The `charge.refunded` webhook arrives carrying the refund id.
4. Litmind queues a `IssueRectificativa` command: original invoice, amount, reason, refund id as
   idempotency key.
5. Numbers v2 issues a rectificativa `por diferencias` for `A`, chains it, renders its PDF, queues
   it for AEAT submission, and returns it.

Note step 2 validates the amount but **Numbers v2 validates it again**, against its own record of
the invoice, and refuses a rectificativa that exceeds what remains uncorrected on it. The source is
not trusted for that: it holds no invoice data.

## Related

- [[architecture]]
- [[verifactu]]
- [[api-contract]]
- [[money]]
