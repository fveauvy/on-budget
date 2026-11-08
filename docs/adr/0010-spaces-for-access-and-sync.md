---
status: accepted
---

# Spaces for access and sync

Every record belongs to exactly one space: each member has a personal space, and each household one shared space. A member sees and syncs their own personal space and their household's shared space, nothing else, so access derives from membership alone. There are no per-ledger grants and no viewer, editor or admin roles: the instance owner is the only administrative role, and nothing crosses households (ADR 0001). Sharing an account moves it and its transactions into the shared space, and no shared-space row ever references a personal-space row: a shared account's transaction that counts in its holder's private plan keeps its category, note and plan in an annotation row in the holder's personal space, and a payment from a private account that counts in the shared plan appears there as a shared entry, showing the payee, amount, date, payer and shares, never the account. A visibility change moves rows between spaces under one sync ID and, like a membership change, makes every device of the household bootstrap again. The server runs one `can(person, action, record)` check per mutation, which sees the record's space, its account holders and its plan's member, and PostgreSQL row-level security on the household backs it up. Why: the maintainer's model has no joint ledgers across households and no household admins, and spaces are also the unit of sync, so one pull covers all of a member's spaces in one snapshot.

## Considered Options

- Personal, household and joint ledgers with per-ledger viewer, editor and admin grants: they allow joint ledgers across households, which ADR 0001 excludes, and turn every pull and permission check into a per-ledger computation.

## Consequences

- Visibility has two values, private and shared, and a shared account or personal plan shows the household everything but what a private plan keeps. New accounts and personal plans start private; the shared plan and joint accounts are always shared, so in a household of three a joint account of two members is shared with the third. A balances-only level stays reserved for later.
- Only account holders edit their accounts' transactions, and every member edits the shared plan.
- A record that would link two members' private records has no space, so such links, like transfers paired across two private accounts, wait for a design that keeps each side in its own space.
- The sync simulator checks that no shared-space row references a personal-space row and that devices converge across visibility changes.
