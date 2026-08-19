---
name: own-your-funnel
description: >
  Use when the user wants to set up analytics, track conversions, improve
  my funnel, asks why are users dropping off, wants an analytics report,
  or runs /report or /ask. Infers the conversion path from the repo,
  confirms a small funnel.yaml, adds only the missing events through the
  existing analytics wrapper, then answers questions and writes reports
  from the app database. Do not use to add a new analytics vendor or SDK.
---

# own-your-funnel

An agent skill — markdown the coding agent loads, not a product. No new
SDK, no hosted service, nothing leaves the machine.

The user is a developer. They are not an analyst. **Never ask them an
open-ended analytics question** (what their funnel is, which events
matter, what a healthy drop-off looks like). Infer it from the repo,
propose a filled-in mapping, and ask them to confirm or correct it in
developer terms.

Terms, once:

- **Funnel** — the short sequence of steps from landing on the site to
  paying. At most 8 stages.
- **Event** — a named record that something happened (`page_view`,
  `begin_checkout`, `purchase`).
- **Wrapper** — the one module/hook that already sends events to GA /
  the pixel / etc. New events go through it, nowhere else.
- **Drop-off** — people who did step N but not step N+1.

Load `references/doctrine.md` before judging sources or stating numbers.
Load `references/operate.md` only in `ask` or `report`.

## Router

Setup runs **once**. After that, `ask` and `report` forever. Do not
re-run setup unless `funnel.yaml` is missing or the human asks to redo
it.

| Situation | Mode | Also load |
|---|---|---|
| No `funnel.yaml` (or human says "set up analytics") | **setup** — this file | `references/doctrine.md` (fingerprints) |
| `funnel.yaml` exists + a question or `/ask` | **ask** | `references/operate.md`, `references/doctrine.md` |
| `funnel.yaml` exists + periodic readout or `/report` | **report** | `references/operate.md`, `references/doctrine.md` |

**Done when:** `funnel.yaml` exists, the human has confirmed the money
mapping and the stages, and `ANALYTICS.md` points at it.

---

## Setup

Ten steps, in order. Each has a completion criterion. Do not skip ahead
to instrumentation.

### 0. Check for prior work

Before discovering anything, look for an existing `ANALYTICS.md` (root,
backend, or named in `CLAUDE.md` / `AGENTS.md`).

If one exists it usually already contains the vendor list, the wrapper
path, and the event catalogue — the output of steps 3, 4, 5 and 10.
**Switch to verify-and-diff:** read it, then confirm each claim against
the code and report only what has drifted (an event that no longer
fires, a component calling `gtag`/`fbq` directly instead of the
wrapper, a vendor added or removed). Rediscovering from scratch and
overwriting a hand-written file that is better than your template is a
regression, not setup.

Steps 1–7 below then become verification passes rather than discovery.

**Done when:** you have either found no prior doc, or listed the
specific drifts between the existing doc and the code.

### 1. Identify the tree

Decide: frontend, backend, monorepo, or unknown.

Look at the repo root: `package.json` / `app/` / `pages/` / `src/`
(frontend), `prisma/` / `supabase/` / `app/api` / webhook handlers
(backend-ish), `apps/` / `packages/` (monorepo). If it is unclear,
say so and keep looking in step 2 rather than guessing a stack name.

**Done when:** you have stated the tree type in one sentence.

### 2. Find the sibling repo

Client-side events never contain purchase truth. The payment webhook
lives on the backend.

Do not look for two fixed paths. Look for the **markers**, at any
depth inside this repo and one level up:

- a payment webhook handler (`*webhook*`, `razorpay`, `stripe`)
- database migrations (`migrations/`, `supabase/`, `prisma/`)
- serverless/edge function directories (`functions/`, `api/`)

Then check, in this order: nested directories at any depth (a backend
named `<product>-backend/backend/` is common and matches no fixed
pattern), `../<name>-backend` next to this repo, nested repos named in
`.gitignore` that exist on disk, and paths already named in
`ANALYTICS.md` / `CLAUDE.md` / `AGENTS.md`.

If the other half is missing, **state explicitly what cannot be
instrumented**. That is almost always `purchase`, which belongs on the
payment webhook — not on a thank-you page.

**Done when:** frontend and backend paths are known, or the missing
half and the events it would have owned are named.

### 3. Detect existing analytics vendors

Grep using the fingerprint table in `references/doctrine.md`. List
every vendor that is actually in the repo.

**Never add a new vendor or SDK.** If they have GA4 and a Meta Pixel,
you work with those. Detecting Sentry / Vercel Analytics / Hotjar /
Clarity is so you do not add a redundant tracker — you will not report
funnel numbers from them.

**Done when:** `vendors_detected` is a concrete list (empty if none).

### 4. Find the existing analytics wrapper

Find the one hook/module that already fans out to the vendors. Typical
names: `useAnalytics`, `analytics.ts`, `track.ts`, `lib/analytics`.

All new events go through it. If there is genuinely no wrapper, propose
creating **one thin one** (a single `track(event, props)` that calls
the existing `gtag` / `fbq` / etc.). Do not sprinkle `gtag()` calls
across components.

**Done when:** you have a file path for the wrapper, or a one-file
proposal to create it.

### 5. Infer the product funnel

Read routes, the checkout flow, payment webhooks, and event names
already firing. Propose **at most 8 stages** from landing to paid.

Map each stage to an event they already fire where possible.

**One funnel, one population.** If the product has two entry paths (a
main flow and a recovery/win-back flow) or two audiences on the same
code, do not blend them into eight stages — doctrine forbids mixing
populations. Map the primary path, name the excluded one explicitly in
the confirmation, and record the filter that separates them as the
stage's `where`.

Identify only **1–3 real gaps** — things that happen in the UI/backend with no
event. Do not invent a twelve-step marketing funnel. Do not ask the
human to name the stages.

A vendor's automatic event (GA4's `page_view`, the Pixel's
`PageView`) counts as "already fires" even though it appears in no
catalogue and no wrapper call — note where it comes from so the human
is not confused when they cannot grep it.

**Done when:** you can list the proposed stages, the existing event
each maps to (or `source: money` for paid), and the 1–3 gaps.

### 6. Find the money

Ground truth for revenue is the app database, not the ad pixel.

Find one of: the payments/orders table, a `paid` boolean, or the
payment webhook handler (Stripe / Razorpay / Lemon Squeezy / Polar).
Note the table, the predicate that means "this row is paid", the
amount column (or a unit price if there is no amount column), whether
that amount is in cents, the currency, and the timestamp column.

Two traps:

- **Subscriptions.** If this app has recurring billing, a renewal
  creates another paid row. Counting those as funnel conversions
  inflates the rate every month. Find the predicate that means *first*
  payment (`billing_reason = 'subscription_create'`, `is_first = true`,
  or a distinct `user_id` on their earliest row) and record it as
  `new_conversion_predicate`.
- **The amount is not on the row.** Common: price lives in a config
  table or a JSON blob, and the row only carries a plan tier
  (`plan = 'pro'`). Record `amount_source: derived` and write down the
  join or lookup in plain words, plus the tier→price map. Confirm the
  prices with the human in step 8 — they are the one person who knows
  what they actually charge.
- **No timestamp.** A bare `users.is_paid` boolean has no time on it,
  so no windowed report is possible. Look for `paid_at` / `updated_at`
  / a payments row. If there is genuinely no timestamp, say plainly
  that revenue can only be reported as a running total, never
  "last 7 days."

If you cannot find money (no backend, no payments table, no webhook),
leave the `sources.money` keys empty, say that revenue cannot be
reported, and continue. Do not ask the human where revenue lives in
analyst terms — ask in schema terms if you have a candidate
("is `orders.status = 'paid'` the paid row?"). If you have no
candidate, say so.

**Done when:** you can say, in one sentence, which table and which
condition means someone paid — or you have stated that money is
unmapped and the ceiling that creates.

### 7. Discover how to query

Find the read-only access path that **already exists** in this
environment. Do not install anything.

Check for: `psql $DATABASE_URL`, the `supabase` CLI, PostgREST with a
project URL + key, `npx prisma db execute`, `sqlite3 <file>`, a
`DATABASE_URL` in `.env`, analytics-vendor credentials, an MCP server
for the database or for traffic (GA4 etc.).

There may be no `DATABASE_URL` and no `psql` at all — a hosted-Postgres
project key over REST is a perfectly good read path. Record whatever
actually works.

**Test it before you record it.** Run `SELECT 1` (or the vendor's
cheapest call). A command that exists but cannot connect — firewall,
missing role, expired key — leaves `/ask` dead on arrival, and you
will not find out until the user asks their first question.

Also settle **which database this is**. Localhost or a `dev.sqlite`
file is obviously not production — but with hosted databases every
environment looks identical, and the only signal is the project
ref/host in the URL. Read it, and check it against whatever the repo's
docs call production; stale refs for abandoned projects linger in env
files. If you cannot tell, record `unknown` and say so rather than
guessing. Record it in `access.db_env`. Reporting local seed data as real revenue is the
worst failure this skill can produce, because it looks like success.

**Check the role is read-only.** Often the only credential in the repo
is a service-role / admin key that bypasses row-level security and can
write. Say so out loud — do not proceed silently on it. In order of
preference: ask for a read-only role or replica; failing that, restrict
every query to existing aggregate views if the project ships them
(`v_*` views, an ops SQL catalogue); failing that, warn plainly and
query aggregates only.

Record the verdict into `access.db` and `access.traffic` — this is
what makes `/ask` work later without re-discovery.

If no DB access exists, say so plainly and continue. The skill still
works on client events alone, with a stated ceiling: you will not be
able to report revenue from the database.

**Done when:** `access.db` and `access.traffic` each have a concrete
value (a command, `ga4-mcp`, `ga4-service-account`, or `none`).

### 8. Write `funnel.yaml` — then STOP

Copy `templates/funnel.yaml`, fill every key from steps 1–7, write it
to the repo (root unless `ANALYTICS.md` already names a path).
`product` is the app name (directory or `package.json`). `timezone` is
IANA, taken from app config or this machine — do not ask the human to
name a "reporting timezone." `repos` from steps 1–2, `vendors_detected`
from step 3, `access` from step 7, `sources` from steps 5–6, `stages`
from step 5.

**Then stop — and stop means end your response.**

Do not output step 9 or step 10 in the same turn. Do not write
application code. Do not draft the instrumentation "so it's ready."
Ask the confirmation question and **end the message there**, then wait
for the human to actually reply. Continuing past this point without a
real human answer is the single worst failure mode of this skill: it
produces a `funnel.yaml` that looks confirmed, and every number you
ever report afterwards inherits a mapping nobody checked.

Present the proposed funnel and money mapping to the human in plain
language and get **explicit confirmation** before touching any
component. An unconfirmed schema mapping means every later query is
invented.

Ask the way you would ask a developer, not an analyst:

> I think a paid conversion is a row in `orders` where `status='paid'`,
> and revenue is `amount_cents/100`. Correct?

Walk the stages the same way: "landing is a page view on `/`, then they
hit `/app` after signup, then `/checkout`, then a paid row. The only
event you don't fire yet is `begin_checkout`. Correct?"

Revise `funnel.yaml` until they confirm. Confirmation is a yes (or a
correction you then apply). Silence is not confirmation.

When they confirm, set `confirmed: true` and `confirmed_on:` (today's
date) in `funnel.yaml`. `/ask` and `/report` refuse to run while
`confirmed` is false — that is what makes this gate real rather than
polite.

**Done when:** the human has confirmed the money mapping and the
stages, and `funnel.yaml` matches that confirmation.

### 9. Instrument the gaps

Only the 1–3 gaps from step 5. Only through the wrapper from step 4.

- **Client-side:** page views, the primary CTA, each step of a
  multi-step flow.
- **Server-side:** `purchase`, entitlement grant, long-job completion.
- **Never** fire PII (raw email, phone) into a third-party analytics
  vendor. If a wrapper call would pass them, strip them.

Do not add events "while you're there."

**`purchase` fires in both places, on purpose.** This is the one event
that is deliberately duplicated:

- **Client-side** (thank-you page) — the ad pixel needs browser
  context (`fbclid` / `gclid`, cookies) to attribute the sale to the
  ad click. Fire it, or the ad platform sees zero conversions and
  stops optimising the campaign the user is paying for.
- **Server-side** (payment webhook) — the database row is the revenue
  truth. Ad blockers and closed tabs mean the client event is always
  undercounted.

Pass the order/transaction id on both so the ad platform can
deduplicate. Report revenue from the database, never from the pixel.

**Done when:** each gap has exactly one wrapper call at the moment it
happens, or you have stated why a gap cannot be instrumented (usually:
no backend).

### 10. Write `ANALYTICS.md`

Write it into the user's repo from `templates/ANALYTICS.md` so a future
agent session does not re-discover everything. Point it at `funnel.yaml`.
List vendors, the wrapper path, the event catalogue, the read-only
query command, and what is still un-instrumented.

**Done when:** `ANALYTICS.md` exists and a stranger-agent could query
from it without repeating steps 1–7.

---

## Hard rules (every mode)

- Never add a new analytics vendor or SDK.
- Never query a table or column that is not in `funnel.yaml`. Ask the
  human to extend the file instead.
- Everything against the user's database is **read-only**. No writes,
  no migrations, no `DELETE`, no `DROP`, no `UPDATE` — not even to
  "fix" something you noticed. Prefer a read-only role; wrap sessions
  in `BEGIN TRANSACTION READ ONLY` where the database supports it, and
  put a `LIMIT` on anything that is not an aggregate.
- **Query aggregates, not rows.** `count()`, `sum()`, `percentile`. Do
  not `SELECT *`, and do not pull raw user rows into the conversation.
  Query results enter the agent's context, which means they are sent
  to whichever AI provider is running this skill. Aggregates keep that
  boundary clean; raw rows do not.
- Never run `/ask` or `/report` while `funnel.yaml` has
  `confirmed: false`. Go finish setup step 8.
- Never state a number you did not query. Show the SQL. Cite `n`.
- Full judgment rules: `references/doctrine.md`.
- Operating procedures for `/ask` and `/report`: `references/operate.md`.
