# Naming

**English everywhere in code**: class names, properties, variables, database tables and columns, API
fields, configuration keys, enum cases, file names. Spanish is not inherited from the systems being
replaced. Decided 2026-09-06.

Documentation prose is different: it uses the regulatory term where that is the precise word, and
gives the English domain name alongside. You cannot discuss Verifactu without writing
`TipoFactura`.

## Three boundaries where Spanish is unavoidable

The rule is about **our** vocabulary. Three external contracts define their own, and those are not
ours to rename. This is exactly what the hexagonal boundary is for: **the adapter translates, the
domain stays English.**

| Boundary | Whose vocabulary | Where it is allowed |
|---|---|---|
| **The AEAT XML** | The regulation. `TipoFactura`, `TipoRectificativa`, `FacturasRectificadas`, `RegistroFacturacionAlta`, `RegistroFacturacionAnulacion`, `PrimerRegistro`, `CalificacionOperacion`, `ClaveRegimen`, `SistemaInformatico` | Only inside the AEAT adapter, in the serialisation layer. Nothing upstream of it uses these names. |
| **The gestor export** | Victor's contract. Sheets `Ingresos` and `Gastos`, headers `Núm factura`, `Fecha emisión`, `Concepto`, `Proveedor` | Only inside the export adapter, as literal output strings. See [[gestor-export]]. |
| **The old databases** | What we are migrating from | Only in the migration's read side, which is throwaway code. |

If a Spanish identifier appears anywhere else, it is a bug.

## The domain vocabulary

The mapping the AEAT adapter performs, in one place:

| Regulation | Our domain |
|---|---|
| factura | `Invoice` |
| factura rectificativa | `CorrectiveInvoice` |
| registro de facturación de alta | `BillingRecord` |
| registro de anulación | `AnnulmentRecord` |
| huella | `RecordHash` (property: `hash`) |
| encadenamiento | `chain`, `previousRecord` |
| serie | `InvoiceSeries` |
| número | `number` |
| fecha de expedición | `issuedAt` |
| base imponible | `taxableBase` |
| IVA | `vat` |
| IRPF | `withholding` |
| tipo de cambio | `exchangeRate` |
| obligado tributario | `Issuer` |
| destinatario | `Customer` |
| proveedor | `Provider` |

`TipoFactura` becomes a `CorrectionReason` enum with English cases, and the R codes live only in the
adapter:

| Code | Enum case |
|---|---|
| R1 | `OperationCancelledOrPriceAltered` |
| R2 | `CustomerInsolvency` |
| R3 | `BadDebt` |
| R4 | `Other` |
| R5 | `SimplifiedInvoice` |

`TipoRectificativa` becomes `CorrectionMethod`: `ByDifference` (`I`) and `BySubstitution` (`S`).

### The two that are not clean translations

**`vat` for IVA** is exact and unambiguous.

**`withholding` for IRPF** is the standard English accounting term for a retención, but IRPF is a
specific Spanish personal income tax and the mapping loses that specificity. This is the one place a
short comment on the property earns its place, naming the Spanish original so nobody wonders which
withholding is meant.

## Misspellings not to inherit

Both source databases carry typos in column names. The migration is the moment to drop them, and
they are worth listing so nobody carefully reproduces one:

| Source | Wrong | Ours |
|---|---|---|
| Litmind `invoices` | `anullation_invoice_id` | `correctedInvoiceId` |
| Litmind `invoices` | `date_emmited` | `issuedAt` |
| Litmind `invoices` | `is_anonimized`, `date_anonimized` | `anonymized`, `anonymizedAt` |
| Old Numbers | `finantialBalances` | `financial_balances` |
| Old Numbers `invoices` | `final` (meaning the total) | `total` |
| Old Numbers `invoices` | `base` | `taxableBase` |
| Old Numbers | `suppliers` | `providers` (owner, decision 25) |
| Litmind `invoices` | `customer_dni_nif_cif` | `customerTaxId` |

`dni`, `nif` and `cif` are three Spanish document types; `taxId` covers all of them and does not
break when a customer is foreign.

## Case conventions

Standard Symfony and Doctrine, so the tooling agrees with the code without configuration:

- `PascalCase` classes, `camelCase` properties and methods.
- `snake_case` database tables and columns, through Doctrine's default naming strategy.
- `SCREAMING_SNAKE_CASE` environment variables.

## Coding standard

**php-cs-fixer with the `@Symfony` ruleset, with one override: no spaces around string
concatenation dots** (owner, 2026-09-06). `"a".$b."c"`, never `"a" . $b . "c"`, matching the house
style across the owner's other projects.

`@Symfony` sets `concat_space` to `one`, so it has to be overridden explicitly:

```php
// .php-cs-fixer.dist.php
return (new PhpCsFixer\Config())
    ->setRules([
        '@Symfony' => true,
        'concat_space' => ['spacing' => 'none'],
    ])
    ->setFinder(
        PhpCsFixer\Finder::create()
            ->in(__DIR__.'/src')
            ->in(__DIR__.'/tests')
    );
```

Everything else stays as `@Symfony` defines it, including **four-space indentation**. That differs
from the tabs used in Litmind, and it is deliberate: only the concatenation rule was carried across,
so the rest of the tooling ecosystem works without configuration.

Run it in CI as a check, not only as a local convenience, or it stops being a standard.

## Related

- [[corrections]] for what the R codes mean
- [[gestor-export]] for the export's fixed Spanish output
- [[migration]] for where the old names are read
