# Findings in the existing data

Measured 2026-09-06 against the Litmind development database (a frozen production snapshot cut on
2026-06-17) and the old Numbers backup committed at `setup/backup.sql`, dumped 2025-08-24.

These are the numbers the [[migration]] plan is built on, and the reconciliation report described
there is essentially this document run automatically.

## Volume

Litmind's `invoices` table: **58,807 invoices** between 2011-02-07 and 2026-06-16.

| Year | Invoices | Per day |
|---|---|---|
| 2011 | 914 | 2.5 |
| 2012 | 1,068 | 2.9 |
| 2013 | 1,231 | 3.4 |
| 2014 | 990 | 2.7 |
| 2015 | 2,134 | 5.9 |
| 2016 | 3,271 | 9.0 |
| 2017 | 3,856 | 10.6 |
| 2018 | 4,962 | 13.6 |
| 2019 | 5,666 | 15.5 |
| 2020 | 5,126 | 14.0 |
| 2021 | 5,139 | 14.1 |
| 2022 | 5,393 | 14.8 |
| 2023 | 5,543 | 15.2 |
| 2024 | 5,458 | 15.0 |
| 2025 | 5,520 | 15.1 |
| 2026 | 2,536 (to 16 June) | 7.0 |

Steady state since 2019 is remarkably flat at about **15 invoices a day**.

Peaks are unremarkable. Busiest day in fifteen years is 150 (2016-08-01, clearly a batch run), next
is 129 (2026-01-13, the repeated-invoices incident below). **The maximum in any single minute across
the whole history is 11.**

This is what makes the strict single-writer chain constraint free rather than expensive. See
[[architecture#The chain is a strict single-writer structure]].

## The issuer has had ten identities

`company_name`, `company_address` and `company_dni_nif_cif` are stored per invoice row, and across
the history they take **ten distinct combinations**: two spellings of the name, five addresses, and
four formats of the same tax id. Old Numbers holds none of this, so it is a field Litmind alone
supplies to the migration, and a re-rendered historical PDF must show what was printed at the time.

One of them is an error. **6,343 invoices between 2024-12-10 and 2026-01-25 carry the tax id as
`ES J39898734`**, with the letter before the digits. It was printed on a year of invoices and is not
correctable; it belongs in the exception list that accompanies the migration.

Going forward, one canonical tax id (`39898734J`) for the records, with the historical display
value preserved separately.

## PayPal is live

27,906 invoices carry no Stripe charge id. Most are history from before Stripe, but **459 since
1 January 2025** have a `payment_id` and no charge: PayPal, at roughly ten percent of current
invoicing. The design has to treat it as a first-class payment source. See
[[review-2026-09-06#R3]].

## Almost no invoice identifies its customer by tax id

**54,964 of 58,807 invoices (93%) have an empty `customer_dni_nif_cif`**, and 53,199 of those go to
customers resident in Spain. For history it is inert. For issuance it decides whether consumer
invoices are complete (`F1`) or simplified (`F2`) invoices, which changes the record, the refund
code and the PDF. See [[review-2026-09-06#R4]].

## Timezone

All Litmind timestamps are UTC; the fiscal date is Spanish local. Checked with timezone tables
loaded: **no invoice in the history changes year** between the two, so the migration is safe. The
rule still has to be explicit going forward.

## Currencies in use

| Currency | Invoices | First | Last | Sum of totals |
|---|---|---|---|---|
| EUR (`2`) | 58,649 | 2011-02-07 | 2026-06-16 | 551,771.06 |
| USD (`1`) | **111** | 2019-05-06 | 2025-12-29 | 1,853.20 |
| **NULL** | **47** | 2019-04-25 | 2019-04-29 | 212.81 |

**USD is live and continuous**, not hypothetical, so multi-currency is a day-one requirement rather
than a later feature. See [[money#Currencies]].

**47 invoices have no currency at all**, all inside a five-day window in April 2019. That window
ends a week before the first USD invoice, which strongly suggests the `currency` column was added
while USD support was being built and these rows were never backfilled. They are almost certainly
EUR, but `Money` cannot be constructed without a currency, so the migration has to resolve them.

**Decided (owner, 2026-09-06): treat them as EUR.** The migration sets the currency explicitly on
these 47 rows rather than inferring it at read time, so the assumption is recorded in the data
rather than buried in code.

## Series integrity

| Check | Result |
|---|---|
| `WEB` series rows | 55,912 |
| `WEB` highest number | 56,012 |
| `WEB` distinct numbers | 55,906 |
| `WEBANULACION` series rows | 2,895 |
| `WEBANULACION` highest number | 2,895 |
| Duplicate codes | **8** |
| Rows whose code does not match its own number and date | **24** |
| Missing numbers in the `WEB` series | **106** |

The `WEBANULACION` series is perfectly contiguous and clean. The `WEB` series is not.

### Duplicate codes

Eight invoices share a code with another invoice. They come from two distinct defects.

**The numbering race**, which is `invoices::create()` taking the next number with
`select number ... order by number desc limit 1` inside the transaction and no lock. A plain
consistent read takes no lock, so two concurrent transactions read the same maximum:

```
8282  8104  WEB/16/08104  2016-08-01 18:02:11  user 79814  35.55
8283  8104  WEB/16/08104  2016-08-01 18:02:11  user 88707   3.949
```

Same shape at 18:03:04, 18:13:03 and 18:45:28 on the same evening, and again at
`2017-03-09 11:50:23` and `2015-08-01 14:52:19`.

**Note that four of the eight landed on the busiest day in the table.** The race fires under batch
load specifically, which is worth knowing before any bulk issuance ever runs through the new system.

**A number and code disagreement**, a different and older defect:

```
2254  number 2202  code WEB/13/02201
5898  number 5765  code WEB/15/05766
```

There are 24 rows in total where the code does not match its own number and date. Some may be
year-boundary cases; 24 is small enough to enumerate one by one.

### Missing numbers

106 numbers in the `WEB` series were never used: highest number 56,012 against 55,906 distinct
numbers. Most likely `Invoice::remove()`, which deletes invoice rows and whose own comment warns
that it breaks legal numbering. This is why Numbers v2 has no delete verb, see
[[api-contract#Verbs]].

### What to do about it

None of this is urgent, because all of it predates the chain and imports as inert history. But it is
about 138 rows that will want a written explanation before an inspection asks for one, and that
explanation is far easier to reconstruct now than in 2031.

## The repeated invoices incident, January 2026

Documented in the Litmind workbench under `issues/2026-01 Repeated invoices issue`. A PayPal IPN
service retrying against an HTTP 500 created multiple invoices for single payments. Customers were
charged only once. 88 invoices had to be annulled with negative counterparts, and it produced the
129-invoice spike on 2026-01-13.

This is the idempotency requirement in [[api-contract#Idempotency]] arriving eight months early and
in production. Under Verifactu the same event would have written duplicates into the chain.

## Old Numbers

The production instance is **complete and up to date and holds every invoice for every source**
(owner, 2026-09-06). It is the migration source of record. The figures below come from the
2025-08-24 backup committed at `setup/backup.sql`, so they are a snapshot, not the live totals.

From that backup: **54,516 invoice rows**, plus 1,504 expenses, 226 financial balances, 109
suppliers and 16 annotations.

By code shape:

| Shape | Rows | What it is |
|---|---|---|
| `WEB/...` | 51,716 | Litmind |
| `WEBANULACION/...` | 2,722 | Litmind rectificativas |
| other | **78** | The manual channel |

Two findings matter.

**Old Numbers is a near-complete mirror of Litmind, not a partial one.** 54,438 Litmind invoices at
2025-08-24 against roughly 54,300 in Litmind at the same date. So both systems hold essentially the
same invoices, and the migration is a reconciliation of two representations rather than a merge of
two disjoint sets. The natural key joining them is `source` + `code`.

**The manual channel is 78 invoices in twenty years.** Codes like `21001` and `20004` (a
year-prefixed sequence), source `0`, mostly monthly consulting invoices to a single client plus a
handful from 2004 to 2006. Numbers v2 will issue these itself through a simple creation flow, and
its series has to continue that year-prefixed sequence. See
[[decision-log#11. Numbers v2 issues the manual invoices itself]].

## Old Numbers holds data Litmind already anonymized

Litmind's `anonimizeInvoices()` blanks the customer name, address, NIF, phone and email on its own
rows and publishes `Users\Events\InvoicingRelatedDataAnonimized`. **Nothing subscribes to that
event and forwards it to Numbers.** Verified 2026-09-06: the only calls into the Numbers API
anywhere in Litmind are `Invoices\Commands\SendToNumbers` and a service-tool test method.

So every invoice Litmind has anonymized still carries its full customer identity in old Numbers.

Two consequences. It is a **live GDPR gap today**, independent of this project. And it is a
migration hazard: importing old Numbers verbatim would carry that data into the new system and undo
an erasure that was supposed to have happened. See
[[migration#Anonymization has to be applied during the migration, not after]].

## What old Numbers does not have

Fields that exist in Litmind's `invoices` and have no counterpart in old Numbers' schema:

| Field | Consequence |
|---|---|
| `anullation_invoice_id` | The rectificativa relationship for 2,722 `WEBANULACION` invoices exists only in Litmind. Verifactu needs it as `FacturasRectificadas`. |
| `is_anonimized`, `date_anonimized` | See above. |
| `number` | Derivable from the code. |
| `company_*` | Constant for the issuer, not a loss. |
| `stripeInvoiceId`, `stripePlanId`, `stripeSubscriptionId` | Only `stripeChargeId` crossed. Not needed for the fiscal record. |

## Amount precision

A migration trap. Litmind stores `subtotal`, `iva`, `irpf` and `total` as **`double`**. Old Numbers
stores `base`, `iva`, `irpf` and `final` as **`float`**, which is single precision and about seven
significant digits.

Consequences:

- For Litmind invoices, which exist in both systems, **take the amounts from Litmind**. Its double
  is the better-preserved value.
- For the 78 manual invoices, old Numbers is the only source and its float is what there is. At
  those magnitudes it is almost certainly exact, but verify each one rather than assuming: there are
  only 78.
- Numbers v2 must use **decimal**, not float and not double. Reconcile with explicit rounding on
  both sides, and prove any difference is representational rather than real.

## Related

- [[migration]]
- [[architecture]]
- [[api-contract]]
