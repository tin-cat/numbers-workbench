# Glossary

The Spanish regulatory vocabulary these documents cannot avoid, in plain terms. Our code uses
English names for all of it, see [[naming]].

**SIF**, *Sistema Informático de Facturación*. Invoicing software. The regulation's term for any
system that issues invoices, and the thing it imposes duties on: chaining every record, putting a QR
on every invoice, identifying itself in what it produces. **Numbers v2 is a SIF.** Litmind stops
being one when it stops issuing.

**VERI\*FACTU**. The mode where a SIF sends every record to the AEAT as it is generated. The
alternative, "NO VERI\*FACTU", keeps the records locally instead and carries considerably heavier
obligations in exchange. We build VERI\*FACTU-only, deliberately. See
[[decision-log#29. Build VERI\*FACTU-only, and never add a non-submitting mode]].

**AEAT**, *Agencia Estatal de Administración Tributaria*. The Spanish tax authority.

**RF**, *registro de facturación*. A billing record. The structured record a SIF generates for each
invoice, chained to the previous one and sent to the AEAT. Not the same thing as the invoice: the
invoice is the document the customer gets, the RF is the record about it.

**RF de alta**. A record that adds an invoice. Ordinary invoices and rectificativas are both altas.

**RF de anulación**. A record that withdraws a previously sent record, for an invoice that should
never have existed because the operation never happened. Rare and narrow. See [[corrections]].

**Alta de subsanación**. A remedial record correcting internal fields of a previous record, ones
that never appear on the printed invoice. Also rare.

**Huella**. Literally "fingerprint": the hash over a record's contents, chained to the previous
record's. We call it `hash`.

**Encadenamiento**. The chaining itself.

**Obligado tributario**, or **OEF**, *obligado a expedir facturas*. The taxpayer who issues the
invoices. Here, NIF ES 39898734J.

**Factura rectificativa**. A corrective invoice. Not a credit note in the Anglo sense: it is a real
invoice, with its own number in its own series, that corrects another. We call it a
`CorrectiveInvoice`.

**Serie y número**. The invoice series and its number within that series. Together with the issuer
and the issue date, they identify a record uniquely to the AEAT.

**Fecha de expedición**. The issue date. Part of the record's identity.

**Devengo**. The moment a tax becomes chargeable. Usually when the service is provided, but an
advance payment brings it forward.

**Base imponible**. The taxable base, the amount before tax. We call it `taxableBase`.

**Cuota repercutida**. The tax charged on that base. Literally "the quota passed on".

**IVA**, *Impuesto sobre el Valor Añadido*. VAT.

**IGIC**, *Impuesto General Indirecto Canario*. The Canary Islands' equivalent of VAT. The islands
sit outside the EU VAT area, which is why they get their own treatment. See [[tax-determination]].

**IRPF**, *Impuesto sobre la Renta de las Personas Físicas*. Spanish personal income tax. On an
invoice it appears as a **retención**, an amount the customer withholds and pays to the tax office
on the issuer's behalf. We call it `withholding`. **It is not part of the billing record.** See
[[tax-determination#IRPF is not in the record at all]].

**ROF**, *Reglamento de Obligaciones de Facturación* (RD 1619/2012). The invoicing rules. Older than
Verifactu and unchanged by it: Verifactu governs the software, the ROF governs the invoice.

**RRSIF**, *Reglamento de requisitos de los sistemas informáticos de facturación* (RD 1007/2023).
The regulation that created all of this.

**OM HAC/1177/2024**. The ministerial order developing the RRSIF into technical detail: field
formats, code lists, the hash algorithm.

**LIVA**. Ley 37/1992, the VAT law. Article 80 is the one that matters here: it lists the grounds on
which a taxable base may be reduced, which is what the R1 to R5 codes point at. See
[[corrections]].

**Declaración responsable**. The statement that a piece of invoicing software complies. Normally the
vendor's duty; for software written by the taxpayer for their own use, it is the taxpayer's.

**Gestor**. Not a regulatory term. The accountant.
