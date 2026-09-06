# The gestor export

An Excel workbook of the complete accounts, in a format Victor integrates into his own systems.

**The format is an external contract and does not change.** It is not ours to improve. Any change to
the layout, the sheet names, the column order, the headers or the cell types can break his
integration silently.

## What the current export produces

From `app/export.php` in old Numbers, using PhpSpreadsheet. This is the specification to reproduce.

**Two sheets, named `Ingresos` and `Gastos`, in that order.**

`Ingresos`, columns A to K:

| Col | Header | Content | Type |
|---|---|---|---|
| A | Núm factura | invoice code | string (explicit) |
| B | Cliente | client name | string (explicit) |
| C | CIF/NIF | client fiscal id | string (explicit) |
| D | Fecha emisión | issue date | date |
| E | Subtotal | base | money, currency-formatted |
| F | IVA | | money |
| G | IRPF | | money |
| H | Total | | money |
| I | Concepto | the line item descriptions | string (explicit) |
| J | % IVA | **formula** `=F{row}/E{row}` | percentage formula |
| K | % IRPF | **formula** `=G{row}/E{row}` | percentage formula |

`Gastos`, columns A to I:

| Col | Header | Content |
|---|---|---|
| A | Fecha | |
| B | Código factura | the provider's invoice code |
| C | Proveedor | provider name |
| D | CIF | provider fiscal id |
| E | Concepto | description |
| F | Base | deductible portion only |
| G | IVA | deductible portion only |
| H | Total | deductible portion only |
| I | % IVA | **formula** `=G{row}/F{row}` |

Details that are part of the contract and easy to lose in a rewrite:

- **Header styling**: bold white text on a black fill, across `A1:K1` and `A1:I1`.
- **Column widths** are set explicitly per column. Reproduce them.
- **Rows are ordered by date ascending**, and **a blank row is inserted between months**. Victor may
  well read the file by eye as well as by machine.
- **The percentage columns are live formulas, not values.** They must stay formulas.
- **Text columns are written with an explicit string type**, so a fiscal id like `39898734J` or a
  code like `21001` is never coerced into a number.
- **Expense amounts are the deductible portion**, through `getDeductibleAmount()`, not the gross.

## Implementation

The Spanish sheet names and headers above are **literal output strings inside the export adapter**,
never identifiers. The code that builds the report model is English like everything else. See
[[naming#Three boundaries where Spanish is unavoidable]].

An outbound port in the hexagonal sense: the domain produces a report model for a period, an adapter
renders Victor's exact workbook. If the format ever does change, that is a new adapter, kept
alongside the old one, not an edit to it.

**Pin it with a golden-file test.** Keep a known-good reference workbook in the repository as a
fixture (old Numbers already ships one at `app/export/contab-loren.xlsx`) and assert the generated
file matches it cell for cell, including types, formulas and the blank-row rhythm. This is the only
mechanism that will keep "the format does not change" true in three years, when nobody remembers why
column J is a formula.

## Delivery

Two ways out, both wanted:

- **Download**, always available. Email delivery fails quietly more often than a download does, so
  this is the fallback that must never not work.
- **Send to the gestor by email**, on an explicit button.

Constraints on the email path, because it sends client names and fiscal ids to an external address:

- The recipient is **configuration**, not typed at click time.
- Every send is **audit logged**: who, when, which period, what file hash.
- The period is in the filename and the subject, so a resend is never ambiguous.
- It is an explicit action with a confirmation, not a side effect of anything else.

## Two settled questions

### Currency

`cellMoney()` writes the raw amount and changes only the cell's **number format**, to a euro or a
dollar symbol depending on the invoice currency. A USD invoice's figures therefore land in the same
Subtotal, IVA and Total columns as the EUR ones, so summing a column adds dollars to euros.

There are 111 USD invoices totalling 1,853.20, so the exposure is small, and Victor may well handle
them by hand. But Spanish books are kept in euros, and Numbers v2 will hold the EUR equivalent of
every foreign-currency invoice anyway (see [[money#The tax figures must also exist in euros]]), so
it *could* export converted figures, or both.

**Decided (owner, 2026-09-06): keep it exactly as it is.** This is a known issue and it is handled
outside the export. Numbers v2 reproduces the current behaviour, raw amount plus a currency number
format, and does not convert.

Worth a comment in the exporter saying so, since it looks like a bug to anyone reading it cold and
the next person will otherwise "fix" it.

### Period

The current export is **all time**, every invoice and expense ever, with no period filter.

**Decided (owner, 2026-09-06): no period filter.** Reproduce the current behaviour. Revisit only if
Victor asks.

## Related

- [[money]] for why amounts must not pass through a float on the way into a cell
- [[architecture]]
