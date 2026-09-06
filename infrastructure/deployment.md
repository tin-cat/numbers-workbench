# Deployment and infrastructure

## Stack

PHP / Symfony, DDD and hexagonal architecture, deployed to Kubernetes.

Hexagonal is a genuine fit here rather than a habit: the AEAT client, the PDF renderer, the object
storage and the per-source adapters are all real ports, each with an obvious test double.

Two traps to watch for, both cases where architectural purity would fight the domain:

- **The chain aggregate.** Its invariant spans every record it contains, and the usual advice to
  keep aggregates small is not available. See
  [[architecture#The chain is a strict single-writer structure]].
- **Retention as eventual consistency.** It is a deterministic scheduled job with a legal clock, not
  a saga. See [[data-retention]].

## Kubernetes, and what does not go in it

Kubernetes is here for its **deployment process and portability**, not for scale. At fifteen
invoices a day there is no scaling problem to solve. Rolling deploys, health checks, declarative
configuration, secrets handling and the ability to stand the whole thing up on a different cluster
from a manifest are the reasons, and they are good ones.

**The database stays outside the cluster.** This follows directly from the portability goal rather
than contradicting it: stateless workloads move between clusters trivially, stateful ones do not. If
the database runs as a StatefulSet on PVCs then a cluster migration stops being a redeploy and
becomes a data migration of the fiscal system of record, which is the operation you least want to
perform casually.

So: application in Kubernetes, database on managed infrastructure or a dedicated host with
replication and PITR, PDFs and backups in S3. A cluster move is then a redeploy plus a connection
string.

At this volume the database instance can be the smallest the provider sells and still be idle.

**Two replicas** for zero-downtime rolling deploys, not for throughput. This is safe without any
distributed coordination because the chain-head lock in the database provides the serialization.

## Durability

The threat is the partial write, not disk failure. Everything commits in one transaction, with a
transactional outbox for AEAT submission and outbound signals. See
[[architecture#Everything commits together]].

Backups, in order of what actually matters:

1. **Point-in-time recovery**, not only nightly dumps. Losing a day means losing chain links that
   cannot be re-derived from anywhere.
2. **Immutable storage**: S3 with versioning and object lock, so a bad script or a compromised
   credential cannot erase history.
3. **Scheduled restore drills.** This is the real deliverable. A backup that has never been restored
   is a hypothesis, and this is a compliance control rather than an ops preference.

At this data size a full dump is megabytes, so restores take seconds and drills can genuinely run on
a schedule.

One free property worth knowing: everything the AEAT has accepted exists in a second place outside
our control. That covers the records but not the PDFs and not the not-yet-submitted tail.

## Secrets

Two crown jewels, and they fail differently.

**The certificate** signs on behalf of the NIF, so its compromise is a fiscal problem rather than
only a data one. Not in an image, not in a plain environment variable. A plain Kubernetes Secret is
base64 in etcd, which is not encryption, so pair it with sealed-secrets or external-secrets backed by
a KMS, or at minimum turn on etcd encryption at rest. KMS or HSM-backed signing if it can be
arranged. Tight RBAC, and an audit trail of every use.

**The source credentials** are the other one. A compromised source can issue invoices under the NIF.
Per-source, rotatable, scoped so a source can only touch its own invoices and its own series, with
an issuance audit log recording which caller asked. See
[[api-contract#Security]].

## What is deliberately not needed

Worth recording, so nobody adds it later out of habit. At fifteen invoices a day, with a peak of
eleven in one minute across fifteen years:

- No sharding, no partitioning, no read replicas.
- No event sourcing, no CQRS read model.
- No message broker. The AEAT outbox is a table and a polling worker.
- No leader election or pod affinity. The database lock is the serialization.
- No autoscaling on the issuance path, which must not scale horizontally in any case.

The entire dataset fits comfortably in memory.
