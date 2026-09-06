# Verifactu

The regulatory contract the whole design serves. RD 1007/2023, developed by Orden HAC/1177/2024.

> **Verify the specifics before building against them.** This document records our working
> understanding as of 2026-09-06, not legal advice. The points marked **CONFIRM** are load-bearing
> assumptions that must be checked against AEAT's own published FAQ and technical specifications,
> and the retention periods should be put in front of the gestor once.

## What the regulation makes of us

Software that issues invoices is a *Sistema Informático de Facturación* (SIF). A SIF must:

1. **Generate a billing record at the moment of issuance** (*registro de facturación de alta*),
   holding the invoice identity (issuer NIF, serie and número, fecha de expedición), the invoice
   type, the description of the operation, the tax breakdown, the totals, a timestamp, and an
   identification of the SIF itself.
2. **Chain each record to the previous one.** Each record carries the huella of its predecessor.
   The very first record in a chain is flagged `PrimerRegistro: S`.
3. **Never modify a record.** Corrections are made with new records: a *registro de anulación* to
   annul, and a rectificativa (types R1 to R5) to correct.
4. **Put a QR code on the invoice**, encoding an AEAT verification URL carrying the NIF, número,
   fecha and importe, together with the VERI\*FACTU legend where operating in that mode.
5. **Submit each record to the AEAT** when operating in Verifactu mode. The alternative mode (no
   submission) trades the web service for considerably stricter local integrity, signature and
   event-logging duties. Verifactu mode is the simpler of the two and is what we are building.
6. **Identify itself in every record**: developer NIF, software name, ID, version and installation
   number. Because this is own-use software written by the taxpayer, the *declaración responsable*
   is on us, not on a vendor, and the "used by multiple obligados" flag is `N`.

## Why this produced a single SIF

**CONFIRM:** a taxpayer may run several SIFs under one NIF, each keeping its own independent chain,
provided the invoice series do not collide between them.

That option was available and was rejected. With five applications issuing under one NIF, being
compliant five times over means five chains to verify, five certificates on five hosts, five
*declaraciones responsables*, and five deploys every time the AEAT changes the specification, which
it will. One SIF collapses that to one of each. It also makes the CONFIRM above stop mattering,
which is worth something on its own.

Series management then becomes internal to Numbers v2 rather than a registry that several projects
have to keep in sync, and collisions become impossible by construction rather than prevented by
someone remembering.

## Why retention had to move to the SIF

The sharp reason, and the one that turned this from a preference into a requirement:

**The huella covers the record's contents.** Litmind's `anonimizeInvoices()` blanks `customer_name`,
`customer_dni_nif_cif`, the Stripe ids and `invoices_items.title` (which is the
`DescripcionOperacion`). Do that to a chained record and it no longer hashes to its own huella. The
chain still verifies structurally, but the ability to prove any of it is permanently gone.

Retention has to live where the chain lives, or the retention process quietly destroys the
evidentiary value of the thing it was conserving. See [[data-retention]].

## Historical invoices are not Verifactu records

Invoices issued before the SIF goes live have no huella, were never submitted, and **must not be
given one**. Retro-generating records for old invoices would be manufacturing attestations about
when they were issued.

So Numbers v2 holds two classes of invoice:

| | Historical | Chained |
|---|---|---|
| Origin | Migrated from Litmind and old Numbers | Issued by Numbers v2 after go-live |
| Chain | Not a participant | Linked, with a huella |
| QR | None | Yes |
| AEAT | Never submitted | Submitted |
| Mutability | Read-only | Immutable, corrected only by new records |

The chain begins with `PrimerRegistro: S` on the first invoice issued after go-live. Everything
before that date is inert data.

This is what makes the migration low-risk: nothing imported can corrupt a chain, because none of it
is in one. See [[migration]].

## Numbering bridges the two classes

The one place where the two classes must connect. The first chained invoice in a series continues
from the last historical one: Litmind is currently at `WEB/26/nnnnn` with the number in the
fifty-six thousands, and Numbers v2 must continue that sequence rather than restart it.

That per-series high-water mark is the single value where an error is unrecoverable. Too low
duplicates a number under the NIF; too high creates a gap that cannot be explained. Numbers v2 must
**refuse to issue in any series whose high-water mark was not explicitly established**.

## Series inventory

One namespace per NIF. Every source gets its own distinct series, allocated by Numbers v2.

| Source | Series | Notes |
|---|---|---|
| Litmind | `WEB` | Shared by litmind, photolancers and starring, all one issuer, one sequence |
| Litmind rectificativas | `WEBANULACION` | Its own independent sequence |
| Other SaaS apps | to allocate | One per app, must not collide |
| Manual invoices | to allocate | Issued by Numbers v2 through its own creation flow. The existing codes are a year-prefixed sequence (`21001`), which the new series must continue |

## Open questions to confirm with the AEAT

- **CONFIRM** multiple SIFs per obligado, each with its own chain. Not load-bearing any more given
  the single-SIF decision, but worth knowing.
- **CONFIRM** whether records must be *submitted* in chain order, or whether the AEAT accepts them
  in any order and validates the chain later. If order is required, the outbox needs per-chain FIFO
  rather than a plain queue.
- ~~The deadline~~ **Answered: 1 July 2027**, the mandatory date for autónomos. See
  [[roadmap#The schedule]]. This is also the date the rollback option expires, see
  [[migration#Rollback]].
- **CONFIRM** the mapping of our tax cases (Spanish B2C, Spanish B2B, EU reverse charge, non-EU) onto
  `CalificacionOperacion`, `OperacionExenta` and `ClaveRegimen`.

## Related

- [[architecture]]
- [[api-contract]]
- [[data-retention]]
- [[migration]]
