# Cancellations and refunds

How an invoice is corrected once it exists. Decided 2026-09-06.

Nothing is ever edited and nothing is ever deleted. Every correction is a new record.

## Two different mechanisms, and the line between them is not where you would guess

Verifactu has two kinds of record, and the distinction is **about the record, not about the
invoice**:

| | `RegistroFacturacionAlta` | `RegistroFacturacionAnulacion` |
|---|---|---|
| What it is | A new invoice record. Includes ordinary invoices *and* rectificativas | Withdraws a record previously sent to the AEAT |
| Used when | An invoice exists and is being issued or corrected | A **record** was sent that should not have been: sent twice, or sent for an invoice that was never actually issued |
| Is it about money? | Yes, it is an invoice | No. It says nothing about the commercial reality, only that a record was wrong |

**The practical rule:**

- **The invoice reached the customer and now needs undoing** (a refund, a cancelled sale, an invoice
  issued in error that the customer already has) → **a rectificativa**. You cannot make a document
  someone is holding disappear.
- **Only the record was wrong** (a bug submitted it twice, or submitted one for an invoice that was
  never issued) → **an anulación**.

An earlier version of this document said an invoice issued in error takes an anulación. That is too
loose and would have produced the wrong flow. Once an invoice has been delivered, correcting it is a
rectificativa regardless of how wrong it was.

So the January 2026 Litmind incident splits by whether the duplicate invoices actually reached
customers. Those that were emailed need rectificativas; any that never left the system would be the
anulación case, had a record been sent. **CONFIRM the exact boundary with the gestor**, because it
turns on whether a document was issued in the legal sense, not on whether a row was written.

## `TipoFactura`: the legal grounds for the correction

**This is the part that is not obvious: R1 to R5 are not severity levels or amounts. They say
*under which article of the VAT law* you are correcting**, and each maps to a different provision of
Ley 37/1992 (LIVA). The full field also covers ordinary invoices:

| Code | Meaning |
|---|---|
| `F1` | Ordinary invoice, with full recipient details. **This is what we issue.** |
| `F2` | Simplified invoice (a ticket, no recipient identified). We never issue these. |
| `F3` | An invoice replacing previously declared simplified invoices. Not applicable. |
| `R1` | Rectificativa under **Art. 80.Uno, Dos and Seis LIVA**, and for an error grounded in law. Art. 80.Uno is the important one: **the taxable base is reduced when the operation is wholly or partly cancelled, or the price is altered after the fact.** "Error grounded in law" covers applying the wrong VAT rate or the wrong exemption. |
| `R2` | Rectificativa under **Art. 80.Tres**: the customer has entered insolvency proceedings (concurso de acreedores). |
| `R3` | Rectificativa under **Art. 80.Cuatro**: bad debts, after the legally defined process. A different flow entirely, with time limits and formal claim requirements. |
| `R4` | Rectificativa, **everything else**. In practice, correcting data that is not the tax base: a wrong name, a wrong address, a mistyped NIF. |
| `R5` | Rectificativa of simplified invoices. Not applicable, since we never issue `F2`. |

### What this means for our cases

**CONFIRM all of this with the gestor**, but the expected mapping is narrow:

| Our case | Expected | Why |
|---|---|---|
| A membership or ad refunded, wholly or in part | **R1** | The operation is cancelled or partly cancelled and the price returned. That is Art. 80.Uno. |
| A duplicate invoice the customer received | **R1** | The operation it describes never existed, so it is cancelled in full. |
| Wrong customer name, address or NIF on an otherwise correct invoice | **R4** | The tax base is not changing; identifying data is. |
| Wrong VAT rate applied | **R1** | An error grounded in law. |
| An unpaid invoice written off | **R3** | Only if the bad-debt procedure is actually followed. Out of scope for now. |
| Customer insolvency | **R2** | Out of scope for now. |

So **R1 covers essentially everything we will actually do**, R4 is the occasional data fix, and
R2/R3 are separate procedures we are not building.

## `TipoRectificativa`: how the correction is expressed

A **second, independent** axis, and the one most often confused with the first.

- **`I`, por diferencias.** States only the **delta**. A 20 EUR refund on a 100 EUR invoice is a
  rectificativa carrying -20.
- **`S`, por sustitución.** **Replaces** the original, restating the corrected invoice in full with
  its new figures, and additionally declaring `ImporteRectificacion`, the original amounts being
  replaced.

**Recommendation: use `I` (por diferencias) for every refund, partial and full alike.**

This revises an earlier note here that suggested `S` for full reversals. Both are legal, but for a
system generating these automatically, `I` is better:

- One code path instead of two. A full refund is a partial refund of the whole amount.
- No `ImporteRectificacion` block to build and get right.
- The delta is what actually happened: money moved back, by an amount.

`S` earns its place when a corrected invoice is being reissued in full, which is the `R4` data-fix
case rather than the refund case.

## The reference to the original

Every rectificativa carries `FacturasRectificadas`, identifying the invoice or invoices it corrects.

**This is the field Litmind holds and old Numbers does not**, which is why the migration has to join
the two. See [[migration#The source of record is old Numbers]].

Today Litmind's `Invoice::refund()` produces a plain negative invoice in the `WEBANULACION` series
with no `TipoFactura`, no `TipoRectificativa` and no structured reference to the original. All three
have to be added.

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
