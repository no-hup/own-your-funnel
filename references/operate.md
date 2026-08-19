# Operate

How to run `/ask` and `/report` after setup. Load this file in those
modes. Load `references/doctrine.md` with it.

`funnel.yaml` is the contract. If it is missing, go back to setup in
`SKILL.md` — do not improvise a mapping in the conversation. If it
exists but `confirmed` is `false`, **stop**: the mapping was never
checked by a human. Finish setup step 8 first.

---

## `/ask` — answer one question

One question, one query (or a small set that answers that question),
then stop. Do not expand into a report.

### Steps

1. **Read `funnel.yaml`.** Also read `ANALYTICS.md` if present, for the
   wrapper path and known gaps — not as a substitute for the yaml.
2. **Refuse if the question needs a table or column that is not
   mapped.** Say which key would have to be added (`sources.money.*`,
   `sources.events.*`, a new stage) and ask the human to extend
   `funnel.yaml`. Do not guess `amount` vs `amount_cents`.
3. **Build the query from mapped fields only.**
   - Money questions → `sources.money.table`, `paid_predicate`,
     `amount_column` (or `count * unit_price` if `amount_column` is
     empty), `currency`, `timestamp_column`. If `amount_is_cents`,
     divide by `100.0` — never `100`, which truncates to whole
     currency units. For funnel conversion counts (not total revenue)
     add `new_conversion_predicate` so renewals are excluded.
   - Cut every window in `timezone` from `funnel.yaml`, not in the
     database server's default timezone.
   - Journey / drop-off questions → `sources.events` (or a row-level
     vendor already in `vendors_detected`). Sessionize from `ts` +
     `user_id` (falling back to `anonymous_id` for pre-signup stages) +
     `event_name`. Filter stages with each stage's `event` and optional
     `where`. Follow one cohort forward — see doctrine, "Count a cohort,
     not a calendar window."
   - Stage-count questions with GA4 only → GA4 aggregate event counts
     are a legitimate funnel shape. Give them the counts and say the
     stages are not necessarily the same people.
   - Traffic questions → `access.traffic`. If it is `none`, say you
     cannot answer traffic from here.
   - Paid stage with `source: money` → always the money table, never
     a client `purchase` event.
4. **Run it read-only** via `access.db` (or the traffic method in
   `access.traffic`). If `access.db` is `none` and the question needs
   the DB, say so and stop. Do not try a write tool. Do not save query
   results to disk.
5. **Answer** with all four of:
   - the SQL (or vendor query) you ran
   - the numbers
   - the sample size (`n`)
   - the one relevant caveat from doctrine (disagreement, small
     sample, mixed population, median-not-mean, directional-only)

Label the conclusion **hypothesis** or **confirmed**.

### Worked examples

The failure mode is a confident unsourced number. These are short on
purpose.

#### 1. "How many people paid last week?"

**Bad:** "Looks like about 20, so conversion is healthy."

**Good:**

```sql
SELECT count(*) AS n_paid
FROM orders
WHERE status = 'paid'
  AND created_at >= (now() AT TIME ZONE 'America/Los_Angeles')::date
                    - interval '7 days';  -- window cut in the funnel.yaml tz
```

23 paid rows (`n=23`) in the last 7 days, timezone `America/Los_Angeles`
as in `funnel.yaml`. **Confirmed** count of DB-paid. Caveat: this is
not "users" if one user can pay twice — it is paid rows. Traffic from
GA4 will not match this; do not divide 23 by a vendor session count
and call it *the* conversion rate without saying both sources.

#### 2. "Why are users dropping off?"

**Bad:** "Checkout is probably the problem, most SaaS apps lose people
there. I'd optimize the form."

**Good:** (only if `sources.events` or a row-level vendor is mapped)

```sql
-- counts are unique users who fired each stage event in the window
-- (replace with the mapped table / columns)
```

Landing 410 → signup 180 (56% drop) → editor 150 (17% drop) →
checkout 90 (40% drop) → paid 23 (74% drop, from `sources.money`).
**Confirmed** worst leak is checkout → paid. Caveat: journeys from
events will undercount vs money, because ad blockers drop client
events and the paid row still lands. If `n_paid < 10`, this is
**directional only**.

Caveat to state whenever stages come from one date window rather than
one cohort: these are not guaranteed to be the same people — someone
who landed last week and paid this week appears only at the bottom.
Cross-sectional funnels can show impossible rates; anchor on a cohort
where the data allows it.

If there is no events table and no PostHog/Mixpanel/Amplitude/Heap:
**do not refuse the question.** Give the stage-count funnel from GA4
aggregate event counts (`page_view`, `sign_up`, `begin_checkout`) plus
paid from the DB, and say that per-user paths and timings are not
available. Refuse only the per-user journey, never the funnel.

#### 3. "What's revenue from Germany?"

**Bad:** any number, if `funnel.yaml` has no country column.

**Good:** "I don't have a country column in `funnel.yaml` (`sources.money`
is `orders.status`, `amount_cents`, `created_at`). I will not guess
`billing_country`. Add it to the file if you want this answered."

---

## `/report` — periodic readout

Write `reports/YYYY-MM-DD.md` (create `reports/` if needed). If the
user prefers no file, print the same document to stdout and do not
write it.

Default window: last 7 days vs the previous 7 days, cut in
`timezone`. State the window at the top. If `n_paid` for the window
is under ~10, mark the whole readout **directional only**.

### When a section's source is missing

**Omit the section** and list it under **Data gaps**. Do not
fabricate. Do not leave an empty heading.

| Section | Required source | If missing |
|---|---|---|
| Funnel | `stages` plus any of: `sources.events`, a row-level vendor, or GA4 aggregate event counts | With GA4 only, print the stage-count funnel and mark it cross-sectional. Only if there is no event source at all, print the paid count alone and say the in-between stages cannot be counted |
| Money | `sources.money` + `access.db` | Omit; data gap |
| Traffic and sources | `access.traffic` not `none` | Omit; data gap |
| Timings | row-level events (`sources.events` or PostHog/etc.) | Omit; data gap |
| Ads | pixel in `vendors_detected` or ads API credentials | Omit; data gap |

### Section order

#### 1. TL;DR

The biggest lever first, in one or two sentences a busy founder reads
alone. A number belongs in it. Not "things look stable."

#### 2. Funnel

Stage-by-stage counts and drop-off percentage **vs the previous
stage**. Name the single worst leak.

Use each stage's `event` + optional `where`, except a stage with
`source: money`, which is counted from `sources.money`.

Run the schema drift check (doctrine): a mapped event going to zero
while a similarly-named new event appears at about the same rate is a
rename, not "conversions down 100%."

#### 3. Money

Conversions and revenue from the DB, with the window stated.

`paid_predicate` defines a paid row; `new_conversion_predicate`
defines a *funnel conversion* (renewals are revenue, not conversions —
report them separately). Revenue is `amount_column` (divided by
`100.0` when `amount_is_cents`) or `count * unit_price` when
`amount_column` is empty. If both `amount_column` and `unit_price` are empty, report
conversion count only and list revenue as a data gap. Show currency.
Show `n`.

#### 4. Traffic and sources

Only if a traffic vendor is reachable via `access.traffic`.

Sessions / pageviews and top sources. Include the **vendor-vs-DB
reconciliation paragraph** every time this section appears: the two
will disagree; they measure different things; here is the gap this
window (vendor sessions vs DB-paid, as a ratio, not as a verdict).

#### 5. Timings

Only if a row-level events source exists. **Medians**, not means.
Examples: median time landing → signup, signup → paid. Join on `user_id`, falling back to
`sources.events.anonymous_id` for stages that happen before signup —
do not silently drop the top of the funnel because it has no
`user_id`.

#### 6. Ads

Only if a pixel or ads API with credentials exists. Spend, attributed
conversions *as the ad platform reports them*, and a reminder that
platform-attributed conversions are not `sources.money`.

#### 7. Anomaly

Exactly one specific thing that changed, with numbers. Not a list.
Not "overall traffic is slightly up." A named stage, source, or
revenue figure vs the previous window.

#### 8. Data gaps

What could not be measured and why. Be explicit; this is how the user
learns what to fix. Missing backend, `access.db: none`, no events
table, traffic `none`, pixel without API credentials — name them.

#### 9. Actions

At most three. Each tied to a number above. Each concrete enough to
do this week. Written for a developer:

> the checkout step loses 60% — the form asks for a phone number
> before showing price; try showing price first

beats "optimize checkout."

Do not pad to three. One real action is better than two slogans.

### Change ledger

Read `git log` and the file named in `annotations` (default
`annotations.md`) **only when the window contains ledger entries**.
If it does, mention them next to the anomaly or the action they
might confound. List confounders. Do not claim causation.

If the window has no ledger entries, do not invent a narrative from
commit messages outside the window.

### Docs

If `ANALYTICS.md` or `funnel.yaml` comments are stale (a vendor
removed, a wrapper moved, a stage renamed), **propose** the update.
Never apply it silently.
