# own-your-funnel

An **agent skill** (a folder of markdown your AI coding agent loads) that
does the analyst thinking in *your* repo. You shipped a site, you run
some ads, you get about 1,000 visits a day, you already have Google
Analytics and/or a Meta Pixel, and you have a database. You are a
competent developer and you do not want another analytics SaaS — and
you should not have to know what a funnel is.

It is **not** an analytics product. No new SDK, no hosted service, no
account to create. The agent reads your code, proposes a short
conversion path, adds the 1–3 events you are actually missing through
the wrapper you already have, then answers questions and writes a
periodic markdown report from queries against your own database.

## Start here

Copy the folder in (below), then paste this to your agent:

```
Read own-your-funnel/SKILL.md and run setup. I don't know what I should be tracking.
```

That is the whole interface.

## Where your data goes

Worth being straight about, because most tools this shape are not.

The skill is markdown — it has no server and phones nothing home. But
the agent runs SQL against your database and reads the results, and
those results go into the model's context, which means they are sent
to whichever AI provider you run this with (Anthropic, Google, OpenAI).

So the rules it follows: query aggregates only (`count`, `sum`,
medians), never `SELECT *`, never pull raw customer rows, never write
to your database, never send emails or phone numbers to an analytics
vendor. Point it at a read-only role and it cannot do damage even if
it tries.

## Requirements

- An AI coding agent that can load a skill folder (Claude Code, Grok
  CLI, Cursor, or anything that takes the same `SKILL.md` convention).
- An analytics vendor already in the repo, and/or a database.

## Install

Copy this folder into the agent's skills directory:

```
.claude/skills/own-your-funnel/
.grok/skills/own-your-funnel/
.cursor/skills/own-your-funnel/
```

Same markdown in all three. Then ask the agent to set up analytics.

## Usage

First run is **setup**. The agent infers the tree, finds the sibling
backend (purchase truth is never in the pixel), detects vendors,
finds your existing `track()` wrapper, proposes at most 8 stages, and
maps money to a real table. Then it **stops** and asks you to confirm.

You will not be asked "what's your funnel?" You will be asked something
like: *I think a paid conversion is a row in `orders` where
`status='paid'`, and revenue is `amount_cents/100`. Correct?*

After that, `/ask` answers one question with SQL shown, and `/report`
writes `reports/YYYY-MM-DD.md`.

### What setup sounds like

```
You: set up analytics. I don't know what I should be tracking.

Agent: This is a Next.js frontend; backend is ../app-backend.
       GA4 (gtag, G-1A2B3C4D) and a Meta Pixel (fbq) already fan out
       through lib/analytics.ts. I will not add another vendor.

       Path I inferred:
         1. landing  — page_view on /
         2. signup   — sign_up (already firing)
         3. editor   — page_view on /app
         4. checkout — begin_checkout (you don't fire this yet)
         5. paid     — a row in orders where status = 'paid'

       I think a paid conversion is a row in orders where
       status='paid', and revenue is amount_cents/100. Correct?

You: yes, but the table is payments, and the flag is paid = true.

Agent: Updated. Confirm the five stages and I'll add begin_checkout
       through lib/analytics.ts, then write ANALYTICS.md.
```

### Example `/report` (abridged)

```
# 2026-04-15  (last 7 days vs previous 7, America/Los_Angeles)

## TL;DR
Checkout → paid is the leak: 90 began checkout, 23 paid (74% drop).
Revenue $667 from 23 payments.

## Funnel  (stage counts from GA4 events; cross-sectional, not a cohort)
landing 410 → signup 180 (−56%) → editor 150 (−17%) →
checkout 90 (−40%) → paid 23 (−74%). Worst leak: checkout → paid.

## Money
23 paid rows in payments where paid = true.
Revenue $667 (amount_cents/100, USD). n=23.

## Traffic and sources
GA4: 612 sessions. Top: google / organic 38%, facebook / paid 29%.
Vendor vs DB: 612 sessions vs 23 paid. These measure different things
(ad blockers, iOS, cross-device). Neither is "wrong." Pixel Purchase
this week: 11 — expected to be below the DB.

## Anomaly
facebook / paid sessions 180 → 92 (−49%) week over week. Meta daily
budget was cut 2026-04-10 per annotations.md [config]. Confounders:
Tue–Thu only in this window; Meta learning phase possible. Not
causation.

## Data gaps
No events table and no PostHog/Mixpanel, so the funnel above is GA4
event counts — stage totals, not the same people followed through.
Per-user paths and median time-to-pay are not available. Add a
first-party events table if you want those. No ads API credentials —
ads section omitted.

## Actions
1. Checkout loses 74% (90 → 23). The form asks for a phone number
   before showing price; try showing price first.
2. signup drop is 56% on mobile (n_mobile_signup=40, directional).
   The signup button sits below a carousel — move it above.
```

## What it will not do

- Will not add a new analytics vendor or SDK.
- Will not spray `gtag()` / `fbq()` through your components — new
  events go through the existing wrapper, or one thin wrapper it
  proposes if you have none.
- Will not write to your database (no migrations, no `DELETE`, no
  dumps to disk).
- Will not reconstruct per-user journeys from GA4 alone. It will still
  give you the stage-count funnel (that is what finds the leak), but
  ordered paths and time-between-steps need an events table or
  PostHog/Mixpanel/Amplitude/Heap, and it will say so instead of
  faking them.
- Will not report a number it did not query, or a column that is not
  in `funnel.yaml`.
- Will not send PII (raw email, phone) to a vendor.
- Will not query a column that is not in `funnel.yaml`.

## License

MIT © 2026 Shaurya. See `LICENSE`.
