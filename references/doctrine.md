# Doctrine

Judgment for own-your-funnel. Load this whenever you pick a source,
state a number, or write a report.

The user has little product/analytics sense. These rules are the
discipline that stops the agent from guessing like a chatbot.

---

## Half 1 — source ranking

Pick the richest **allowed** source per question, in this order. "Allowed"
means it is already in the repo or already reachable in this environment.
Do not add a vendor to reach a higher tier.

### 1. Money → app DB

Orders / payments tables, a `paid` column, or Stripe / Razorpay /
Lemon Squeezy / Polar webhook rows.

The backend always wins for revenue. The ad pixel is not revenue.

### 2. Journeys / step-level drop-off

Two different questions hide under "drop-off." Do not conflate them —
this distinction decides whether most installs of this skill are
useful or useless.

**Stage counts** — how many people reached each step. "410 fired
`page_view`, 90 fired `begin_checkout`, 23 paid." This is a funnel
shape, and it is enough to find the leak.
*Sources:* a first-party events table, a row-level vendor, **or GA4
aggregate event counts.* GA4 will happily tell you how many times each
named event fired. Use it. A coarse funnel from GA4 event counts is
the normal case for this skill's target user, and it answers "where am
I losing people."

**Per-user journeys** — *this* person did A then B then C, ordered,
deduplicated, cohorted. Median time between steps. Path branching.
*Sources:* a first-party events table, else PostHog / Mixpanel /
Amplitude / Heap **if already installed**. GA4 cannot do this and
neither can privacy-first traffic tools. Say so rather than
approximating a journey from sessions.

So: with GA4 only, give them the stage-count funnel and state that
per-user paths and timings are not available. Do not refuse the whole
question — refusing a funnel you could have counted is the most
expensive mistake in this document.

### 3. Traffic and acquisition source

GA4, else Plausible / Fathom / Umami / Simple Analytics.

Sessions, source/medium, landing pages. Not step-level paths.

### 4. Ads

Meta / Google Ads / TikTok / LinkedIn pixels, and their ads APIs **only
if credentials already exist**.

Pixels are an instrumentation and ad-platform mechanism, not a source
of revenue truth. Use them to answer "did the pixel fire" and "what did
the ad platform report," never "how much did we earn."

### 5. Ignore for the funnel

Sentry, Vercel Analytics, Hotjar, Clarity.

Detect them so we do not add a redundant tracker. Never report funnel
counts, drop-off, or revenue from them.

---

## Vendor fingerprint table

Grep the repo for these. List what exists. Never add a row that is not
already there.

| Grep for | Class | What you may ask it |
|---|---|---|
| `gtag(`, `GTM-[A-Z0-9]{5,}`, `G-[A-Z0-9]{6,}`, `@next/third-parties/google`, `googletagmanager` | GA-like aggregates | Sessions, source/medium, named conversion event **counts** (a stage-count funnel is fine). **Not per-user journeys.** Match `G-` with the character class shown — a bare `G-` also matches CSS and UUIDs. |
| `fbq`, `react-facebook-pixel`, `facebook-pixel` | Ad pixel (Meta) | Whether the pixel is wired, which events it is sent. Not revenue. |
| `AW-` | Google Ads | Ads conversion tagging. Not revenue, not journeys. |
| `posthog-js`, `posthog-node`, `PostHogProvider` | Row-level product analytics | Journeys and drop-off. Prefer this over inventing a first-party events table. |
| `mixpanel`, `amplitude`, `heap` (SDKs / `heap.track`) | Row-level product analytics | Same as PostHog: journeys OK. |
| `@segment/analytics`, `analytics-node`, `rudder-sdk`, `RudderAnalytics` | CDP | Read its **destinations**. The CDP is a pipe, not the store of truth. Report from the destinations (or the DB), not from "Segment says." |
| `plausible`, `fathom`, `umami`, `simple-analytics`, `simpleanalytics` | Privacy aggregates | Traffic only (pageviews, referrers). Journeys are impossible — say so. |
| `ttq`, `TiktokPixel`, `lintrk`, `linkedin_insight`, `_linkedin_partner_id` | Ads only | Pixel fire / ad-platform stats if credentials exist. Not revenue. |
| tables/models named `events`, `app_events`, `analytics_events` | First-party events | Best journey source. Sessionize from `ts` + user + `event_name` as mapped in `funnel.yaml`. |
| `stripe`, `razorpay`, `lemonsqueezy`, `lemon-squeezy`, `polar` webhooks / SDK handlers | Money | Revenue and paid conversions. Always prefer the resulting DB row over the webhook payload at query time. |
| `@sentry`, `Sentry.init` | Ignore for the funnel | Detect only. Errors, not funnel. |
| `@vercel/analytics` | Ignore for the funnel | Detect only. |
| `hotjar`, `hj(` | Ignore for the funnel | Detect only. Session replay is not a funnel count. |
| `clarity.ms`, `clarity(` | Ignore for the funnel | Detect only. |

---

## Half 2 — analyst rules

Each rule has a *why*. The human may not know why it matters; you must.

### Never state a number you did not query

Show the SQL (or vendor query). Cite the sample size (`n`).

*Why:* An unsourced number is indistinguishable from a guess, and the
user will treat it as fact.

### Traffic and money will disagree

Traffic numbers (from the analytics vendor) and money numbers (from the
DB) **will** disagree, often by 2–3×. Ad blockers, iOS tracking
restrictions, and cross-device gaps cause this.

Explain the gap in every report rather than letting the user think the
agent is wrong. Never call either source "wrong" — they measure
different things. Vendor = "sessions we observed in the browser."
DB = "rows we collected on our server."

*Why:* A founder who sees 400 GA sessions and 23 paid rows will assume
one of those is a bug. It usually is not.

### Prefer median over mean for timings

A single stuck request destroys a mean.

*Why:* One 45-minute tab left open on checkout makes "average time to
pay" look like a disaster.

### Never mix populations in one number

Do not combine mobile + desktop, or paid + organic, into one rate
without saying so.

*Why:* A blended conversion rate hides that mobile is leaking and
desktop is fine (or that ads convert and organic does not).

### Small sample → no causal claims

Rule of thumb: fewer than ~10 conversions in the window → label the
finding **"directional only"**.

*Why:* 2-out-of-8 is not a 25% conversion rate you can act on. It is
noise.

### Label every conclusion **hypothesis** or **confirmed**

*Why:* The user cannot tell a well-queried fact from a plausible story
unless you mark it.

### The change ledger is `git log` plus `annotations.md`

Never model memory. Correlate changes with metric moves, but list
confounders (ad budget change, day of week, an ad platform's learning
phase) and do not claim causation.

*Why:* You cannot see that they doubled the Meta budget. Without
`annotations.md` you will blame the checkout form.

### Every report must name **one specific anomaly**

Without this the report decays into "traffic is steady" every week and
the user stops reading it.

*Why:* The point of a periodic readout is the thing that changed, not
a restatement of the funnel.

### Schema drift check

If a mapped stage suddenly stopped firing and a similarly-named new
event started firing at about the same rate, the event was renamed —
say that. Do not report "conversions down 100%."

*Why:* Renames look like collapse. They are usually a deploy.

### Bound what enters context

Query aggregates. Cap row-level pulls. Never dump a whole table into
the conversation.

*Why:* Cost, privacy, and you will start pattern-matching on individual
users instead of the funnel.

### Sanity-check surprises

If a result is surprising, query it from a second angle before
asserting it.

*Why:* Off-by-one dates, the wrong `paid_predicate`, and timezone
mismatches produce confident fiction.

### Refuse unmapped columns

If a question needs a table or column that is not in `funnel.yaml`,
refuse. Ask the human to extend the file instead.

Guessing column names is how the agent starts inventing numbers.

*Why:* `amount` vs `amount_cents` vs `total` is a 100× error.

### Never send raw email or phone to a third-party analytics vendor

*Why:* PII in GA/Meta is a compliance hole and you do not need it to
count a funnel.

### Failure isolation

If one source is unavailable, produce the report from the rest and
list the missing one under data gaps. Never abort the whole report.

*Why:* A down GA4 MCP must not also hide the fact that checkout is
losing 60%.

### Count a cohort, not a calendar window

To measure drop-off, follow **one group of people forward**. Do not
count each stage inside the same date window and divide.

*Why:* Someone who landed in week 1 and paid in week 2 appears as a
purchase with no landing. Do that across a funnel and you get
conversion rates above 100% and negative drop-offs — numbers that are
obviously broken, reported confidently. Anchor on users who entered
the funnel in the window, then look for their later steps **even if
those fall outside it**. If the data cannot support that (GA4 event
counts, for instance, cannot), say the funnel is a cross-sectional
snapshot and that the stages are not the same people.

### Do the window arithmetic in the configured timezone

`now() - interval '7 days'` runs in the database server's timezone,
which is almost always UTC. `funnel.yaml` names an IANA timezone; cast
to it (`created_at AT TIME ZONE 'UTC' AT TIME ZONE '<tz>'`, or the
equivalent) and cut on day boundaries there.

*Why:* Off by 7-8 hours means a chunk of one day lands in the wrong
week, and week-over-week comparisons move for no reason. This is the
most common way a report shows a change that did not happen.

### Money arithmetic is decimal, never integer

If the amount is in cents, divide by `100.0` — not `100`. Cast
explicitly where the database needs it.

*Why:* Postgres and SQLite return an integer from integer division.
`2999 / 100` is `29`. Every price silently loses its cents and revenue
comes out low, consistently, forever.

### A renewal is not a conversion

If the app bills recurring, filter new conversions with
`sources.money.new_conversion_predicate`. Total revenue includes
renewals; the *funnel* counts only first payments.

*Why:* Month 2's renewals get attributed to month 2's traffic, so
conversion rate climbs every month while nothing has improved.

### Top of funnel has no `user_id`

Landing and pricing-page views happen before signup, so `user_id` is
NULL there. Join those stages on the anonymous/device id
(`sources.events.anonymous_id`) and switch to `user_id` only after the
identity is known.

*Why:* Requiring `user_id` everywhere silently drops the entire top of
the funnel — and a funnel that starts at signup hides the leak that is
usually the biggest.

### Know which database you are pointed at

Check `access.db_env`. If it is local or a dev copy, label every
number in the output as non-production.

*Why:* Seed data reports beautifully. A founder acting on local test
revenue is worse off than one with no report at all.

### Read-only against the user's database

No writes, no migrations, no `DELETE`. Never dump production data to
disk.

Prefer a read-only role or a replica. Wrap sessions in
`BEGIN TRANSACTION READ ONLY` where supported. `LIMIT` anything that
is not an aggregate.

*Why:* This skill answers questions. It does not operate the database.
And "read-only" here is an instruction, not an enforced permission —
if the connection string can write, only the database can actually
stop it.

### Query results leave the machine

Everything a query returns enters the agent's context, which means it
is transmitted to whichever AI provider is running this skill. Query
aggregates. Never `SELECT *`, never pull raw user rows, never page
through a customer table.

*Why:* "It runs locally" is true of the skill and false of the data.
An aggregate is a number; a row is somebody's email address.
