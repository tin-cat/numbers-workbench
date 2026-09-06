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

## Mapping onto Verifactu

Here is the part that still needs the gestor, and it is now a small concrete table rather than an
open question.

The existing system produces **percentages**. Verifactu needs **classification codes**
(`CalificacionOperacion`, `OperacionExenta`, `ClaveRegimen`). A 0% line can be several fiscally
distinct things: not subject by localisation rules, exempt, reverse charge, or an export. **The
existing data does not distinguish them**, because it only ever stored the resulting percentage.

So the mapping has to be made once, per group:

| Case | Percentage today | Verifactu code |
|---|---|---|
| Spain, any customer | 21% | to confirm, expected "sujeta y no exenta" |
| Europe, consumer | 21% | to confirm |
| Europe, business | 0% | to confirm: reverse charge, or not subject by localisation? |
| Rest of world, any | 0% | to confirm: not subject, or export? |
| Canary Islands, business | 0% | to confirm, and see the note below |
| Canary Islands, consumer | 21% | to confirm |
| IRPF 15% | withholding | how it is expressed in the record |

**Six rows and one withholding question.** That is the whole of the largest remaining piece of work,
and it is a conversation with the gestor rather than a research project.

For the Canary Islands specifically, the AEAT developer FAQ has a dedicated section (23, "Desglose
del registro de facturación de alta cuando se trata de entregas de bienes o prestaciones de
servicios localizadas en Canarias") covering IGIC rather than IVA. Read it before that
conversation.

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
