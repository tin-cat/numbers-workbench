# Money

How amounts are represented, stored and moved. Decided 2026-09-06.

## Never a float, never a double, anywhere

The existing systems get this wrong in both directions: old Numbers stores money as `float` (single
precision, about seven significant digits), Litmind as `double`. Neither can represent a decimal
amount exactly, and both accumulate error under arithmetic.

Numbers v2 stores money as an **integer of minor units plus an explicit scale**, wrapped in value
objects, and never converts to a floating point type at any point in its life. That includes the
API, see [[#Money on the wire]].

## The value objects

**`Currency`.** An ISO 4217 code and its natural scale (EUR 2, JPY 0). A small closed set,
immutable, comparable by identity.

**`Amount`.** A scaled integer with no currency: an integer of minor units plus a scale. This is the
decimal type, used for amounts, percentages and rates.

**`Money`.** An `Amount` and a `Currency` together. Every monetary field on an invoice is one of
these.

Rules the objects enforce:

- **Immutable.** Every operation returns a new instance.
- **Mixing currencies throws.** Adding EUR to USD is a programming error, not a conversion.
- **Addition and subtraction of the same currency are exact**, with no rounding, ever.
- **Multiplication by a rate does not round implicitly.** Applying a VAT percentage produces a
  higher-scale result, and turning it back into a payable amount requires an explicit, named
  rounding call with a stated mode. There is no operator that quietly loses precision.

## Scale

Currency scale is not enough on its own. Litmind's existing data carries amounts like `3.949`, and
`invoices::create()` rounds to **four** decimal places throughout. Line-level amounts genuinely need
more precision than the currency's two.

So scale is stored explicitly alongside the integer rather than derived from the currency:

| Field | Type | Example |
|---|---|---|
| `amount_minor` | `BIGINT` | `39490` |
| `scale` | `SMALLINT` | `4` |
| `currency` | `CHAR(3)` | `EUR` |

Line items carry the higher scale. Invoice totals and tax figures are rounded to the currency's
natural scale, once, explicitly, at the point where the invoice is composed.

## Currencies

**EUR and USD** for now. Both are already live: of Litmind's 58,807 invoices, 58,649 are EUR and
**111 are USD**, issued continuously between 2019-05-06 and 2025-12-29. USD is not hypothetical and
cannot be treated as a future feature.

`Currency` is a closed set, so adding a third later is a data change rather than a design change.

### The tax figures must also exist in euros

An invoice may be denominated in any currency, but the VAT amount has to be expressed in euros.
**CONFIRM** the exact wording with the gestor, but plan for it: a USD invoice carries its amounts in
USD *and* its tax figures in EUR.

So the record holds both, plus the rate that connects them.

### The exchange rate is part of the fiscal record

The rate used is **captured at issuance and stored permanently on the invoice**, together with its
date and its source. It is never recomputed.

This is not tidiness. The huella covers the amounts, and a PDF rendered in 2030 from a rate fetched
at render time would not match the record submitted in 2026. A re-derived rate is a broken record.

Store: the rate as an exact decimal at high scale, the rate date, and the source that published it.

**CONFIRM** which rate applies: the usual answer is the reference rate published by the Banco de
España or the ECB on the fecha de devengo, but the gestor should say.

### Rate availability must never block issuance, and never be guessed

Fetching a rate from an external service at issuance time would put an outside dependency in the
middle of the one operation that must not fail unpredictably.

So: rates are **cached locally and refreshed on a schedule**, and issuance reads only from the local
store. If no rate is available for the date required, Numbers v2 **refuses to issue** rather than
using yesterday's, or the nearest, or a default. Consistent with
[[architecture#Rejecting is the safe default]]: an invoice with a guessed rate is wrong forever.

A missing rate should raise an alert, not a fallback.

### Order of operations

1. Compute the invoice in **its own currency**, with its own rounding, exactly as a EUR invoice is
   computed.
2. Convert the **tax figures** to EUR.
3. Round the converted figures once, at that point.

Never convert the inputs and then compute, and never convert an already-rounded total by a second
route: the two paths disagree by cents and both would be defensible, which is how a discrepancy
becomes unresolvable.

### `ExchangeRate` is a value object too

From currency, to currency, an exact decimal rate at high scale, a date, and a source.

`Money::convertTo(Currency, ExchangeRate)` is explicit and takes the rate as an argument. It never
fetches one. A conversion with no rate passed is a compile-time impossibility, not a runtime
lookup.

### Corrections carry the original rate

A rectificativa or an annulment of a foreign-currency invoice uses **the rate stored on the invoice
it corrects**, not today's rate. Otherwise the correction does not net against the original and the
books never balance. See [[corrections]].

## Rounding

Rounding happens in exactly two places, and nowhere else:

1. When a tax figure is computed from a base and a rate.
2. When an invoice total is composed.

Both use one stated mode (half up is the Spanish convention for VAT; **CONFIRM** with the gestor)
and both are explicit calls, never a side effect of storage or display.

## The invariant that catches everything

A `TaxBreakdown` value object (English field names throughout, see [[naming]]) that guarantees, in
its constructor, that

```
taxableBase + vat - withholding == total
```

exactly, as integers, and refuses to exist otherwise. Rounding errors are invisible until they break
this, so make it impossible to construct a breakdown that does.

This is also the reconciliation check for the migration, applied per invoice.

## Money on the wire

The API must **not carry money as a JSON number.** Most JSON parsers turn a bare number into a
double, which silently reintroduces exactly the problem the storage model exists to avoid.

Money crosses the API as either a decimal **string** (`"3.9490"`) or an object
(`{"minor": 39490, "scale": 4, "currency": "EUR"}`). Pick one and hold it. Same rule for anything
persisted as JSON.

## Migration conversion

Converting old Numbers' `float` and Litmind's `double` into integer minor units is a place to be
careful. A stored float of `3.949` may actually be `3.94899999...`, and a direct cast to integer
truncates it to `3948`.

**Convert through the decimal string representation** (`printf("%.4f")` or the database's own
`CAST(... AS DECIMAL(18,4))`), never by multiplying the float and casting.

Then reconcile: for every invoice, the `TaxBreakdown` invariant must hold, and the summed totals per
source per fiscal year must match the source system exactly. See
[[migration#Reconciliation is the deliverable]] and
[[source-data-findings#Amount precision]].

## Related

- [[architecture]]
- [[api-contract]]
- [[migration]]
