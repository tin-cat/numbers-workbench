# Roadmap

Sequencing, driven by what is actually compliance-critical rather than by what is interesting.

> The deadline itself is unconfirmed. It has already moved once, by RD 254/2025, and the date for
> personas físicas sat later than the one for sociedades. Confirming it is the first task, because
> it sets everything below and it closes the rollback window. See
> [[verifactu#Open questions to confirm with the AEAT]].

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
- Backups with PITR, and one rehearsed restore.

## Can land afterwards without compliance risk

- The other SaaS applications, one at a time.
- The invoice admin, moved across from Litmind.
- **The retention executor.** This is worth stating plainly: the clock is five fiscal exercises, so
  the first invoice actually due for anonymization is years away. Build the **signal receiver** from
  day one, because cancellation and erasure dates are cheap to record and lossy to reconstruct
  later, and defer the job that acts on them.

## Cutover order

Shadow mode first, then flip authority one source at a time, Litmind first, manual channel last. See
[[migration#Cutover]].

Cut over well before the deadline. Afterwards, rollback stops being available and everything becomes
forward-fix.
