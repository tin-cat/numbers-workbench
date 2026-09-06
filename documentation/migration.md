# Migration

Getting from the current arrangement (each application issues its own invoices, old Numbers keeps a
mirror) to the new one (Numbers v2 issues everything).

The measured state of the data this plan is built on is in [[source-data-findings]].

## The source of record is old Numbers

**Owner, 2026-09-06:** the production instance of old Numbers is complete and up to date and holds
every invoice for every source. History is migrated from there.

Litmind is not a second source. It is an **independent witness**, and there are three specific
things it must still supply, because old Numbers does not have them:

| What | Why Litmind is needed |
|---|---|
| **Rectificativa links** | Old Numbers has no `anullation_invoice_id` equivalent. The relationship between each of the 2,722 `WEBANULACION` invoices and the invoice it corrects exists only in Litmind, and Verifactu needs it as `FacturasRectificadas`. |
| **Amount precision** | Old Numbers stores money as `float`, Litmind as `double`. See [[source-data-findings#Amount precision]]. |
| **Anonymization state** | Old Numbers was never told about anonymizations. See below and [[source-data-findings#Old Numbers holds data Litmind already anonymized]]. |

Everything else, including the 78 manual invoices that exist nowhere else, comes from old Numbers.

## It is two migrations plus a cutover

| | What | Risk |
|---|---|---|
| **1** | Old Numbers to Numbers v2, enriched from Litmind for the three fields above | Medium. One source, but a join that has to be right. |
| **2** | The cutover, where an application stops issuing locally and starts calling Numbers v2 | High. Live, money flowing, webhooks that do not pause. |

They have different failure modes and different remedies, and separating them is most of the work.

## The rule that de-risks 1 and 2: do not retro-chain

Historical invoices are not Verifactu records and must not be given a huella. See
[[verifactu#Historical invoices are not Verifactu records]].

Everything migrated is imported as **inert history**: no chain participation, no QR, no submission,
read-only. Nothing imported can corrupt a chain because none of it is in one. That is what makes the
first two migrations far safer than they look.

## The one value that must be exactly right

The per-series high-water mark. The first chained invoice continues from the last historical one.

Too low duplicates a number under the NIF. Too high creates a gap that cannot be explained. Derive
it, verify it independently against both source systems before go-live, and have Numbers v2 **refuse
to issue in any series whose high-water mark was not explicitly established**.

For Litmind that means `WEB` continuing past 56,012 and `WEBANULACION` past 2,895, both as of the
snapshot date, re-derived against live data at cutover.

## Reconciliation is the deliverable

The migration script is the easy part. What makes this reliable is a report that proves it, run
after every rehearsal:

- **Counts** per source, per series, per fiscal year.
- **Summed `subtotal`, `iva`, `irpf`, `total`** per source per fiscal year, matching exactly. Round
  explicitly on both sides or the float-to-decimal conversion will produce discrepancies that are
  not real. See [[source-data-findings#Amount precision]].
- **Series integrity**: for each series, a contiguous range with no gaps and no duplicates. **This
  will fail today**, with 8 duplicate codes, 24 code/number mismatches and 106 missing numbers.
  Failing is the correct outcome; the deliverable is the enumerated, explained exception list.
- **Rectificativa links**: every `anullation_invoice_id` resolves to an invoice that exists.
- **Rendered PDFs**: a sample of historical invoices re-rendered by Numbers v2 and compared against
  the old renderer's output. Litmind does not store PDFs, it renders them on demand in
  `Invoice::buildPdf()`, so the templates have to be ported and a template port is a place where a
  total can quietly shift.

The migration must be **idempotent and re-runnable**, keyed on the natural key `source` + `code`,
converging on the same state however many times it runs. Never "insert everything"; always upsert by
natural key. This is what makes rehearsal possible.

Rehearse against restored production snapshots, repeatedly, with the report as the pass gate. When
the report comes back clean twice in a row on fresh restores, you are ready.

Fix the source-identity problems first: `NUMBERS_SOURCE` is `1` in both the litmind and moonvillage
configs, and `sourceId` carries a user id rather than an invoice id. See [[api-contract#Identity]].

## Anonymization has to be applied during the migration, not after

Litmind's `anonimizeInvoices()` blanks the customer name, address, NIF, phone and email on its own
rows and publishes `Users\Events\InvoicingRelatedDataAnonimized`. **Nothing subscribes to that
event and forwards it to Numbers.** The only calls into the Numbers API are invoice creation and a
test tool.

So old Numbers still holds the full personal data of every customer Litmind has anonymized. This is
a live gap today, and it becomes a migration hazard: importing old Numbers verbatim would carry that
data into the new system and undo an erasure that was supposed to have happened.

**The migration must apply Litmind's anonymization state as it imports.** Any invoice Litmind has
marked `is_anonimized = 1` is imported with its customer fields blanked, taking Litmind's version of
those rows rather than old Numbers'. Reconcile on it explicitly: the count of anonymized invoices in
Litmind and the count of blanked ones in Numbers v2 must match exactly.

Worth deciding separately whether old Numbers should be purged of that data before it is frozen.

## Things that will surface

- **Anonymized invoices** have blanked names and NIFs in Litmind and full ones in old Numbers. See
  above; the blanked version wins.
- **`customer_dni_nif_cif` defaults to an empty string** in `invoices::create()`, so it is not
  reliably populated. Fine for history. A hard problem for go-live, where Verifactu wants a NIF for
  a Spanish B2B invoice.
- **The `WEBANULACION` series and the `anullation_invoice_id` links** must survive as rectificativa
  relationships, not as loose negative invoices.
- **The 78 manual invoices** use a different code convention entirely (`21001`, year plus sequence)
  and need their own series in the new scheme.
- **`suppliers` becomes `providers`** (owner, 2026-09-06), in the table, the domain and the
  interface. 109 rows. Rename during the import so the old term does not survive anywhere; expenses
  reference them, so the foreign key moves with it.
- **Every Spanish and misspelled column name is dropped here**, not carried forward: `date_emmited`,
  `anullation_invoice_id`, `is_anonimized`, `finantialBalances`, `base`, `final`. The migration's
  read side is the only code allowed to mention them, and it is throwaway. Full mapping in
  [[naming#Misspellings not to inherit]].
- **47 invoices have a NULL currency**, all within five days of April 2019, totalling 212.81. The
  owner decided on 2026-09-06 to treat them as **EUR**. Set it explicitly during the import, as a
  named migration step with its own line in the reconciliation report, so that in 2031 the record
  shows a decision rather than a coincidence. See [[source-data-findings#Currencies in use]].

## Cutover

**Reject dual-write.** Writing to both systems for a comfort period means two systems assigning
numbers from the same series, which is exactly the collision that must never happen.

### Step 1: shadow

The application keeps issuing locally as it does today, and additionally calls Numbers v2, which
issues into a **throwaway series** and discards the result. Real production traffic through the real
code path with zero fiscal consequence.

Run it long enough to see a renewal cycle, a refund and a foreign-VAT case.

### Step 2: flip authority, one source at a time

Litmind first, because it is the one that can be seen and tested end to end. Then the other
applications. The manual channel last.

Here the asynchronous-write decision pays off. Because issuance is a queued command rather than a
synchronous call inside the Stripe webhook, **the cutover is a consumer pause**:

1. Stop the queue consumer.
2. Charges keep succeeding; invoice commands keep queueing.
3. Flip the switch.
4. Drain the queue through the new path.

No downtime for customers, no lost invoices, and no window in which an invoice can fall between two
systems.

There is also a natural clean line at the cutover moment: every invoice before it is historical,
every invoice after it is chained. The *kind* of record changes at T, so there is no ambiguity about
which system owns which invoice.

## Rollback

Anything chained and submitted cannot be un-issued. So rollback means stopping issuance through
Numbers v2, reverting the application to local issuance, and **resuming its series from where
Numbers v2 left it**.

That requires the local issuance path to survive behind a feature flag rather than being deleted on
day one, and to accept an externally supplied high-water mark on resume.

**The rollback window expires.** After the compliance deadline, invoices issued by a reverted local
path are not Verifactu records, so rollback stops being an option and everything becomes
forward-fix. That is the practical argument for cutting over well before the deadline rather than
against it.

## Old Numbers is frozen, not decommissioned

When Numbers v2 goes live, old Numbers becomes read-only and stays. If a discrepancy surfaces in two
years it is the only independent witness, and the cost of keeping it is nothing.

## Related

- [[source-data-findings]] for the measured state of the data
- [[architecture]]
- [[verifactu]]
