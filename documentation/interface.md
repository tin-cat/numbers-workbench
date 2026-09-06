# The web interface

Decided 2026-09-06. Nothing implemented.

## The choice

**Server-rendered Twig, with Symfony UX (Turbo + Stimulus), Bootstrap 5, and the Tabler admin
theme.**

No SPA. No separate frontend build beyond Symfony's AssetMapper.

## Why not a React or Vue front end

The application is forms, tables and a small number of workflows. That is the exact shape
server-rendered HTML is best at, and the exact shape where an SPA costs the most for the least
return.

Specifically, an SPA would require **exposing the whole domain through a second API surface** just
so the interface can draw a table. Numbers v2 already has one API, and it is a deliberately narrow,
machine-facing, per-source-credentialed contract with the source applications
(see [[api-contract]]). Widening it, or building a parallel one with session auth, doubles the
surface that has to be secured and kept honest on a system whose entire job is being trustworthy.

Turbo gives near-SPA navigation (no full page reloads, preserved scroll) with none of that. Stimulus
covers the sprinkles: a confirmation modal, a dependent select, a live total on the manual invoice
form.

In hexagonal terms the interface is a driving adapter, and keeping it server-rendered keeps that
adapter thin.

## Why Bootstrap 5 and Tabler

The brief is "the usual stuff": left column with sections, top menu with global menus, tables, tabs,
buttons, form elements. That is a solved problem and not worth solving again.

- **Bootstrap 5** is the most widely supported CSS framework for exactly this vocabulary, and it
  needs no build step.
- **[Tabler](https://tabler.io)** is a mature, actively maintained, MIT-licensed admin theme built
  on Bootstrap 5. It ships the sidebar, the top navbar, cards, data tables, tabs, form layouts,
  badges and empty states already assembled and coherent, in light and dark.

**The alternative considered was Tailwind**, which is excellent but ships no components. An admin
panel built on it means either adding a component library on top or building the entire chrome by
hand. For a back office where the goal is to look like every other back office, that is effort spent
in the wrong place.

## Why not EasyAdmin

EasyAdmin would produce a working back office fastest, and for a pure CRUD admin it would be the
right answer.

This application is not pure CRUD. Issuing a manual invoice, annulling one, issuing a partial
rectificativa, retrying a failed AEAT submission and generating the gestor export are **workflows
with domain rules**, not entity forms. EasyAdmin is pleasant until the screens stop being entity
forms, and then it is an obstacle.

Rejected on fit, not on quality.

## Assets

**AssetMapper**, Symfony's built-in importmap-based pipeline. No Node, no Webpack, no build step in
the container image, which matters more than usual given the deployment story in
[[deployment]].

Charts are **ApexCharts**, which is what Tabler's own chart components are built around, so they
look native to the theme. It vendors through AssetMapper without a build step. See
[[dashboard#Charts]].

Caveat to know in advance: AssetMapper does not compile Sass. Tabler ships compiled CSS, so this is
fine as long as customisation stays at the CSS-variable level. If deep Bootstrap-source theming ever
becomes necessary, that is the point at which Webpack Encore comes back, and it is a decision worth
avoiding.

## Content Security Policy

Nonce-based, no inline scripts, with one deliberate exception: **Cloudflare Turnstile** loads a
script and an iframe from `challenges.cloudflare.com` and the policy has to allow both explicitly.
Write it down before the first deploy rather than debugging it afterwards. See
[[access-control#Turnstile]].

## Layout

The standard shape, because it is standard:

- **Home**: the dashboard, panels and charts, for anyone with `statistics.view`. See
  [[dashboard]].
- **Left column**: Invoices, Manual invoice, Expenses, Providers, Verifactu (submission queue),
  Export, Users, Audit. Each section hidden when the signed-in user lacks its permission, not shown
  and then refused.
- **Top bar**: current user, sign out, environment indicator, and a visible **AEAT submission
  backlog** count. That last one is not decoration: a growing backlog is the main operational
  signal this system produces, and it belongs where it cannot be missed. See
  [[architecture#Submission never blocks issuance]].
- **Content**: tables with server-side pagination, filtering and sorting.

## One thing to get right in the tables

**Server-side pagination, always.** There are already 58,807 invoices, and the client-side data
table pattern of shipping every row and filtering in the browser will appear to work in development
and fall over on real data. See [[source-data-findings#Volume]].

## Presenting retention state

Because invoices age through minimize, anonymize and purge, a list or detail view can encounter a
record whose fields are gone. The interface shows this as a state with its date, not as blanks or a
404: "details retained for tax purposes until 30/6/2031". See
[[data-retention#Read-path status codes]].

The same applies to the source applications, which is why that is an API concern too.

## Versions

Pin the current stable Symfony at scaffold time and check what that is rather than assuming. Symfony
majors land every two years in November, with the last minor of each cycle being the LTS, so the
choice is between the current major and the previous LTS. For a system with this lifespan, **the
LTS** is the better default.

## Related

- [[access-control]] for who can see which section
- [[gestor-export]] for the export screen's contract
- [[corrections]] for the annul and refund flows the interface has to express
