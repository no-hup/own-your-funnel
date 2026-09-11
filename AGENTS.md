# AGENTS.md

An agent skill that maps a product's conversion funnel from its own code, adds
only the tracking events that are actually missing, and then answers analytics
questions and writes reports by querying the app's own database read-only. No
new analytics vendor, no SaaS, no SDK.

## Use this when

The user is asking for product/funnel analytics on a codebase they own, e.g.:

- "set up analytics" / "what should I be tracking?"
- "why are users dropping off at checkout?"
- "how's my conversion rate?" / "how many signups converted last week?"
- "give me a weekly report on the product"
- "am I tracking the right events?" / "is my Meta Pixel firing on purchase?"
- "add conversion tracking to this app"
- they type `/ask` or `/report`
- an `ANALYTICS.md` or `funnel.yaml` already exists and needs checking against the code

Good fit when: the repo has a real user flow ending in a payment, a database
you can read, and probably a GA4 tag or a Meta Pixel already glued in
somewhere.

Do not use it to add a new analytics vendor or SDK, to build a dashboard
product, or for marketing/ad-spend analysis. It reports on the product's own
data only.

## How to use it

Install: copy the repo folder into the agent's skills directory, e.g.
`.claude/skills/own-your-funnel/` (also works under `.grok/skills/`,
`.cursor/skills/`, and anything else that loads skill folders).

Entry point is `SKILL.md`. If your agent doesn't auto-load skills, read it
directly: "Read own-your-funnel/SKILL.md and run setup."

`SKILL.md` has a router at the top. Three modes:

- **setup** — runs once, when there's no `funnel.yaml`. Ten ordered steps:
  identify the tree, find the backend, detect existing analytics vendors, find
  the existing tracking wrapper, infer the funnel (max 8 stages), find where
  money lives in the DB, find and *test* a read-only query path, write
  `funnel.yaml`, stop for human confirmation, instrument the 1–3 real gaps,
  write `ANALYTICS.md`.
- **ask** — one question, one query, show the SQL, answer.
- **report** — last 7 days vs the previous 7: drop-off, revenue, anomalies,
  actions.

Supporting files: `references/doctrine.md` (source ranking and the rules for
stating numbers — load before judging any source), `references/operate.md`
(procedures for ask/report), `templates/funnel.yaml`, `templates/ANALYTICS.md`,
`templates/annotations.md`.

## Gotchas

- **Step 8 is a hard stop.** Write `funnel.yaml`, ask the human to confirm the
  money mapping and the stages, and end the message. Do not continue into
  instrumentation in the same turn. `confirmed: false` blocks `/ask` and
  `/report` on purpose.
- **Never query anything not in `funnel.yaml`.** If a question needs a new
  table or column, ask the human to extend the file.
- **Read-only, aggregates only.** `count()`/`sum()`/`$group`, never `SELECT *`,
  never raw user rows — query results go into the agent's context and therefore
  to whatever AI provider is running it. No writes, ever.
- **Check which database you're on.** Hosted dev and prod look identical except
  for the project ref. Reporting seed data as revenue is the worst failure this
  skill has, because it looks like it worked.
- **Test the read path before recording it.** A `psql` that exists but can't
  connect leaves `/ask` dead later.
- **Subscriptions inflate conversion.** Renewals are new paid rows. Find the
  first-payment predicate.
- **`purchase` is fired twice on purpose** — client-side for ad attribution,
  server-side for revenue truth. Same order id on both. Revenue always comes
  from the DB.
- Prerequisites: an agent that loads skill folders, an existing analytics setup
  and/or a database in the repo. If the backend is missing, say plainly what
  can't be instrumented instead of faking it on a thank-you page.
