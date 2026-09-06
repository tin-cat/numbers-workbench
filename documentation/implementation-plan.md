# Implementation plan

From nothing to Numbers v2 running in production Kubernetes, receiving invoices from production
Litmind, with everything migrated from old Numbers, by a March 2027 cutover against a 1 July 2027
deadline. Written 2026-09-06 after the [[review-2026-09-06|review]]; it supersedes the earlier
[[roadmap]].

Two rules govern the whole plan.

**Every action on a production system is the owner's hand.** Numbers v2's production database, the
production Kubernetes cluster, production Litmind, production old Numbers, the AEAT production
service. The plan produces scripts, runbooks and exact commands; the owner runs them. This is not a
formality: it is the standing rule for this project, and the cutover runbook is written as a
checklist for a person, not as automation.

**Nothing reaches the AEAT production service until the cutover step that says so.** Every earlier
phase, on the development machine and in production alike, talks to the AEAT pre-production
environment (`prewww2.aeat.es`), which exists for this, requires the real certificate, and has no
fiscal effect. See [[review-2026-09-06#R1. Shadow mode as designed would issue real invoices]].

## Phases at a glance

| Phase | What | When | Exit criterion |
|---|---|---|---|
| 0 | Decisions and access | Sep 2026 | Gestor answers in hand, certificate obtained, pre-prod access working, cluster and database targets named |
| 1 | Litmind prerequisites | Sep to Oct 2026 | Issuance is a queued, outboxed, idempotent command; the three known defects are fixed; PayPal is first-class |
| 2 | **The full system on the development machine** | Oct to Dec 2026 | The owner issues, refunds, exports and logs in, end to end, and sees the records at the AEAT pre-production service |
| 3 | Migration tooling and rehearsals | Dec 2026 to Jan 2027 | Reconciliation report clean twice on fresh restores |
| 4 | Production infrastructure | Jan 2027 | Numbers v2 up in production, empty, against pre-prod, restore drill passed |
| 5 | Production rehearsal | Feb 2027 | Production Litmind traffic flows through Numbers v2 against pre-prod for a full renewal cycle |
| 6 | **Cutover** | early Mar 2027 | First real chained record accepted by the AEAT production service; reconciliation clean |
| 7 | Afterwards | Mar to Jun 2027 | Other sources, manual channel, the rest of the back office |

Phases 0 and 1 overlap. Phase 7 is the buffer: everything in it can slip without touching the
deadline, and the deadline should never be what the cutover date is measured against.

---

## Phase 0: decisions and access

Nothing here is code. All of it has lead time, and some of it gates later phases.

**With the gestor, one conversation.** Now the longest list, and the first item is the one that
changes the most:

1. **Are consumer invoices `F1` or `F2`?** 93% of invoices have no customer tax id and 53,199 of
   them go to customers in Spain. If they should be simplified invoices, the record omits the
   recipient block, the refund code for them becomes `R5`, and the PDF changes. See
   [[review-2026-09-06#R4]].
2. The six classification rows in [[tax-determination#The six rows]].
3. The rectificativa type mapping (expected R1 nearly always, R4 for data fixes).
4. Whether a delivered duplicate is annulled or rectified.
5. **Does voluntarily starting VERI\*FACTU submission bind us for the calendar year?** This decides
   whether rollback exists after cutover at all. See [[review-2026-09-06#R8]].
6. Whether EU VAT numbers must be validated for reverse charge (row 3).
7. Whether a zero-value supply needs an invoice.
8. Whether OSS is a consideration for EU consumers at this volume.

**Certificate.** Obtain the qualified certificate of the obligado tributario (or confirm the
existing one is usable for web-service authentication), note its expiry in an operational calendar,
and store a copy outside any cluster. Without it, nothing can be tested even against pre-prod:
"si se desea poder probar el SIF, deberá ser con el certificado electrónico cualificado válido"
(FAQ, section 11).

**AEAT pre-production access.** Confirm the certificate authenticates against `prewww2.aeat.es`
with a minimal hand-built submission. This is a one-hour spike and it de-risks the whole of
Phase 2.

**SIF identity.** Decide the values for the system's self-identification in every record: developer
NIF (the owner's), software name, id, version scheme, installation number. Trivial, but they are
permanent once used.

**Infrastructure targets.** Name the Kubernetes cluster, where the database will live (outside the
cluster, with PITR), the S3 bucket for PDFs and backups with versioning and object lock, and the
Sentry project.

**Read the code lists.** The permitted values for the classification fields, from the AEAT record
design spreadsheet, into the workbench, so the gestor conversation and the build use the same
source.

---

## Phase 1: Litmind prerequisites

All in Litmind, none dependent on Numbers v2 existing, all harder once a chain depends on them.
Each ships behind a flag where it changes behaviour, so production Litmind keeps issuing locally
throughout.

1. **Fix the numbering race** in `invoices::create()`: a locking read, and a unique index on
   `(series, number)`. See [[source-data-findings#Duplicate codes]].
2. **Retire `Invoice::remove()`.**
3. **Model refunds as corrections**: type, method, and a structured reference to the corrected
   invoice, replacing the untyped negative invoice.
4. **Make issuance a command.** `IssueInvoice` and `IssueCorrection` commands, written to a
   **transactional outbox table in the same transaction as the payment record**, relayed to the
   ServiceBus. The outbox is what closes the silent-loss window documented in the ServiceBus notes.
   See [[review-2026-09-06#R2]]. In this phase the command handler still calls the local
   `invoices::create()`; the transport changes, the authority does not.
5. **PayPal as a first-class source.** The PayPal transaction id as idempotency key, the IPN handler
   producing the same commands as the Stripe webhook, and the reconciliation job covering both. See
   [[review-2026-09-06#R3]].
6. **Require the customer tax id when invoicing data is supplied**, and validate its shape. What
   "supplied" means for consumers depends on the F1/F2 answer from Phase 0.
7. **Email attachment by invoice code**, not by local row id, so the pipeline can fetch from
   Numbers v2 later without a second change. See [[review-2026-09-06#R12]].
8. **Source identity**: distinct `NUMBERS_SOURCE` values per site, and a per-source credential in
   `.env`. `NUMBERS_IS_ACTIVE` enabled in development.
9. **The charge-to-invoice reconciliation job**, running against the local `invoices` table for
   now, so it exists and is trusted before it has to watch a remote system.
10. **Audit the other SaaS applications** for the same three defects. Fix as found.

Exit: production Litmind runs on the new code path, still issuing locally, with the outbox and the
reconciliation job live and quiet.

---

## Phase 2: the full system on the development machine

The phase the owner asked for by name, and the one that turns the design into something that can be
seen. Everything runs on this machine: Litmind's existing development stack, Numbers v2 alongside
it, and the AEAT pre-production service at the far end.

### What runs where

| Component | On the dev box | Talks to |
|---|---|---|
| Litmind dev stack | As today (`docker compose`, `*.devel` hosts) | Numbers v2 at `host.docker.internal:8088`, which is already the configured development endpoint |
| **Numbers v2** | New `docker compose` project: PHP, its own database, a mail catcher | AEAT **pre-production** `prewww2.aeat.es` with the real certificate |
| Stripe | Test mode, webhooks forwarded to the dev box | |
| PayPal | Sandbox | |
| Twilio Verify | The existing development behaviour: every code goes to one fixed phone | |
| Turnstile | Cloudflare's always-pass test site key | |
| AEAT | **Pre-production only.** Production endpoint not configured anywhere in this phase | |

The certificate on the development machine is the owner's real fiscal signing certificate. That is
acceptable on the owner's own box; it is worth saying out loud, and it should not be committed,
imaged, or copied anywhere else.

### Test data

The Litmind development database is a production snapshot. Test invoices issued from it would carry
real customer names and tax ids to the AEAT pre-production service. **Use synthetic test customers
for this phase**, created for the purpose, so that nothing about a real person leaves the machine
during rehearsal.

### What is built, in order

1. **Scaffold** Numbers v2: Symfony (current stable), hexagonal layout, php-cs-fixer with the
   concatenation override, the money value objects with their tests first.
2. **The chain**: `BillingRecord`, `InvoiceSeries`, the chain-head lock, the hash, the unique
   constraint. Tested with concurrent issuance before anything else is built on it.
3. **Issuance**: the source API with per-source credentials, `issueInvoice` with idempotency, tax
   determination ported from [[tax-determination]], series allocation, the refusal rules.
4. **The AEAT adapter**: record serialisation, signing, the **flow-controlled batching sender**
   honouring `TiempoEsperaEnvio` and the 1,000-record cap, the three response states kept distinct,
   `alta por rechazo` handling. Against pre-prod from the first test.
5. **PDF rendering** with the QR, the two-total layout for IRPF invoices, per-source templates.
6. **Corrections**: `issueCorrection` for the R-types and both methods, annulment, the same
   idempotency.
7. **Login, 2FA, permissions**, recovery codes, the break-glass command, Turnstile, the audit log.
8. **The manual invoice flow.**
9. **Read API**: `getInvoice`, `listInvoices`, `getPdf`, `getInvoiceByExternalReference`, with
   retention status in every response.
10. **The gestor export**, with its synthetic-fixture golden test.
11. **Retention signal receiver** (recording only; the executor is Phase 7).
12. **Sentry, metrics, the backlog alert.**
13. **Litmind wired through**: the Phase 1 command handler switched, on the dev box only, from local
    `create()` to calling Numbers v2; account pages and the data-package export reading live.

### Exit criterion: a checklist the owner runs by hand

- Log in with password and SMS code. Use a recovery code once.
- Trigger a Stripe test-mode membership payment in dev Litmind. Watch the command land in the outbox,
  the invoice appear in Numbers v2, the PDF render with its QR, the record reach the AEAT
  pre-production service and come back `Correcto`.
- Do the same through PayPal sandbox.
- Trigger a test-mode partial refund. See the rectificativa issued with the right type and method,
  referencing the original, submitted, accepted.
- Issue a manual invoice from the Numbers v2 interface.
- Force an AEAT rejection (a deliberately malformed record) and watch the state machine hold it as
  rejected, distinct from accepted-with-errors.
- Take Numbers v2 down, make a payment, bring it back, watch the queue drain and the invoice appear
  once.
- Generate the gestor export and compare it by eye against an old-Numbers export.
- Open the account page in dev Litmind and see the invoices, served live.
- Run the reconciliation job and read a clean report.

When every line passes, the design is proven and the remaining phases are about moving it, not
building it.

---

## Phase 3: migration tooling and rehearsals

Built and rehearsed **on the development machine**, against restored copies. Production data comes
to the dev box as backups the owner provides; the migration never reads a production system.

1. **The import**, reading old Numbers as source of record and Litmind for the four fields it alone
   holds: rectificativa links, amount precision, anonymization state, and **issuer identity per
   invoice** (see [[review-2026-09-06#R5]]). Idempotent, keyed on `source` + `code`, converting money
   through decimal strings, applying the 47 EUR currency decisions and the providers rename, and
   importing `expenses`, `providers`, `financial_balances` and `annotations` as well as invoices.
2. **The reconciliation report**, as specified in [[migration#Reconciliation is the deliverable]],
   extended to every table, plus the anonymization count check, the issuer-identity distribution, and
   the per-invoice `TaxBreakdown` invariant.
3. **The high-water-mark derivation**, per series, with its independent cross-check.
4. **Rehearse** on fresh restores until the report is clean twice in a row, with the exception list
   (8 duplicate codes, 24 mismatches, 106 gaps, 6,343 malformed-NIF invoices, 47 currency-less
   invoices) enumerated and explained in a document that will outlive the migration.
5. **Re-render a sample of historical PDFs** and compare them against Litmind's renderer for the
   same invoices, including one from each of the ten issuer identities.

Exit: a runbook with exact commands, timed, and a report format the owner has read and approved.

---

## Phase 4: production infrastructure

Numbers v2 up in production, **empty**, and still pointed at AEAT pre-production.

1. Kubernetes manifests: two replicas, health checks, rolling deploys, no autoscaling on the writer.
2. The database outside the cluster, with PITR enabled and verified.
3. S3 for PDFs and backups, versioning and object lock on, backup job scheduled.
4. **A restore drill, performed and timed.** Not scheduled for later; performed.
5. Secrets: the certificate via sealed or external secrets with a KMS, source credentials, Twilio,
   Turnstile. etcd encryption at rest confirmed.
6. Sentry project, Prometheus scrape, the backlog alert wired to something a person sees.
7. The first admin created by the console command, by the owner, on the production pod.
8. **Nothing configured for the AEAT production endpoint yet.** The value does not exist in
   production configuration until Phase 6.

Owner's hand throughout. The plan produces manifests and a runbook.

---

## Phase 5: production rehearsal

Real production traffic through the real production Numbers v2, with no fiscal effect, for long
enough to trust it.

Production Litmind keeps issuing locally, as the authority. **Additionally**, the outbox relays every
`IssueInvoice` and `IssueCorrection` to production Numbers v2, which issues into a **rehearsal
chain** and submits to **pre-production**. Every real charge, renewal and refund of February 2027
produces a matching rehearsal record.

What is watched: that every local invoice has a rehearsal twin with identical amounts (a third
reconciliation, temporary), that the AEAT accepts them, that the backlog never grows, that a
renewal cycle, a refund, a PayPal payment, a USD invoice and an IRPF invoice have all been seen.

Exit: a full month clean. **The rehearsal chain and its records are then wiped**, entirely, before
Phase 6. The first real record chains to nothing.

---

## Phase 6: cutover

The short downtime, precisely.

**What is down.** Nothing customer-facing. Litmind keeps taking payments; invoice emails are
delayed by the length of the window. Old Numbers becomes read-only and stays so forever. The window
is the time between pausing Litmind's issuance consumer and resuming it against Numbers v2, and at
58,807 rows the migration itself runs in minutes; the hour is verification.

**Timing.** A quiet weekday morning in early March 2027, Spanish time, with the owner at the
keyboard and the whole day clear. Not a Friday, not the end of a quarter.

**Runbook, every step the owner's:**

| Step | Action | Check before proceeding |
|---|---|---|
| T-7d | Final Phase 3 rehearsal against fresh backups | Report clean |
| T-1d | Confirm the Phase 5 rehearsal chain has been wiped from production Numbers v2 | Chain empty, zero records |
| 1 | Pause Litmind's issuance consumer | Charges still succeeding; outbox growing |
| 2 | Freeze old Numbers read-only; take its final backup | Backup verified restorable |
| 3 | Take the Litmind invoices snapshot | Row count matches expectation |
| 4 | Run the import into production Numbers v2 | **Reconciliation report clean.** If not, stop here: nothing has changed for customers, resume the consumer, retry another day |
| 5 | Confirm the high-water marks per series; enable issuance on them | Numbers v2 refuses issuance until this is done, by design |
| 6 | Switch the AEAT endpoint to production; set the SIF installation number | A dry-run signature succeeds |
| 7 | Flip Litmind: command handler becomes authoritative, local `create()` disabled | Flag confirmed in production config |
| 8 | Resume the consumer | Outbox drains; **the first real record is issued with `PrimerRegistro: S`** |
| 9 | Verify the first invoice by scanning its QR against the AEAT cotejo service | AEAT shows the invoice |
| 10 | Run the reconciliation: every charge since step 1 has exactly one invoice | Clean |
| 11 | Run the gestor export; send to Victor with a note that the system changed | |

**Rollback.** Up to step 7, trivial: resume the consumer, nothing has changed. After step 8, the
first real record has been submitted. If the answer to Phase 0 question 5 is that early submission
binds for the year, **there is no rollback from here**, only forward-fix, and the runbook should say
so on the page. If it does not bind, the local path stays behind its flag until 1 July 2027 and
resuming it requires the high-water mark from Numbers v2.

---

## Phase 7: afterwards

In the buffer, none of it deadline-critical:

- The other SaaS applications, one at a time, each through its own short Phase 5 and Phase 6.
- The manual invoice channel goes live; Excel is retired.
- The invoice admin moves from Litmind to Numbers v2; Litmind's `invoices` tables and classes are
  removed.
- The dashboard.
- The retention executor (the receiver has been recording since Phase 2).
- The remaining back office: expenses, providers, audit viewer, user management.
- Old Numbers: kept, read-only, indefinitely. Decide separately whether to purge the anonymized
  customer data still sitting in it.

---

## Questions that gate a phase

| Question | Gates |
|---|---|
| F1 or F2 for consumer invoices | Phase 2 record shape, corrections, PDF |
| The six classification rows | Phase 2 tax determination |
| Does early submission bind for the year | Phase 6 rollback wording, and possibly the cutover date |
| Certificate obtained and pre-prod access confirmed | All of Phase 2 |
| Cluster and database targets named | Phase 4 |

Everything else in the plan can proceed while these are open.

## Related

- [[review-2026-09-06]] for why each of these steps exists
- [[migration]] for the reconciliation detail
- [[verifactu]] for what the AEAT actually requires
- [[decision-log]]
