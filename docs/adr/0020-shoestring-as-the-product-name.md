---
status: accepted
---

# Shoestring as the product name

The product is called Shoestring, from "on a shoestring": on a tight budget. It replaces the working name on-budget, and the repository was renamed to match. The maintainer wanted an English name that says "on a budget". The product isn't sold, so the name needs no registered trademark and no domain: there is no store listing (ADR 0019), and instance addresses don't depend on the name (ADR 0011). On 2026-11-08, the US and French App Stores had no finance app called Shoestring, and GitHub used the word only in that idiom.

## Considered Options

- Coined French names from two screening rounds (Mensaire, Tirlou, Firmette, among others), chosen for free domains and no identical mark in classes 9, 36 and 42. Rejected: once the product wasn't for sale, those checks stopped mattering, and plain English won.
- Keeping on-budget: literal, but generic.

## Consequences

- Permanent identifiers use `shoestring`: the URL scheme, token prefixes and `$extensions` keys. Bundle IDs, the app group and Keychain access groups append `shoestring` to the publisher's own reverse-DNS prefix, set in untracked configuration (ADR 0019).
- No trademark policy asks forks to rename (ADR 0013).
- Package, CLI and NixOS module names are `shoestring`. The research notes keep the working name.
