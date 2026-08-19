# Analytics

Written by own-your-funnel during setup. Next session: read this
and `funnel.yaml` instead of re-discovering the tree.

Confirmed by human: `<confirmation-date>` (mirrors `confirmed_on` in
`funnel.yaml`; if that file says `confirmed: false`, setup is not done
and `/ask` and `/report` must refuse).

## Vendors detected

Do not add a vendor that is not listed here.

- `<vendor-name>` (`<class: ga-like | ad-pixel | row-level | cdp | privacy-aggregates | ads | money | ignore>`) — `<what you may ask it>`

## Wrapper

`<path-to-wrapper>` — all new events go through this. Do not call
`gtag` / `fbq` / vendor SDKs from components.

If a thin wrapper was created during setup, it lives at the same path
and fans out to: `<destination-list>`.

## Event catalogue

| event | where it fires | purpose |
|---|---|---|
| `<event-name>` | `<file-or-handler>` | `<one-line purpose>` |

Gaps instrumented this setup: `<event-list-or-none>`.

## Funnel

Canonical mapping: `<path-to-funnel.yaml>`

Stages (confirmed): `<stage-names-in-order>`

Money: a paid conversion is a row in `<sources.money.table>` where
`<sources.money.paid_predicate>`. Revenue is
`<amount_column-or-count-times-unit_price>` in
`<sources.money.currency>`. Timestamp:
`<sources.money.timestamp_column>`.

## How to query (read-only)

Never write. Never dump tables to disk.

- DB: `<access.db>` — hits `<access.db_env>`, read-only role: `<access.db_readonly>`
- Aggregates only. No `SELECT *`, no raw customer rows: query output is
  sent to the AI provider running the agent.
- Traffic: `<access.traffic>`
- Events table: `<sources.events.table-or-none>`
- Timezone: `<timezone>`

## Change ledger

`<path-to-annotations>` (from `annotations` in `funnel.yaml`). The
human logs price / ads / paywall / landing changes here. Copy
`templates/annotations.md` if the file does not exist. Do not invent
entries.

## Repos

- Frontend: `<repos.frontend>`
- Backend: `<repos.backend-or-missing>`

## Un-instrumented

What is known to be missing, and why. Be explicit.

- `<gap>` — `<reason, e.g. no backend repo so purchase cannot fire on the payment webhook>`
