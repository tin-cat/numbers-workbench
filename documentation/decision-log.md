# Decision log

Decisions taken, with the options that were rejected and why. All of these were settled in the
design session of **2026-09-06** unless stated otherwise.

---

## 1. Verifactu is integrated in the invoicing system, not in the expense ledger

**Decided.** The obligation attaches to the act of *expedición*. The system that assigns the series
and number, dates the invoice, computes the tax and hands the document to the customer is the SIF.

The original Numbers was never a candidate: it receives invoices asynchronously and with retries, so
records would arrive out of order, late and occasionally not at all. A hash chain built on arrival
order is not a hash chain. It also holds no PDF, and the QR must be on the document the customer
receives.

---

## 2. One SIF for the NIF, not one per application

**Decided.** The NIF `ES 39898734J` issues invoices from Litmind, from other SaaS products, and by
hand in Excel.

**Rejected: each application as its own SIF**, each with its own chain and its own distinct series.
This is permitted (**CONFIRM** with the AEAT) and was the first design. It fails on maintenance:
five compliance surfaces, five certificates on five hosts, five *declaraciones responsables*, and
five deploys every time the specification changes. The compliance surface is the real cost of
Verifactu, not the hashing.

**Chosen: Numbers v2 is the only SIF.** Applications become callers. Series management becomes
internal, collisions become impossible by construction, the numbering race gets fixed once, and the
multiple-SIF assumption stops being load-bearing.

---

## 3. Numbers v2 issues, including the PDFs

**Decided.** Numbers v2 assigns the series and number, chains the record, renders the PDF with a
per-source template, and submits to the AEAT.

The earlier objection to this (the QR has to be on a PDF that Numbers does not hold) dissolves once
Numbers renders the PDF itself.

---

## 4. Writes asynchronous, Numbers authoritative

**Decided.** These are separate axes and conflating them was the error in the first design pass.
Numbers v2 is authoritative for issuance **and** issuance is queued.

Litmind creates invoices inside a Stripe webhook, after the membership renewal side effects have
already committed. A blocking call there would mean a Numbers outage takes the money, grants the
membership and produces no invoice, with an ugly recovery path through the
`stripe_processed_events` guard.

**Chosen:** the source queues an `IssueInvoice` command with a stable idempotency key. An outage
delays invoices; it never loses them. This also makes the cutover a consumer pause, see
[[migration#Cutover]].

---

## 5. Source applications hold no invoice data at all

**Decided.**

**Rejected: a local read model in each application**, so account pages never depend on Numbers being
up. This was proposed and then rejected by the owner on GDPR grounds, correctly: every copy is a
retention leak, every retention transition would have to be pushed and applied everywhere, and a
missed push is a silent breach.

**Chosen: no copies, live reads.** The cost turned out to be small. Only four places in Litmind read
`invoices` outside its own module, and nothing in `statistics` touches it, so no aggregate reporting
was quietly depending on a local table. Revenue reporting runs off `payments`, which stays.

Caching is not forbidden, only bounded: a short-lived non-durable response cache flushed on the
erasure signal would not meaningfully reintroduce a copy. Do not build it up front.

---

## 6. Numbers v2 owns retention; sources send signals, not commands

**Decided.** The decisive reason is technical: the huella covers the record's contents, so
anonymizing a chained record destroys the ability to prove it matches its own hash. Retention must
live where the chain lives.

The legal reason is just as good. The retention clock belongs to the invoice, not to the customer's
account, and Article 17(3)(b) exempts processing required by a legal obligation. "Retained until
30/6/2031, then erased" is the correct answer and the current design cannot express it.

Litmind's existing policy moves across intact: it already runs invoicing anonymization separately,
at the end of the fifth fiscal exercise, on a June 30th because that is when the IRPF closes.

Two signals flow inward, both recorded rather than obeyed: **customer cancelled on date X** and
**erasure requested on date X**. The infringing level is not among them: it only moves the partial
anonymization date, and invoices are touched at complete anonymization.

Moving the clock also resolves the open DPD question in Litmind's config, since Numbers v2 anchors
on the invoice's own exercise of issuance rather than on an account state.

---

## 7. Numbers v2 is a rebuild, not an evolution

**Decided.** The original Numbers is a small PHP application with per-action `.php` endpoints, one
shared static `Key:` header, form-encoded POSTs and `float` money columns. That is a mirror's
architecture. It cannot become a system of record by extension.

The valuable thing in the old system is **the data, not the code**: 54,516 invoices, 1,504 expenses,
109 suppliers and the series history. The rewrite risk is in the migration, not in the rewrite. See
[[migration]].

Old Numbers is frozen read-only at cutover, never decommissioned.

---

## 8. Kubernetes for deployment and portability, with the database outside it

**Decided.** The reasons given are the deployment process and the ease of moving between clusters,
neither of which depends on scale. At fifteen invoices a day there is no scaling problem.

The database stays outside the cluster, precisely in service of the portability goal: stateless
workloads move trivially, stateful ones do not. See [[deployment]].

---

## 9. Historical invoices are imported inert, never retro-chained

**Decided.** Records issued before go-live have no huella and must not be given one; that would be
manufacturing attestations about when they were issued. The chain starts at go-live with
`PrimerRegistro: S`.

This is what makes the first two migrations low-risk: nothing imported can corrupt a chain.

---

## 10. No delete verb, ever

**Decided.** Litmind's `Invoice::remove()` does not survive. It left 106 gaps in the `WEB` series,
which its own comment warned it would. Corrections are made with new records.

---

## 11. Numbers v2 issues the manual invoices itself

**Decided.** A simple invoice creation flow in Numbers v2 for consultancy work and anything else
invoiced by hand, rather than sending that channel to the AEAT's free basic application.

The channel is small (78 invoices in twenty years, see [[source-data-findings#Old Numbers]]) but
once issuance, series allocation, chaining and PDF rendering exist for the SaaS sources, a form over
them is close to free, and it keeps every invoice under the NIF in one system with one chain.

Excel stops being used for invoicing. It is not a SIF and cannot become one.

Note this is **deadline-critical**, not deferrable: a manual invoice issued after the deadline is
issued under the same NIF and must be a Verifactu record like any other.

---

## 12. The migration source of record is old Numbers

**Decided (owner, 2026-09-06).** The production instance of old Numbers is complete and up to date,
and holds every invoice for every source. It is the migration source.

Litmind is not a second source but an **independent witness**, used for three specific things it
holds and old Numbers does not:

- **The rectificativa links.** Old Numbers has no `anullation_invoice_id` equivalent, so the
  relationship between each of the 2,722 `WEBANULACION` invoices and the invoice it corrects exists
  only in Litmind. Verifactu needs it (`FacturasRectificadas`).
- **Amount precision.** Old Numbers stores money as `float`, Litmind as `double`. See
  [[source-data-findings#Amount precision]].
- **Anonymization state.** See [[source-data-findings#Old Numbers holds data Litmind already anonymized]].

---

## 13. Money is an integer of minor units plus an explicit scale

**Decided.** Never a float, never a double, nowhere, including on the wire. `Money`, `Currency` and
`Amount` value objects, immutable, with no operation that rounds implicitly, and a `TaxBreakdown`
that refuses to exist unless `base + iva - irpf == total` exactly.

Scale is stored explicitly rather than derived from the currency, because line amounts in the
existing data carry four decimal places, not two.

Full detail in [[money]].

---

## 14. Litmind triggers the Stripe refund, Numbers v2 issues the rectificativa

**Decided.** The source application moves the money; Numbers v2 records the fiscal consequence. A
refund in Litmind is never only a refund (it cancels the ad, revokes the membership, sends the
email), and giving the SIF the Stripe keys of every source would let a Numbers compromise move
money.

**Order matters and is not symmetrical**: refund first, rectify second. A refund without its
rectificativa is a recoverable gap; a rectificativa without its refund is a permanent immutable
record of something that never happened.

The trigger stays on Litmind's existing `charge.refunded` webhook rather than moving to the admin
button, so that a refund issued by hand from the Stripe dashboard also produces a rectificativa.

Full detail in [[corrections]].

---

## 15. The gestor export is an external contract

**Decided.** The Excel workbook Victor integrates does not change: two sheets, `Ingresos` and
`Gastos`, the exact columns, headers, widths, styling, the blank row between months, and the
percentage columns as live formulas rather than values.

Pinned with a golden-file test against a reference workbook kept in the repository, because that is
the only thing that keeps the promise true once nobody remembers why column J is a formula.

Downloadable always, emailable on an explicit button with the recipient in configuration and every
send audit logged.

Full detail in [[gestor-export]].

---

## 16. Multi-currency from day one: EUR and USD

**Decided.** Not a later feature. 111 of Litmind's invoices are USD, issued continuously since May
2019.

The parts that matter are not the currency field but what surrounds it: the tax figures also have to
exist in euros, the exchange rate used is **captured at issuance and stored permanently** because
the huella covers the amounts and a re-derived rate is a broken record, rates are cached locally so
issuance never blocks on an external service, a missing rate is a **refusal to issue** rather than a
guess, and a correction uses the rate of the invoice it corrects rather than the rate of the day.

Full detail in [[money#Currencies]].

---

## 17. Payment processor references are stored, but generically and with a retention class

**Decided.** Numbers v2 keeps the charge id, the refund id, the processor invoice id and the
customer id, because reconciliation needs them: walking from an invoice to the charge that paid it,
and asserting every refund has exactly one rectificativa.

Three constraints. They are modelled as generic `{provider, kind, id}` references rather than
`stripe_*` columns, because manual invoices have no processor and a second product could use a
different one. They are **metadata, never part of the fiscal record**, so nothing about a record's
validity may depend on one resolving. And plan and subscription ids are **not** stored: those are
product data and stay in the source.

The **customer id is personal data** and belongs in the minimize set, unlike the transaction-scoped
ids. Litmind already treats it that way in `anonimizeInvoices()`.

Full detail in [[api-contract#External references, for traceability]].

---

## 18. The 47 currency-less invoices are EUR

**Decided (owner, 2026-09-06).** 47 invoices from a five-day window in April 2019 have no currency,
totalling 212.81. The window closes a week before the first USD invoice, so the column was very
likely added while USD support was being built and these were never backfilled.

They are imported as EUR, set **explicitly on the row** during the migration rather than defaulted
at read time, so the assumption lives in the data with a decision behind it.

---

## Open

### Confirmations required

Listed in [[verifactu#Open questions to confirm with the AEAT]]. The deadline confirmation is the
one that sets the schedule and the one that closes the rollback window.

### Does Victor want a period filter?

The current export is all time, with no period filter. A gestor normally wants a quarter or a year.
Adding a selector does not change the format, but it is a behaviour change and his integration may
depend on receiving the whole history. Ask before assuming. See
[[gestor-export#An open question for Victor]].

### Does Victor want USD converted?

The current export writes a USD invoice's raw amount into the same Subtotal, IVA and Total columns
as the EUR ones, changing only the cell's number format. A column sum therefore adds dollars to
euros. With 111 USD invoices totalling 1,853.20 this is small, but it is either something Victor
handles by hand or something nobody has noticed. Ask before changing anything: it touches the format
contract. See [[gestor-export#An open question for Victor]].

### Rectificativa type mapping

Which `TipoFactura` (R1 to R5) our refund cases fall under. Needs the gestor. See
[[corrections#Rectificativa shape]].

### Tax determination migration

Lifting the country and province tax configuration out of each source application is the largest
single piece of work and has not been scoped. See
[[api-contract#Tax determination lives here]].
