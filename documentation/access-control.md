# Users, authentication and permissions

Decided 2026-09-06. Nothing implemented.

## Two front doors, two firewalls

Numbers v2 is reached two ways, and they must not share an authentication path:

| | Who | How |
|---|---|---|
| **The web front** | People | Login and password, plus a second factor |
| **The source API** | Litmind and the other applications | Per-source credentials, scoped to that source's own invoices and series. See [[api-contract#Security]] |

Separate Symfony firewalls, separate credential stores, separate audit trails. A machine credential
must never be usable to log into the web front, and a person's session must never authenticate an
API call.

## Password storage: Argon2id, not PBKDF2

PBKDF2 is the answer when FIPS-140 compliance is required. It is not the best available answer
otherwise, because it is cheap to accelerate on GPUs and ASICs.

**Use Argon2id**, the winner of the Password Hashing Competition and OWASP's first recommendation.
It is memory-hard, so specialised hardware buys an attacker far less.

- PHP supports it natively: `password_hash($p, PASSWORD_ARGON2ID)`, given a build with argon2
  support. Symfony's `auto` hasher selects it when available.
- Parameters as a starting point, from the OWASP Password Storage guidance: **m = 19456 KiB (19
  MiB), t = 2, p = 1**. **CONFIRM** against current OWASP guidance when the code is written; these
  numbers move.
- Configure `migrate_from` and rehash on successful login, so raising the parameters later upgrades
  every account as people sign in rather than needing a reset.

If a future host build lacks argon2, the fallback order is **scrypt, then bcrypt**, never plain
PBKDF2 unless something external demands it. Symfony's `auto` hasher handles this, but the intent
should be explicit in the configuration rather than left to a default.

## The rest of the login path

A system holding the fiscal records of several businesses deserves more than a password:

- **SMS second factor, required for every user.** See [[#Two-factor authentication]].
- **Cloudflare Turnstile on every form**, login included. See [[#Turnstile]].
- **Login throttling**, using Symfony's built-in `login_throttling`, per account and per IP.
- **Session cookies** `secure`, `httponly`, `samesite=lax`, with session regeneration on login.
- **CSRF tokens on every form.** Symfony's form component does this by default; the thing to avoid
  is turning it off for convenience on an internal tool. Litmind's admin has no CSRF tokens and that
  is a known open item there. Do not repeat it here.
- **A password policy that follows NIST rather than folklore**: a long minimum, no composition
  rules, no forced rotation, and a check against known-breached passwords. Symfony's
  `NotCompromisedPassword` constraint does the last one via a k-anonymity call to Have I Been
  Pwned, which is an outbound request at password-set time only. Acceptable, but a deliberate
  choice: note it or disable it, do not let it be a surprise.
- **Nonce-based CSP** with no inline scripts.
- **Audit log** of logins, failed logins, permission changes and every issuance. See
  [[#What gets audited]].

There is no self-service password reset to begin with. With a handful of users, an admin-initiated
reset that emails a short-lived signed link is simpler and has less attack surface than a public
"forgot password" form. Revisit if the user count ever justifies it.

## Two-factor authentication

**SMS codes through Twilio Verify, required for every user without exception.**

Reuse the arrangement Litmind already runs (`doc/twilio-sms.md` there): one Twilio account, and
**one Verify service per product**, because the service's friendly name is what appears in the
message body. Numbers v2 needs its own, so the SMS reads as Numbers rather than as Litmind.

Verify rather than plain SMS sending is the right call for the same reason it was there: codes are
generated, sent and checked from Twilio's compliance-managed sender pool, so no sender number has to
be bought and no 10DLC or toll-free registration is needed.

Operational facts inherited from that setup, worth knowing before designing the screens: **codes
live ten minutes and die after five wrong attempts**, both fixed by Twilio. In the development
environment every verification goes to one fixed phone regardless of destination.

Phone numbers are set **at enrolment, through the console command or the user screen**. They are
deliberately **not written into this repository**: a 2FA phone is an authentication factor, and an
authentication factor in git history is permanent. Litmind already carries that lesson in the form
of R2 keys that are still in its history and still need rotating.

### The caveat, stated once

SMS is the weakest of the common second factors. SIM swap and SS7 interception are real, and NIST
deprecated SMS as an out-of-band authenticator for exactly that reason. For a system holding the
fiscal records of several businesses, it is worth knowing that this is the soft spot.

It is also the pragmatic choice here: the Twilio integration exists, it works, and a second factor
that is actually used beats a better one that is not. Building it as asked.

**Recommendation, not a blocker:** allow TOTP as an *alternative* second factor for anyone who wants
it, keeping SMS as the default and the fallback. Twilio Verify supports TOTP too, so this is a
configuration of the same service rather than a second integration.

### Recovery, which is not optional here

There is one admin. If the phone is lost, stolen, or Twilio is unreachable, **nobody can log into
the system that holds the fiscal records**. That is a genuine single point of failure and it needs
two answers:

- **Recovery codes**, generated at enrolment, displayed once, stored hashed like passwords. Single
  use, with a count shown in the interface so a dwindling supply is visible before it runs out.
- **A break-glass console command** on the server (`app:user:reset-2fa`) that clears a user's second
  factor so they can re-enrol. Shell access is already the most privileged thing anyone has, so this
  adds no new attack surface, and it is the path that works when Twilio itself is down. Every use is
  audit logged.

## Turnstile

**Cloudflare Turnstile on every form, including login.**

Two things to settle in advance, because they are the ones that cause trouble:

**Content Security Policy.** Turnstile loads a script and an iframe from
`challenges.cloudflare.com`, so the CSP has to allow it explicitly. This is compatible with the
nonce-based, no-inline-script policy, but it has to be written down or the first deploy fails
confusingly.

**What happens when Turnstile itself is unavailable.** The verification is a server-to-server call
to Cloudflare, and that call can time out. Failing closed everywhere means a Cloudflare incident
stops all work in the back office; failing open everywhere makes the control decorative.

Recommended split, **to confirm**:

| Form | On verification failure |
|---|---|
| Login, and anything reachable without a session | **Fail closed.** Retrying is cheap, and this is where the control actually earns its place, against credential stuffing. |
| Forms inside the authenticated area | **Fail open, log it, alert.** The real controls there are the session and the CSRF token; Turnstile is defence in depth, and it should not be able to halt the accounts. |

Whichever way this goes, it must be a configured, documented behaviour rather than whatever the HTTP
client happens to do on a timeout.

## The permission model

Permissions are **named capabilities held by a user**. There is no role hierarchy, because with a
handful of users a hierarchy is ceremony that hides who can do what.

`admin` is special: it implies every other permission. It is not stored as the full set, it
short-circuits in the voter, so a permission added in 2028 is automatically held by admins without a
data migration.

**loren@tin.cat holds `admin`.**

### The permissions

| Permission | Grants |
|---|---|
| `admin` | Everything, including every permission added later |
| `users` | Add and edit users, set their permissions |
| `statistics.view` | The dashboard and its charts. See [[dashboard]] |
| `invoices.view` | See invoices, their PDFs and their submission state |
| `invoices.issue` | Create manual invoices |
| `invoices.correct` | Annul an invoice, issue a rectificativa |
| `expenses.view` | See expenses |
| `expenses.add` | Add expenses |
| `providers.manage` | Add and edit providers |
| `export` | Generate the gestor export and send it |
| `verifactu.operate` | See the AEAT submission queue, retry failed submissions |
| `audit.view` | Read the audit log |

Two notes. **Providers, not suppliers**, everywhere: interface, domain, database and API. Old
Numbers calls the table `suppliers`, and the migration renames it rather than letting the two terms
drift (owner, 2026-09-06). And `statistics.view` gates the dashboard, so a user without it lands on
their first permitted section instead. See [[dashboard#Who sees it]].

### What no permission grants

Worth stating explicitly, because the instinct to add an override is strong and the whole design
depends on resisting it:

- **No permission deletes an invoice.** There is no delete, for anyone. See
  [[decision-log#10. No delete verb, ever]].
- **No permission edits a chained record.** Corrections are new records. See [[corrections]].
- **No permission changes a series number** or forces a chain past a refusal.
- **No permission suppresses an AEAT submission.**

The permission system controls **access**, never the immutability guarantees. An "admin can force
it" escape hatch would quietly become the thing that breaks the chain at three in the morning.

### Editing expenses

**Decided (owner, 2026-09-06):** `expenses.add` also covers **editing and deleting entries the user
created themselves**. Amending someone else's requires `admin`.

That keeps the two permissions as specified without leaving a mistyped amount uncorrectable, and it
gives the ownership check somewhere to live: an expense records who entered it, and that field is
what the voter reads.

This is unrelated to invoice immutability. An expense is a bookkeeping entry about someone else's
invoice, not a fiscal document we issued, so it is editable in a way an invoice never is.

### Two invariants on the `users` permission

- **A user cannot grant a permission they do not themselves hold.** Without this, `users` is
  silently equivalent to `admin`, since anyone holding it could simply grant themselves admin. With
  it, `users` is still a powerful permission, but a bounded one.
- **The last admin cannot be removed or demoted.** Enforced at the domain level, not in the form, so
  it holds however the change arrives.

Even with the first guard, treat `users` as near-admin when deciding who gets it.

## Bootstrap

The first admin is created by a **console command** (`app:user:create`), interactively, at
deployment. Never a seeded password in a migration, never a default credential, never a fixture that
could reach production.

## What gets audited

An append-only log, readable with `audit.view` and never editable through the application:

- Logins, failed logins, second-factor failures, recovery-code use, and every break-glass 2FA reset.
- Every user creation and every permission change: who, whom, what changed, when.
- Every invoice issued, annulled or rectified, with the acting user or the calling source.
- Every gestor export generated or sent, with the period and the recipient. See
  [[gestor-export#Delivery]].
- Every use of the signing certificate.

## Related

- [[interface]] for how this is presented
- [[api-contract#Security]] for the machine-facing side
- [[deployment#Secrets]]
