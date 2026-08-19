# Change ledger

The agent cannot see that you doubled the ad budget, changed the
price, or flipped a paywall. `git log` only shows code. Without this
file it will attribute a metric move to the last commit.

Add a line when you change something that could move traffic, signup,
checkout, or revenue. The operator reads this during `/report` only
for entries inside the report window, and will list them as
confounders — it will not claim they caused the move.

## Format

```
YYYY-MM-DD: short description [flag1,flag2]
YYYY-MM-DDTHH:MM: description with a time [pricing]
```

Flags (use only these): `pricing`, `paywall`, `landing`, `config`.

- `pricing` — price, plan, trial length, coupon
- `paywall` — what is gated, when the gate appears
- `landing` — homepage / ads landing copy, hero, CTA
- `config` — ad budget, pixel, feature flag, analytics wiring

## Examples

```
2026-03-12: doubled Meta daily budget $20 → $40 [config]
2026-03-18: price $19 → $29 on the Pro plan [pricing]
2026-04-02T14:00: email required before checkout [paywall]
2026-04-10: new hero headline and primary CTA on / [landing]
```
