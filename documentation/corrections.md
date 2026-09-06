# Cancellations and refunds

How an invoice is corrected once it exists. Decided 2026-09-06.

Nothing is ever edited and nothing is ever deleted. Every correction is a new record.

## Five cases, from the AEAT's own developer FAQ

Section 17 of the AEAT developer FAQ ("Forma de proceder ante errores cometidos al facturar",
version 1.3, 4 December 2025) sets this out directly. It supersedes both earlier framings in this
document, which were each wrong in a different direction.

| # | When | What to do |
|---|---|---|
| 1 | Error found **before** the invoice is issued, while still editing | Just fix it. No record exists yet. |
| 2a | Error found after issuance, of a kind the **ROF** covers (amounts, content, anything on the printed invoice) | **A rectificativa.** A new invoice of type R1 to R5. |
| 2b | Error found after issuance in **internal record fields** that do not appear on the invoice, such as a tax classification code | **An `alta de subsanación`**: correct the original and generate a new alta record carrying the corrected data. "Should be very infrequent." |
| 2c | Error affects neither the ROF nor any record field | Fix the invoice. No new record of any kind. |
| 2d | The whole invoice **should never have existed**, and the ROF does not require a rectificativa | **An `RF de anulación`** (art. 11 RRSIF). "Should be very infrequent." |

### The test for annulment is whether the operation was real

The FAQ is unambiguous:

> Con carácter general, todas las facturas emitidas, en la medida en que respondan a operaciones
> realmente efectuadas (como es el caso habitual) **no pueden anularse**.

Annulment applies "cuando se haya emitido erróneamente una factura", and the example given is an
invoice for a service or delivery **that does not exist and was never performed**. Test invoices and
training invoices are the cited cases.

So the criterion is **the reality of the operation**, not whether money moved and not whether the
document was delivered. A refund of a real sale can never be an annulment, however completely the
money went back.

Delivery is a **secondary** consideration, and the FAQ says so: if the faulty invoice was **not**
delivered to the customer, that supports treating it as a failed issuance and annulling it, then
issuing a fresh correct original rather than a rectificativa.

Both mechanisms are described as exceptional: "deben ser casos muy excepcionales, siempre a valorar
y utilizar con prudencia."

### Where that puts the January 2026 incident

The PayPal IPN retried against a 500 and produced extra invoices for payments that happened once.
Those extra invoices describe operations that were never performed, which is squarely the annulment
case. Where one was delivered to the customer, the secondary consideration pulls the other way and
it is worth asking the gestor.

The design consequence matters more than the classification: **that situation must not be reachable
again.** See [[#Preventing it at the bottom]].

### `alta de subsanación` after an AEAT rejection

A case the submission state machine has to handle. If the AEAT **rejects** a record, the record does
not exist at the AEAT at all, and the correction is an alta de subsanación carrying
`Subsanacion = "S"` and `RechazoPrevio = "X"`. If the record was **accepted with errors**, it stays
at the AEAT with those errors forever and the correction is an ordinary alta de subsanación.

So "rejected" and "accepted with errors" are not the same state and must not be collapsed into one
in the state machine. See [[architecture#Submission never blocks issuance]].

## Preventing it at the bottom

### Two different duplicates, and only one of them is caught for us

Worth separating, because they fail in opposite ways:

| | **Same code** | **Same sale** |
|---|---|---|
| What it is | Two invoices sharing a serie and número | Two invoices, each with its own valid number, for one transaction |
| In our data | **8 invoices**, from the numbering race. See [[source-data-findings#Duplicate codes]] | **88 invoices**, the January 2026 PayPal incident |
| What the AEAT does | **Rejects** it: "Registro de facturación duplicado" | **Accepts** both, happily. Each has a distinct identity, so nothing looks wrong |
| Who has to catch it | The AEAT will, eventually and noisily | **Only we can** |

The second is the worse problem. A same-code duplicate is loud: the submission fails and someone
investigates. A same-sale duplicate is silent, lands cleanly in the chain, and becomes a customer
charged once and invoiced twice, correctable only by issuing further records.

### The mechanics of the first

The AEAT identifies a record by **`Emisor` + `SerieYNúmeroFactura` + `FechaExpedición`**. Submitting
a second record with the same identity returns **"Registro de facturación duplicado"**, and the FAQ
is explicit that a number **cannot be reused even after an annulment**:

> El sistema VERI*FACTU no acepta estas operaciones porque, una vez que se comunica un registro de
> anulación, cuando se intenta subir el subsiguiente registro de alta con el mismo número provoca un
> error "Registro de facturación duplicado."

This is exactly what Litmind's `select max(number)+1` race produces, and it has already fired eight
times in production. See [[source-data-findings#Duplicate codes]].

### Three structural defences

Duplicate prevention is not a validation rule bolted on at the API. It is structural:

1. **Numbering is serialised**, by a locking read on the chain head, so two concurrent issuances
   cannot claim the same number. See
   [[architecture#The chain is a strict single-writer structure]].
2. **A unique constraint** on `(issuer, series, number)`, so the database refuses a duplicate even
   if the application logic is wrong.
3. **Idempotency on the caller's key**, so a retried request returns the original invoice instead of
   issuing a second one. This is what would have prevented the January 2026 incident: the PayPal
   retries carried the same payment. See [[api-contract#Idempotency]].

**The first two close the same-code case. Only the third closes the same-sale case**, which is why
it is not optional and why the caller's key has to be something that identifies the *transaction*
rather than the request. For the SaaS sources that is the payment processor's charge id, which is
stable across every retry.

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
