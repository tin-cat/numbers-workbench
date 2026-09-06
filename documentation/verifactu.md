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

## Answered from the AEAT's own documentation

Researched 2026-09-06 against the AEAT developer FAQ ("Aclaraciones a dudas de los desarrolladores",
version 1.3, 4 December 2025) at
`sede.agenciatributaria.gob.es/static_files/AEAT_Desarrolladores/EEDD/IVA/VERI-FACTU/FAQs-Desarrolladores.pdf`.

### The deadline, with its legal source

**1 January 2027** for those filing Impuesto de Sociedades, **1 July 2027** for everyone else, which
is us. The dates moved from 2026 by **Real Decreto-ley 15/2025 of 2 December**, which is what
version 1.3 of the FAQ was published to reflect.

The AEAT actively encourages adopting early: a compliant system may be used with full fiscal
validity today, and they recommend it "para evitar apuros de tiempo y problemas de última hora".

### Order: the requirement is on generation, not submission

The obligation is that records are **generated** in the chronological order the invoices are issued:

> basta con que el sistema SOLO admita la generación de los RFs por el orden cronológico en que se
> encolen, o sea, que la generación de los RF se produzca por la misma secuencia cronológica de
> facturación

That is exactly the single-writer constraint in
[[architecture#The chain is a strict single-writer structure]], now confirmed as a requirement
rather than an inference.

**Nothing found imposes an order on submission.** Queueing with retries is explicitly normal:

> los RF quedarían "encolados", pendientes de remisión, con reintentos periódicos, como si se
> tratara de una incidencia, sin que ello suponga ningún problema

**Decision: keep per-chain FIFO on the outbox anyway.** At fifteen invoices a day it costs nothing,
and it removes a question that would otherwise need re-asking every time the queue misbehaves.

### Building VERI\*FACTU-only removes two obligations

A significant and easily-missed scope reduction. A SIF that can **only** operate in VERI\*FACTU mode
("SOLO VERI\*FACTU"), as opposed to a "DUAL" one that can also run in the non-submitting mode:

- **Is not required to implement a registro de eventos.** The event log is obligatory only for
  systems that can operate in "NO VERI\*FACTU" mode.
- **Is not required to verify the previous record's chaining before generating each new one.** That
  pre-flight check is "SIEMPRE OBLIGATORIA" only for "NO VERI\*FACTU".

Both are worth doing anyway as cheap integrity checks, but they are ours to schedule rather than
requirements to satisfy. **Build VERI\*FACTU-only, and do not add a non-submitting mode**, because
adding one later drags both obligations in with it.

### Record identity, and why a number can never be reused

A record is identified by **`Emisor` + `SerieYNúmeroFactura` + `FechaExpedición`**. A second record
with the same identity is rejected with **"Registro de facturación duplicado"**, and this holds even
after an annulment: the old practice of deleting an invoice and reusing its number is explicitly
dead.

See [[corrections#Preventing it at the bottom]] for the three structural defences this implies.

## Still open with the AEAT or the gestor

- **CONFIRM** multiple SIFs per obligado, each with its own chain. Not load-bearing any more given
  the single-SIF decision, but worth knowing.
- **The Verifactu classification codes** for our tax cases. No longer open-ended: the existing rules
  are documented in [[tax-determination]] and the mapping is now six rows and one withholding
  question for the gestor.

## Related

- [[architecture]]
- [[api-contract]]
- [[data-retention]]
- [[migration]]
