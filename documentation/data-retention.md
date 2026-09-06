# Data retention

Who owns the clock on invoice data, and what crosses the boundary.

## The principle

**Numbers v2 owns retention for invoices, entirely.** Source applications do not anonymize, minimize
or purge invoice data, and cannot. They send signals; Numbers v2 decides and acts.

Two reasons, one technical and one legal.

**Technical.** The huella covers the record's contents. Anonymizing a chained record destroys the
ability to prove it matches its own hash. Retention has to live where the chain lives.
See [[verifactu#Why retention had to move to the SIF]].

**Legal.** The retention clock is a property of the invoice, not of the customer's account. Right
now in Litmind a user pressing "delete my account" reaches a code path that touches fiscal records,
which is the wrong shape: Article 17(3)(b) GDPR exempts processing necessary for compliance with a
legal obligation, and invoices are the textbook case. The correct answer to an erasure request is
not "erased" but "retained under legal obligation until date X, then erased". Numbers v2 holding the
clock is what makes that answer expressible.

## The policy already exists and moves across intact

This is an asset, not a rewrite. Litmind already separates
`anonimizeAllDataExceptInvoicingRelated()` from `anonimizeInvoicingRelatedData()`, and runs the
second only at `COMPLETELY_ANONIMIZED`, whose clock is the end of the fifth fiscal exercise, on a
June 30th, because that is when the IRPF closes. That is already a fiscal clock rather than a
product clock, and it is already the correct instinct.

What Litmind loses is `anonimizeInvoicingRelatedData()` and the invoice half of
`MigrateUserToCompletelyAnonimizedDataStateService`. The rest of the `UserDataState` machinery stays
exactly as it is, because it is about content and profiles, which Numbers v2 has no business
knowing about.

## Moving the clock resolves an open question

From `app/config/user-data-state.config.inc.php` in Litmind:

> Attention: This might also apply to invoices from non-cancelled users, we might delete them after
> this same period even if the user hasn't been cancelled. This should be checked with the DPD.

That question exists because Litmind's clock is anchored on the **user**:
`USER_DATA_STATE_COMPLETE_ANONIMIZATION_ON_EFFECTIVE_END_OF_NTH_YEARLY_FISCAL_EXERCISE` counts five
exercises from the cancellation date. So an invoice from 2019 belonging to a still-active member
never ages out, which is almost certainly not what the policy intends.

In Numbers v2 the natural anchor is the **invoice's own exercise of issuance**, because that is the
only thing the fiscal clock actually cares about. The question stops being open, and the answer
stops depending on an account state that has nothing to do with it.

## Four operations, four different clocks

Keep these separate in the model and in the API. Collapsing any two of them causes trouble later.

| Operation | What it does | Floor |
|---|---|---|
| **Minimize** | Drops what the fiscal record does not need: phone, email, possibly the payment processor ids. Keeps name, NIF, address, amounts, dates, description, huella. | Earliest clock. Can run while the record is still fully live. |
| **Anonymize** | Strips the customer identity. | Cannot run while any retention period is open: an invoice that does not identify its customer is not a valid invoice. This is the fifth-exercise clock. |
| **Purge** | Removes the detail. | **The chain link cannot go.** Position, huella and the link to the previous record must survive, or every later record becomes unverifiable. "Purge" means purging the detail and keeping the skeleton, never deleting a row. |
| **Cancel** | Not a retention operation at all. Annulling an invoice is a *registro de anulación*, a fiscal act that creates a new record in the chain. | Keep it out of this family entirely. |

The last row matters more than it looks. If one API verb ever means both "annul this invoice" and
"we have finished retaining it", that ambiguity will outlive the project.

## Computing the clock

Numbers v2 takes the **maximum** of the applicable periods (tax prescription, the commercial books
requirement under the Código de Comercio, and the Verifactu conservation duty), computed from the
close of the exercise rather than from the invoice date. Sources never see the arithmetic.

The periods themselves should be put in front of the gestor once. The architecture does not change
either way; only a number in the config does.

## The signals that flow inward

Smaller than expected, because Litmind's own configuration settles it. The invoice clock anchors on
the **cancellation date** and nothing else. The infringing level, which drives the 180 day / 2 year
/ 5 year fork, only moves the *partial* anonymization, and invoicing data is touched at complete
anonymization. So Numbers v2 never needs to know the infringing level exists.

That leaves two messages:

- **customer cancelled on date X** (and uncancelled, since a cancelled user can come back)
- **erasure requested on date X**

Both are **recorded, not obeyed on arrival**. Numbers v2 applies the fiscal clock and acts at the
earliest lawful moment.

`Users\Events\MigratedToUserDataState` is already published on every transition in Litmind, so the
carrier exists. It needs a subscriber that forwards the cancellation transitions.

## Read-path status codes

Because sources hold no copy, a source asking for an invoice must be told what state it is in. The
response carries a status (live, minimized, anonymized, purged) and the relevant date, so the
account page can say honestly: "invoice WEB/24/01234, 12.10 EUR, details retained for tax purposes
until 30/6/2031" rather than either lying or returning a 404.

## Related

- [[architecture]]
- [[verifactu]]
- [[api-contract]]
