# Architecture

> Decided 2026-09-06. See [[decision-log]] for the reasoning behind each choice and the options
> that were rejected.

## The shape

```
  Litmind          SaaS app 2        SaaS app 3        Manual invoices
     |                  |                 |             (Excel today)
     |  issue           |  issue          |  issue            |
     +------------------+-----------------+-------------------+
                                |
                                v
                        +---------------+
                        |  Numbers v2   |   the SIF
                        |               |
                        |  - assigns series + number
                        |  - chains the record, computes the huella
                        |  - renders the PDF with the QR
                        |  - submits to the AEAT
                        |  - owns retention
                        +---------------+
                          |            |
                          v            v
                        AEAT          S3 (PDFs, backups)
```

Everything that issues an invoice under the NIF calls Numbers v2. Numbers v2 is the only SIF, keeps
the only chain, holds the only certificate, and is the only place invoice data lives.

## Ownership boundary

The single most useful line in this design:

**Source applications own the commercial fact. Numbers v2 owns the fiscal document.**

Litmind keeps knowing that a charge succeeded and that a membership runs until a given date. That is
its `payments` table and its membership fields, and none of it depends on Numbers being reachable.
What Litmind stops owning is the invoice: the series, the number, the customer's fiscal identity,
the amounts as a fiscal record, and the PDF.

Those are already separate tables in Litmind, so the split is clean.

## No local copies

Source applications hold **no invoice data at all**. Not a mirror, not a cache, not a read model.
When Litmind needs to show a member their invoices, it asks Numbers v2 at that moment.

This was a deliberate choice over the alternative (each app keeping a local read model for
availability). The reasons:

- **GDPR becomes tractable.** One copy means one erasure point, one honest answer to a subject
  access request, and no possibility of a stale copy surviving an anonymization.
- **No state propagation.** With copies, every retention transition would have to be pushed to
  every source and applied there, and a missed push would be a silent retention breach. With no
  copies there is nothing to push.
- **The cost turned out to be small.** Only four places in Litmind read the `invoices` table
  outside its own module (the invoice admin, an admin per-user count, and two reads in the GDPR
  data package export). Nothing in `statistics` touches invoices, so no aggregate reporting was
  quietly depending on a local table. Revenue reporting can run off `payments`, which stays, and
  arguably should have been its source anyway.

What this costs, and what to design for, is in [[api-contract#Consequences of holding no local data]].

## Writes are asynchronous, reads are live

These are separate axes and conflating them is the mistake to avoid. Numbers v2 is authoritative
for issuance **and** issuance is asynchronous.

**Writes.** Litmind creates invoices inside a Stripe webhook, after the membership renewal side
effects have already committed. A blocking HTTP call there would mean that a Numbers outage takes
the money, grants the membership and produces no invoice. So the source queues an `IssueInvoice`
command on its own bus with a stable idempotency key, a worker calls Numbers v2, and the result is
whatever Numbers says. A Numbers outage delays invoices; it never loses them.

This also buys a clean cutover. See [[migration#Cutover]].

Two conditions make the "never loses them" true, and both are Litmind-side work: the command is
written through a **transactional outbox** in the same transaction as the payment record, because
the ServiceBus is documented as able to lose a published message silently; and a **reconciliation
job** asserts that every successful charge and PayPal transaction has exactly one invoice. See
[[review-2026-09-06#R2]].

**Reads.** Live calls, no caching to begin with. If latency proves to be a problem, a short-lived
non-durable response cache flushed on the erasure signal does not meaningfully reintroduce a copy.
Do not build it up front.

## The chain is a strict single-writer structure

Every record links to the previous one. Appends must be strictly serial per chain. Two concurrent
issuances that both read the same chain head produce two records claiming the same predecessor, and
the chain is silently broken from that point on, discoverable only at an inspection.

This constraint fights several instincts and has to be confronted explicitly in the design:

- **DDD.** The chain is an aggregate whose invariant spans all of its records. The usual advice
  (keep aggregates small) is not available. Do not model each invoice as its own aggregate with the
  chain as an eventually-consistent projection. The chain is the consistency boundary and it is a
  big one.
- **Kubernetes.** Do not autoscale the issuance path.
- **CQRS.** Fine on the read side. Dangerous anywhere it could reorder a write.

**At the actual volume this costs nothing.** See [[source-data-findings#Volume]]: peak load in
fifteen years is eleven invoices in one minute. A locking read on a chain-head row
(`SELECT ... FOR UPDATE`) *is* the serialization. No leader election, no distributed coordination,
no single-consumer queue partition, no pod affinity rules. Multiple replicas are safe because the
database orders them.

## Everything commits together

The threat to durability is not disk failure, it is the partial write. The invoice, its items, the
chain link, the idempotency record and the "needs submitting to the AEAT" entry either all commit or
none do. One relational database, one transaction, and a **transactional outbox** for the AEAT
submission and for outbound signals. "Write the invoice and publish an event" across two systems is
precisely where records disappear.

## Submission never blocks issuance

The AEAT will be down. It has maintenance windows, it rate limits, and it can return partial batch
outcomes. A record is valid the moment it is chained; sending it is a separate, retryable duty with
its own state machine (pending, sent, accepted, accepted with errors, rejected, permanently failed),
its own retries and its own alert path.

A submission outage should produce a growing visible backlog and a notification. It must never
produce a failed invoice.

## Rejecting is the safe default

A fiscal API is not ordinary CRUD. An accepted-but-wrong invoice is in the chain forever and can
only be undone with a *registro de anulación*, which is itself a permanent record. Therefore:

- Validate completely before anything is assigned, and **assign the series and number as late as
  possible in the transaction**, after every check has passed. A number burned on a request that
  then failed validation is a gap that has to be explained.
- Reject on missing or inconsistent tax-determination input rather than defaulting. Litmind's
  existing NULL-to-`TypeError` in its IVA getter is the cautionary version of a default nobody
  chose.
- Treat "no series configured for this source" as a hard refusal, never an auto-create.

## Related

- [[verifactu]] for what the regulation actually requires
- [[api-contract]] for the surface between sources and Numbers v2
- [[data-retention]] for who owns the clock and what signals flow inward
- [[deployment]] for Kubernetes, the database, backups and secrets
- [[migration]] for how to get from here to there
