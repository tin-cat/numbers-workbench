# Tax determination

How IVA and IRPF are decided for an invoice. Ported from Litmind, where the rules have been in
production for years and **have been checked by the gestor** (owner, 2026-09-06).

This is the answer to what was previously an open question. The task is not to invent tax rules: it
is to **port these exactly** and add the Verifactu classification codes on top of them.

## The rules as they exist

Three inputs decide everything: the customer's **country**, their **province**, and whether they
have **provided invoicing data**.

### Where the country and province come from

If the customer has invoicing data, the invoicing country and province. Otherwise the profile
country and province. A province setting supersedes the country's where present.

### Whether the customer counts as a business

`isHasInvoicingData()` is the proxy. A customer who has filled in invoicing details (name, address,
fiscal id) is treated as a business; one who has not is treated as a consumer.

This is worth naming explicitly because it is doing real fiscal work while looking like a
convenience check.

### IVA

Three methods, held per country and overridable per province:

| Method | Meaning | Percentage applied |
|---|---|---|
| `0` do not apply | Never any IVA | 0% |
| `1` apply to individuals | IVA only for consumers | 21% if **no** invoicing data, otherwise 0% |
| `2` apply to individuals and companies | Always | 21% |

### IRPF

Two methods:

| Method | Meaning | Percentage applied |
|---|---|---|
| `0` do not apply | Never | 0% |
| `1` apply voluntarily | The customer may choose | 15% **only if** they have set `invoicing_is_irpf` |

So IRPF is opt-in by the customer, in the jurisdictions that allow the choice. That is the
"they can even request to have IRPF" behaviour.

## How the world is actually divided

Measured against the live country and province tables:

| Group | Countries | IVA | IRPF | Effect |
|---|---|---|---|---|
| **Spain** | 1 | `2` always | `1` voluntary | 21% to everyone; 15% withholding if the customer opts in |
| **Europe and neighbours** | 53 | `1` individuals | `0` | 21% to consumers, 0% to businesses |
| **Rest of the world** | 173 | `0` | `0` | No IVA, ever |

Plus one province-level override, and it is the interesting one:

| Provinces | Override |
|---|---|
| Gran Canaria, Tenerife, Fuerteventura, Lanzarote, Gomera, La Palma, Hierro | IVA method `1` instead of Spain's `2` |

The Canary Islands sit outside the EU VAT area, so a Canary business is not charged Spanish IVA.
IRPF is not overridden there, so it falls through to Spain's voluntary setting.

## IRPF is not in the record at all

**Answered from the AEAT developer FAQ, section 20.** This confirms the owner's instinct that the
existing systems already answer the withholding question, but it has a consequence worth
understanding.

> En conclusión la retención a cuenta del IRPF o IS que vaya en factura, **no se incluirá en el
> registro de facturación**, ya que no es uno de los elementos constitutivos de la factura de
> acuerdo con la Directiva UE y el art 6 del Reglamento de Obligaciones de Facturación.

IRPF appears on the printed invoice and nowhere in the Verifactu record. The record covers the
indirect tax (IVA, or IGIC, or IPSI) and nothing else.

### The consequence: two different totals

The record's `ImporteTotal` is defined as the sum of taxable base plus tax charged:

> ImporteTotal - Se validará que sea igual a Ʃ (BaseImponibleOimporteNoSujeto + CuotaRepercutida +
> CuotaRecargoEquivalencia) de todas las líneas de detalle de desglose

**It does not subtract the withholding.** So for any invoice carrying IRPF:

| | |
|---|---|
| **Total factura** (`ImporteTotal`, and what the QR carries) | base + VAT |
| **Total a pagar** (what the customer actually pays) | base + VAT − withholding |

Litmind's `invoices.total` column holds the **second** of these. It is what was charged. It is
**not** the Verifactu `ImporteTotal`, and mapping it straight across would be wrong.

**Scale of it:** 190 of Litmind's 58,807 invoices carry IRPF, 317.54 EUR of withholding in total,
across 25 customers who have elected it. Small, ongoing, and exactly the kind of thing that produces
a handful of unexplainable records years later if it is got wrong now.

### The invoice PDF must show both

The AEAT recommends it explicitly, because a customer who scans the QR will otherwise see a number
that does not match what they paid:

> el sistema informático de facturación (SIF) podría incluir en la factura ambos importes, "Importe
> total factura" (que es el que aparece en el QR tributario) y "Total a pagar", debidamente
> diferenciados

So the PDF template needs two clearly labelled lines, not one. This is a concrete requirement on the
renderer that Litmind's current template does not have.

There is a **±10.00 EUR tolerance** on the `ImporteTotal` validation, returning a warning rather
than a rejection, and it does not apply for `ClaveRegimen` 03, 05, 06 or 09.

## Mapping onto Verifactu: the six rows

This is the part that still needs the gestor. It is one table, not a research project.

### Why a percentage is not enough

The existing system stores a **percentage**. Verifactu wants a **classification**, and 21% or 0% does
not determine it. Three fields carry the classification:

**`CalificacionOperacion`** answers: is this operation subject to VAT, and if so, who accounts for
it? The distinctions it draws are between an operation that is subject and not exempt with the
issuer charging the tax; one that is subject but where **the recipient** accounts for it (reverse
charge, *inversión del sujeto pasivo*); and one that is **not subject** at all, either by an article
of the law or because the place-of-supply rules put it outside Spanish VAT.

**`OperacionExenta`** applies when an operation is subject but **exempt**, and says on what grounds:
an ordinary exemption, an export, an intra-EU supply, and so on. It is used *instead of* charging
tax, not alongside.

**`ClaveRegimen`** says which VAT regime the operation falls under: the general regime, or one of
the special ones (cash basis, travel agencies, used goods, OSS/IOSS, and others).

**A 0% line in our data could be any of: not subject by localisation, exempt as an export, or
reverse charge.** Those are three different records with three different codes, and the current
database cannot tell them apart because it only ever stored the zero. That is why this needs
answering per group rather than deriving.

### The six rows

| # | Case | Today | The question |
|---|---|---|---|
| 1 | **Spain**, any customer | 21% | Expected the plainest case: subject, not exempt, general regime, issuer charges the tax. Confirm. |
| 2 | **Europe**, consumer | 21% | We charge Spanish VAT to an EU consumer. Confirm this is the same classification as row 1, and whether OSS applies to these at our volume. |
| 3 | **Europe**, business | 0% | The important one. Is this **reverse charge** (subject, recipient accounts) or **not subject by place-of-supply rules**? Different codes, and the answer probably depends on whether the customer's VAT number was validated. |
| 4 | **Rest of world**, any customer | 0% | **Not subject by localisation**, or an **exempt export**? |
| 5 | **Canary Islands**, business | 0% | The islands are outside the EU VAT area, so IGIC territory rather than IVA. Read FAQ section 23 ("Desglose del registro de facturación de alta cuando se trata de entregas de bienes o prestaciones de servicios localizadas en Canarias") before this one. |
| 6 | **Canary Islands**, consumer | 21% | We currently charge Spanish VAT to a Canary consumer. Given row 5, this is the row most worth sanity-checking rather than just classifying. |

### Where to read the actual code values

The permitted values are in the AEAT's record design spreadsheet, "Diseños de registro de
facturación", linked from
`sede.agenciatributaria.gob.es/Sede/iva/sistemas-informaticos-facturacion-verifactu/informacion-tecnica/disenos-registro.html`.
**Read the codes from there rather than from memory or from a summary**, including this one: they
are a compliance surface and they are versioned.

Row 3 also raises a question the current system does not ask: **do we validate EU VAT numbers?**
Reverse charge normally requires a valid one. Today `customer_dni_nif_cif` is free text and defaults
to an empty string. See [[migration#Things that will surface]].

## Two defects to fix rather than port

**The null-country crash.** `getInvoicesIvaMethod(): int` returns
`$province['invoicesIvaMethod'] ?? $country['invoicesIvaMethod']`. Every country row currently holds
a value, so this is safe today, but a customer whose `country_id` does not resolve produces `null`
from an `int` return type, which is an uncaught `TypeError` **outside** the surrounding try block.
In Numbers v2 an unresolvable country is an **explicit refusal to issue**, not a crash and not a
default. See [[architecture#Rejecting is the safe default]].

**An inconsistency between the two getters.** IVA uses `??` (null-coalescing), IRPF uses `?:`
(falsy). So a province that explicitly sets IRPF method `0` would be **ignored** and fall back to
its country, while the same province setting IVA method `0` would be honoured. No province currently
sets IRPF to `0`, so this is latent rather than live. Resolve it deliberately in the port instead of
reproducing it: a province override is present or absent, and `0` is a value.

## Where this lives in Numbers v2

The caller passes **facts**, not conclusions: country, province, fiscal id, whether the customer is
a business, the customer's IRPF election. Numbers v2 decides the treatment and produces both the
percentages and the Verifactu codes. See [[api-contract#Tax determination lives here]].

Field names are English: `vat` and `withholding`. See [[naming]].

## Related

- [[verifactu]]
- [[api-contract]]
- [[migration]]
