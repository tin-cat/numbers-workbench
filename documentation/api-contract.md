# API contract

The surface between source applications and Numbers v2.

## The governing rule

**The caller never proposes a series or a number. It says which source it is; Numbers v2 assigns.**

The moment a caller can suggest a number, the chain has a hole in it. This is the one rule that must
never be relaxed for convenience.

## Verbs

| Verb | Purpose |
|---|---|
| `issueInvoice` | Issue a new invoice. Idempotency key, source, customer facts, line items, currency. |
| `issueRectificativa` | Correct or annul an existing invoice. References the original, states R1 to R5 and whether it is sustitutiva or por diferencias, full or partial. |
| `getInvoice` | One invoice by its code. |
| `listInvoices` | A source's invoices for one of its customers. |
| `getPdf` | The rendered PDF. |
| `getInvoiceByExternalReference` | The invoice for a given payment: provider, kind, id. What the account page uses to show "the invoice for this charge", since the source holds no invoice data. |
| `signalCustomerState` | Retention signals. See [[data-retention]]. |

Refunds are not a verb. A refund is a rectificativa.

**There is no `deleteInvoice`, and there never will be.** Litmind's `Invoice::remove()` does not
survive this change, which is a feature: see [[source-data-findings#Missing numbers]] for the 106
gaps it left behind.

## Identity

A source is identified by its own credential. A customer is identified by `source` + `sourceId`,
where `sourceId` is whatever the source already uses (for Litmind, the user id). This means a source
stores nothing at all beyond identifiers it already has.

Note two problems in the current arrangement that must be fixed before anything is keyed on them:

- `NUMBERS_SOURCE` is defined as `1` in **both** the litmind and moonvillage configs, so source
  alone does not disambiguate.
- Old Numbers sends the *user* id as `sourceId`, so it does not identify an invoice. The natural key
  for an invoice is `source` + `code`.

## Idempotency

Load-bearing rather than a nicety, because a source that holds no invoice data cannot answer "did I
already invoice this charge" without asking.

- **The caller supplies the key.** For the SaaS sources it is the payment processor's transaction
  identity: the **Stripe charge id**, or the **PayPal transaction id**. Both are already in Litmind's
  `payments`. PayPal is live at roughly ten percent of current invoicing and is the source of the
  January 2026 duplicates, so it is a first-class case, not a footnote. See
  [[review-2026-09-06#R3]].
- **Store the key, a hash of the request body, and the response**, in the same transaction as the
  chain append. A replay returns the identical invoice payload, not a bare "already exists" that
  leaves the caller with nothing to render.
- **Unique constraint on the key.** Let the database refuse the duplicate. Never check-then-insert.
- **A reused key with a different body is a 409**, not a silent return of the old invoice. That case
  is a caller bug, and quietly serving the wrong invoice is worse than failing.

This is not theoretical. In January 2026 a PayPal IPN retrying against a 500 created duplicate
invoices for single payments in Litmind; 88 invoices had to be annulled with negative counterparts.
The incident is documented in the Litmind workbench under
`issues/2026-01 Repeated invoices issue`. Under Verifactu the same event would have written
duplicates into the chain. See also [[source-data-findings#Duplicate codes]].

## External references, for traceability

Numbers v2 stores the payment processor references that came with an invoice, because reconciliation
genuinely needs them: walking from an invoice to the charge that paid it, and asserting that every
Stripe refund has exactly one rectificativa. See [[corrections#Reconciliation]].

Three rules make this safe.

**Model them generically, not as `stripe_*` columns.** Manual invoices have no processor at all, and
a second product could use a different one. Store a small set of typed external references, each
`{provider, kind, id}`, owned by the source that supplied them. The SIF must not be coupled to one
payment processor.

**They are metadata, never part of the fiscal record.** The huella is computed over the field set
the AEAT defines, and these are not in it. Nothing about the validity or verifiability of a record
may depend on an external reference resolving.

**Transaction references in, product references out.**

| Reference | Store? | Why |
|---|---|---|
| Charge id | Yes | It is the issuance idempotency key already |
| Refund id | Yes | The rectificativa idempotency key, and the reconciliation handle |
| Processor invoice id | Yes | Reconciling against the processor's own invoice list |
| Customer id | Yes, with a caveat below | Walking from a customer to their invoices during a dispute |
| Plan id, subscription id | **No** | Product data. Numbers has no business knowing what plan someone was on. It stays in the source. |

**The customer id is personal data and is minimized early.** Unlike a charge id, which is scoped to
one transaction, a processor customer id is a stable identifier that stays resolvable to a named
person with an email and a card for as long as the processor account exists. It belongs in the
**minimize** set, not kept for the life of the record.

Litmind already treats these this way: `anonimizeInvoices()` nulls `stripeInvoiceId`,
`stripeChargeId`, `stripePlanId` and `stripeSubscriptionId`. That classification carries across. See
[[data-retention#Four operations, four different clocks]].

## Tax determination lives here

The caller passes **facts**, not conclusions: customer country, NIF, whether it is B2B or B2C, and
the product type. Numbers v2 decides the treatment and produces `CalificacionOperacion`,
`OperacionExenta` and `ClaveRegimen`, because that is Verifactu vocabulary and belongs with the SIF.

This is the largest single piece of migration work, because it means lifting the country and
province tax configuration out of each source application. In Litmind that is
`getInvoicesIvaPercentage` / `getInvoicesIrpfPercentage` and the `invoicesIvaMethod` /
`invoicesIrpfMethod` columns on the country and province tables.

Carry across the known defect while doing it: `getInvoicesIvaMethod(): int` returns
`$province['invoicesIvaMethod'] ?? $country['invoicesIvaMethod']`, and for a country where that
column is NULL this is a `TypeError` rather than a decision. In Numbers v2 that case is an explicit
rejection, not a default.

## Consequences of holding no local data

Things that follow from [[architecture#No local copies]] and need designing for:

**The invoice email.** Litmind currently attaches the PDF via `attachedInvoiceId`, rendering at send
time rather than storing bytes. That pattern survives: it becomes "fetch from Numbers v2 at send
time", which keeps the email queue free of invoice bytes. The alternative, Numbers v2 sending the
email itself, would need the templates, the branding and the four languages, so sending stays in the
source. It is already on a queued, retryable path.

**The account invoices page gains a failure mode.** Needs an honest empty state, not a blank page
and not a fatal. Same for the admin per-user invoice count.

**The GDPR data package export becomes a live call.** This is an improvement: the export then
returns exactly what Numbers v2 holds, which is the honest answer to a subject access request rather
than a second copy's opinion of it. Currently `UserDataPackageService.php:2898` and `:3013`.

**The invoice admin moves to Numbers v2** in its entirety, and becomes one admin for all sources
instead of one per application.

## Security

**Per-source credentials**, rotatable, scoped so a source can only touch its own invoices and its own
series, with an issuance audit log recording which caller asked. The current single shared static
`Key:` header does not survive this change: a compromised source can issue invoices under the NIF,
which is a fiscal problem rather than only a data one.

The certificate is handled separately, see [[deployment#Secrets]].

## Related

- [[architecture]]
- [[verifactu]]
- [[data-retention]]
