# Roadmap

Sequencing, driven by what is actually compliance-critical rather than by what is interesting.

## The schedule

**The mandatory date for autónomos is 1 July 2027.** From today, 6 September 2026, that is
**roughly ten months**.

Working backwards, with the constraint that the cutover should land well before the deadline because
that is when rollback stops being available (see [[migration#Rollback]]):

| Target | What |
|---|---|
| Now, in parallel | The Litmind-side prerequisites below. They depend on nothing and get harder later. |
| Oct to Dec 2026 | The core build: issuance, chain, huella, QR, PDF, AEAT submission, login and permissions, the manual invoice flow. |
| Dec 2026 | Migration rehearsals against restored snapshots, until the reconciliation report is clean twice running. |
| Jan to Feb 2027 | Shadow mode on Litmind, long enough to see a renewal cycle, a refund and a foreign-currency invoice. |
| **Mar 2027** | **Cutover.** Litmind first, then the other applications, manual channel last. |
| Mar to Jun 2027 | Buffer. Everything deferred below, and room for the cutover to slip without touching the deadline. |

Ten months is workable but not generous for the scope in [[architecture]], and the buffer is the
part to protect. A cutover in June 2027 would technically meet the date while leaving no room to
retreat if something is wrong, which is the worst place to be.

## Before anything is built

1. **Confirm the deadline** and the AEAT questions in [[verifactu]].
2. **Fix the numbering race** in Litmind's `invoices::create()`, and in the other applications if
   they share its lineage. It is currently a cosmetic duplicate; under Verifactu it is a rejected
   submission and a compromised chain. See [[source-data-findings#Duplicate codes]].
3. **Model refunds as rectificativas** rather than negative invoices, in each application. Needs
   `TipoFactura` R1 to R5 and a reference to the invoice being corrected.
4. **Retire `Invoice::remove()`** and its equivalents.
5. **Fix the source identity problems**: `NUMBERS_SOURCE` is `1` in two configs, and `sourceId`
   carries a user id. Nothing can be keyed on them until this is sorted.
6. **Decide what to do about the anonymized data still sitting in old Numbers**, which is a live gap
   regardless of this project. See
   [[source-data-findings#Old Numbers holds data Litmind already anonymized]].

These are prerequisites, not cleanups. They are also the same shape in every application, so audit
them all in one pass.

## Must exist by the deadline

- Issuance, the chain, the huella.
- The QR and the PDF.
- AEAT submission with its state machine and retries.
- Idempotency, properly, with stored responses.
- **One source integrated end to end**, which should be Litmind, because it is the one that can be
  seen and tested here.
- The migration of history, with a clean reconciliation report.
- **The manual invoice creation flow.** Not deferrable: a consultancy invoice issued after the
  deadline is issued under the same NIF and must be a Verifactu record like any other.
- **The cancellation and refund flows**, emitting annulments and rectificativas correctly. Refunds
  do not stop happening because a deadline passed. See [[corrections]].
- **The gestor export**, reproducing the existing workbook exactly, with its golden-file test. The
  accounts still have to reach Victor. See [[gestor-export]].
- **Login, two-factor, the permission model and the screens the manual invoice flow needs.** Not the
  whole back office, but enough of it: nobody can issue a manual invoice without an interface to do
  it in. Recovery codes and the break-glass command ship with the second factor, not after it. See
  [[access-control]] and [[interface]].
- Backups with PITR, and one rehearsed restore.

## Can land afterwards without compliance risk

- The other SaaS applications, one at a time.
- The invoice admin, moved across from Litmind.
- The rest of the back office: expenses, providers, the audit log viewer, user management screens
  beyond what the first admin needs.
- **The dashboard.** It is the home page, but nothing fiscal depends on it and it is the easiest
  thing to build well once the aggregates have real data behind them. See [[dashboard]].
- **The retention executor.** This is worth stating plainly: the clock is five fiscal exercises, so
  the first invoice actually due for anonymization is years away. Build the **signal receiver** from
  day one, because cancellation and erasure dates are cheap to record and lossy to reconstruct
  later, and defer the job that acts on them.

## Cutover order

Shadow mode first, then flip authority one source at a time, Litmind first, manual channel last. See
[[migration#Cutover]].

Cut over well before the deadline. Afterwards, rollback stops being available and everything becomes
forward-fix.
