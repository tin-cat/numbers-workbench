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

## 19. Argon2id for password storage, not PBKDF2

**Decided.** PBKDF2 is the FIPS-compliance answer, not the best-available one: it is cheap to
accelerate on GPUs. Argon2id is memory-hard, is OWASP's first recommendation, and PHP supports it
natively.

Starting parameters m = 19456 KiB, t = 2, p = 1, with `migrate_from` configured so raising them
later rehashes accounts on login rather than needing a reset. Fallback order if a host ever lacks
argon2: scrypt, then bcrypt.

Alongside it: login throttling, CSRF on every form (which Litmind's admin lacks and this must not
repeat), and no self-service password reset to start with. The second factor is decision 22.

Full detail in [[access-control]].

---

## 20. Named permissions, with `admin` implying all

**Decided.** No role hierarchy: with a handful of users it hides who can do what. Users hold named
capabilities; `admin` short-circuits in the voter rather than being stored as the full set, so a
permission added in 2028 is automatically held by admins.

**loren@tin.cat holds `admin`.** The first admin is created by an interactive console command at
deployment, never by a seeded password.

Two invariants that make `users` safe enough to grant: **a user cannot grant a permission they do
not hold** (without which `users` is silently equivalent to `admin`), and **the last admin cannot be
removed**, enforced in the domain rather than the form.

And the part that matters most: **no permission deletes an invoice, edits a chained record, changes
a series number or suppresses a submission.** The permission system controls access, never the
immutability guarantees. An "admin can force it" escape hatch would become the thing that breaks the
chain.

Full detail in [[access-control#The permission model]].

---

## 21. Server-rendered Twig with Symfony UX, Bootstrap 5 and Tabler

**Decided.** The interface is forms, tables and a few workflows, which is what server-rendered HTML
is best at. Turbo gives near-SPA navigation, Stimulus covers the interactive sprinkles, Tabler
supplies the sidebar, tables, tabs and form components already assembled.

**Rejected: a React or Vue SPA.** It would require exposing the whole domain through a second API
surface purely so the interface can draw a table, doubling the attack surface of a system whose job
is being trustworthy. The existing API is deliberately narrow, machine-facing and
per-source-credentialed.

**Rejected: Tailwind**, which ships no components, so an admin panel means building the whole chrome
by hand.

**Rejected: EasyAdmin**, on fit rather than quality. Issuance, annulment, partial rectification,
submission retry and the gestor export are workflows with domain rules, not entity forms, and
EasyAdmin becomes an obstacle once the screens stop being CRUD.

Assets through AssetMapper, so there is no Node toolchain in the container image.

Full detail in [[interface]].

---

## 22. SMS second factor via Twilio Verify, required for every user

**Decided.** Reusing the arrangement Litmind already runs: one Twilio account, one Verify service
per product so the friendly name in the message reads as Numbers. Verify rather than plain SMS,
because it needs no purchased sender number and no 10DLC registration.

**Stated once and then built as asked:** SMS is the weakest of the common second factors, and NIST
deprecated it as an out-of-band authenticator over SIM swap and SS7. It is the pragmatic choice here
because the integration exists and works. Recommendation, not a blocker: allow TOTP as an
*alternative* for anyone who wants it, which Twilio Verify also supports.

**Recovery is not optional.** There is one admin; a lost phone or a Twilio outage locks the fiscal
records away from everyone. So: single-use recovery codes generated at enrolment and stored hashed,
plus a break-glass console command that clears a second factor, audit logged.

Phone numbers are set at enrolment and are **never written into this repository**. A 2FA phone is an
authentication factor, and an authentication factor in git history is permanent.

Full detail in [[access-control#Two-factor authentication]].

---

## 23. Cloudflare Turnstile on every form, login included

**Decided.** Two things settled in advance because they are what causes trouble: the CSP must
explicitly allow `challenges.cloudflare.com` for both script and frame, and the behaviour when
Cloudflare's verification call fails must be configured rather than accidental.

Recommended split, **to confirm**: fail closed on login and anything reachable without a session,
where the control earns its place against credential stuffing; fail open with an alert on forms
inside the authenticated area, where the session and the CSRF token are the real controls and a
Cloudflare incident must not stop the accounts.

Full detail in [[access-control#Turnstile]].

---

## 24. A statistics dashboard as the home page

**Decided.** Panels of current position, yearly trends and period-over-period comparisons, gated on
`statistics.view`. Users without it land on their first permitted section rather than an empty page.
Charts with ApexCharts, which is what Tabler is built around.

The design content is in the traps, all specific to this data: **rectificativas must net rather than
add**, **currencies cannot be summed** so aggregates convert at each invoice's own stored rate,
**partial periods must compare the same day range** or seasonality reads as a trend, **money must
not become a JSON number** on the way to a chart, and aggregates happen in SQL rather than through
the ORM.

Reassuringly, retention does not break any of it: minimization strips identity, not amounts, so
fifteen years of trends stay correct.

Full detail in [[dashboard]].

---

## 25. Expense editing follows `expenses.add`, and suppliers become providers

**Decided (owner, 2026-09-06).** Two small ones settled together.

`expenses.add` covers **editing and deleting the entries a user created themselves**; amending
someone else's requires `admin`. So an expense records who entered it, and that field is what the
voter reads. This does not touch invoice immutability: an expense is a bookkeeping entry about
someone else's invoice, not a fiscal document we issued.

**Providers, not suppliers**, everywhere: interface, domain, database and API. Old Numbers' 109-row
`suppliers` table is renamed during the migration, along with the expense foreign key that points at
it, so the old term survives nowhere.

---

## 26. Deadline, Turnstile failure, and the export questions

**All decided by the owner, 2026-09-06.**

**The deadline is 1 July 2027**, the mandatory date for autónomos. Ten months from now. The schedule
built back from it is in [[roadmap#The schedule]], targeting a **March 2027 cutover** so there is
room to retreat.

**Turnstile fails closed, everywhere.** If the challenge cannot be verified the submission is
refused and says so, with the form contents preserved. This stacks two external dependencies
(Cloudflare and Twilio) on every login, so it comes with one emergency lever: an environment flag
disabling Turnstile, requiring shell or deploy access to set, loudly visible while active, and
audited. See [[access-control#Turnstile]].

**No period filter on the gestor export.** Reproduce the current all-time behaviour.

**USD stays unconverted in the export.** A known issue, handled outside the export. Worth a comment
in the exporter so the next reader does not "fix" it. See [[gestor-export#Currency]].

---

## 27. English everywhere in code, Spanish only at the boundaries

**Decided (owner, 2026-09-06).** Class names, properties, variables, database tables and columns,
API fields, configuration keys and enum cases are all English. Nothing Spanish is inherited from the
systems being replaced.

Three external contracts define their own vocabulary and are not ours to rename: **the AEAT XML**,
**Victor's export**, and **the old databases being read during migration**. Each is confined to its
adapter, which is what the hexagonal boundary is for. A Spanish identifier anywhere upstream of an
adapter is a bug.

So `TipoFactura` becomes a `CorrectionReason` enum with English cases and the R codes live only in
the serialiser, `huella` becomes `hash`, `IVA` becomes `vat`, `IRPF` becomes `withholding`.

The migration is also the moment to drop the typos both source schemas carry: `anullation_invoice_id`,
`date_emmited`, `is_anonimized`, `finantialBalances`.

Full mapping in [[naming]].

---

## 28. php-cs-fixer with `@Symfony`, minus the concatenation spacing

**Decided (owner, 2026-09-06).** The `@Symfony` ruleset, with `concat_space` overridden to `none`:
`"a".$b."c"`, never `"a" . $b . "c"`, matching the house style across the owner's other projects.

Only that rule is carried across. Indentation stays at `@Symfony`'s four spaces rather than
Litmind's tabs, so the rest of the tooling works unconfigured. Enforced in CI, not only locally.

Config in [[naming#Coding standard]].

---

## 29. Build VERI\*FACTU-only, and never add a non-submitting mode

**Decided 2026-09-06**, from the AEAT developer FAQ. A SIF that can only operate in VERI\*FACTU mode
is **not required to implement a registro de eventos**, and is **not required to verify the previous
record's chaining before generating each new one**. Both obligations attach only to systems that can
also run in "NO VERI\*FACTU" mode.

Both are cheap enough to do anyway as integrity checks, but as our choice rather than as
requirements. The decision that matters is the negative one: **do not add a non-submitting mode
later**, because it drags both obligations in with it.

---

## 30. Submission order: FIFO by choice, not by requirement

**Answered 2026-09-06.** The regulation requires records to be **generated** in the chronological
order the invoices are issued, which is the single-writer constraint already designed for. Nothing
found imposes an order on **submission**, and the FAQ explicitly treats a queue with periodic
retries as normal and unproblematic.

Keep per-chain FIFO on the outbox regardless. At fifteen invoices a day it costs nothing and it
retires the question permanently.

---

## 31. Duplicate prevention is structural, at three levels

**Decided 2026-09-06**, prompted by the owner's instruction to prevent duplicate invoicing "from the
very bottom of the system", and confirmed as necessary by the AEAT: a record is identified by
`Emisor` + `SerieYNúmeroFactura` + `FechaExpedición`, a second one is rejected as "Registro de
facturación duplicado", and **a number cannot be reused even after an annulment**.

1. **Serialised numbering**, by a locking read on the chain head.
2. **A unique constraint** on `(issuer, series, number)`, so the database refuses a duplicate even
   when the application is wrong.
3. **Idempotency on the caller's key**, so a retry returns the original invoice rather than issuing
   a second one.

The first two make duplicates impossible; the third makes them unnecessary. The third is the one
that would have prevented the January 2026 incident.

---

## 32. Corrections have five cases, not two

**Decided 2026-09-06**, from section 17 of the AEAT developer FAQ. This supersedes two earlier and
differently wrong framings in [[corrections]].

**The test for annulment is whether the operation was real**, not whether money moved and not
whether the document was delivered: "todas las facturas emitidas, en la medida en que respondan a
operaciones realmente efectuadas... no pueden anularse." Whether the invoice reached the customer is
a secondary consideration that supports annulment when it did not.

The fifth case is one we had missed entirely: **`alta de subsanación`**, for errors in internal
record fields that never appear on the printed invoice, such as a tax classification code. Also the
distinction between a record **rejected** by the AEAT (never exists there, correction carries
`Subsanacion = "S"` and `RechazoPrevio = "X"`) and one **accepted with errors** (stays wrong there
forever). Those two are different states and the submission state machine must not collapse them.

---

## 33. Tax determination is ported from Litmind, not invented

**Decided (owner, 2026-09-06).** Litmind's existing IVA and IRPF rules have been in production for
years and **have been checked by the gestor**. They are the specification.

Three inputs: country, province, and whether the customer has provided invoicing data (which is the
business-or-consumer test). Spain charges 21% to everyone with IRPF opt-in; 53 European countries
charge consumers only; 173 others never; the seven Canary Island provinces override Spain because
they sit outside the EU VAT area.

What remains is only the **classification codes**, because the existing system stores a percentage
and a 0% line can be not-subject, exempt, reverse charge or an export. That is now six rows and one
withholding question for the gestor rather than an open research problem.

Two defects to fix rather than port: the null-country `TypeError`, which becomes an explicit refusal
to issue, and the `??` versus `?:` inconsistency between the two getters that would silently ignore
a province setting IRPF to `0`.

Full detail in [[tax-determination]].

---

## Open

Everything answerable from the AEAT's own documentation has been answered, including the deadline
and the submission-order question. See
[[verifactu#Answered from the AEAT's own documentation]]. What is left falls into three groups.

### 1. For the gestor, in one conversation

Six questions, all concrete enough to answer in a single sitting.

**Corrections:**

1. **The rectificativa type mapping.** Expected answer is narrow: **R1 for essentially everything**
   (Art. 80.Uno LIVA), R4 for corrections of customer data, R2 and R3 out of scope. Plus **`I`, por
   diferencias, for every refund**. See [[corrections]].
2. **Whether a duplicate invoice that was delivered to the customer** is annulled or rectified. The
   operation was never real, which points to annulment; delivery points the other way. See
   [[corrections#The test for annulment is whether the operation was real]].

**Tax classification:**

3. **The six classification-code rows** in [[tax-determination#Mapping onto Verifactu]], plus how
   IRPF withholding is expressed in the record. Read FAQ section 23 on Canarias first.

**Foreign currency**, which is 111 invoices and rising:

4. **Must the tax figures be expressed in euros on a USD invoice?** Planned for either way, but it
   decides whether the EUR conversion is mandatory or merely useful. See
   [[money#The tax figures must also exist in euros]].
5. **Which exchange rate applies**, and on which date. Expected: the Banco de España or ECB
   reference rate on the fecha de devengo. This is stored permanently on the record, so it has to be
   right the first time. See [[money#The exchange rate is part of the fiscal record]].
6. **The rounding mode** for VAT computation. Expected: half up. See [[money#Rounding]].

### 2. Technical, at the moment the code is written

- **Argon2id parameters** against the OWASP guidance current at build time. The recorded m=19456,
  t=2, p=1 is a starting point and these numbers move. See [[access-control]].
- **The Symfony version.** The current major against the previous LTS; prefer the LTS for a system
  with this lifespan. See [[interface#Versions]].

### 3. Not blocking, and possibly never needed

- **Whether one obligado may run several SIFs**, each with its own chain. This was load-bearing
  under the rejected multi-SIF design and is now merely interesting. See [[verifactu]].
