---
status: accepted
---

# Money in integer minor units, time in the household time zone

Amounts are an integer count of minor units plus an ISO 4217 code: `Int64` in Swift, `bigint` plus `char(3)` in PostgreSQL, a JSON integer, and in TypeScript a branded safe-integer `number`, never a `bigint`, which JSON can't carry. Nothing uses floating point: provider decimal strings are parsed exactly, and an amount with surplus decimals is held for review, never rounded. Instants are UTC; bank dates and budget dates are civil dates that never pass through an instant; and the household time zone defines today, periods, pace and due dates. Local times become instants only in TypeScript or Swift, with Temporal's `compatible` policy, and reach SQL as parameters, because PostgreSQL resolves the autumn DST overlap the other way. Why: every runtime handles amounts exactly, household amounts stay far below 2^53, and members see the same today and the same periods wherever their devices are.

## Consequences

- Sync IDs, counters and other 64-bit values travel as strings.
- Splits use the largest remainder, so shares add up to the total, and every formatter call sets its digits and rounding.
- Each household has one base currency, and the MVP has no foreign-currency accounts; such accounts, and their conversion rules, come later.
- Temporal is native on Node and polyfilled on Hermes and in browsers that lack it.
- While travelling, the app's today can differ from the local clock.
