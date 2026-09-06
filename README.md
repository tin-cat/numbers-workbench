# Numbers workbench

Notes, documentation and decisions for the **Numbers v2** project: the invoicing system of record
for every business operating under the NIF `ES 39898734J`.

The code lives in [numbers-2](https://github.com/tin-cat/numbers-2). This repository holds the
thinking behind it.

- **documentation** Architecture, the Verifactu contract, the API surface, retention policy,
  migration plan, findings from the existing data, the decision log and the project timeline.
- **infrastructure** Deployment, database, backups, security.

## Index

Read in this order the first time:

1. [[documentation/architecture|Architecture]], the shape of the system and why it is that shape.
2. [[documentation/verifactu|Verifactu]], the regulatory contract the whole design serves.
3. [[documentation/decision-log|Decision log]], every decision taken so far, with the options
   rejected and why.

The rest:

- [[documentation/api-contract|API contract]], the surface between sources and Numbers v2.
- [[documentation/corrections|Cancellations and refunds]], annulment against rectificativa, and who
  calls Stripe.
- [[documentation/money|Money]], how amounts are represented so they never drift.
- [[documentation/gestor-export|The gestor export]], the Excel contract with Victor.
- [[documentation/access-control|Users, authentication and permissions]], who gets in and what they
  can do.
- [[documentation/interface|The web interface]], the framework choices and the layout.
- [[documentation/dashboard|The dashboard]], the statistics home page and the traps in it.
- [[documentation/naming|Naming]], English everywhere, and the three boundaries where it is not.
- [[documentation/tax-determination|Tax determination]], the IVA and IRPF rules ported from Litmind,
  and the six-row mapping still owed to the gestor.
- [[documentation/data-retention|Data retention]], who owns the clock and what signals cross.
- [[documentation/migration|Migration]], the three migrations, reconciliation and cutover.
- [[documentation/source-data-findings|Findings in the existing data]], measured, and what the
  migration plan is built on.
- [[documentation/roadmap|Roadmap]], sequencing by what is actually compliance-critical.
- [[documentation/timeline|Timeline]].
- [[infrastructure/deployment|Deployment]], Kubernetes, the database, backups and secrets.

## What this project is, in one paragraph

Verifactu (RD 1007/2023) makes the software that issues an invoice a *Sistema Informático de
Facturación*, with legal duties: an unbroken hash chain across every record it issues, a QR on every
invoice, and submission of each record to the AEAT. Several applications currently issue invoices
under one NIF (Litmind, other SaaS products, and invoices written by hand in Excel). Making each of
them a compliant SIF would mean N chains, N certificates and N compliance surfaces to maintain
against a spec that will change. Instead there is **one SIF**: Numbers v2 issues everything, and the
applications become callers that hold no invoice data of their own.
