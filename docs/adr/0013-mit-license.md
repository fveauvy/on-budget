---
status: accepted
---

# MIT license

Shoestring is MIT-licensed, the whole repository. The research recommended AGPL-3.0-or-later, as Firefly III, Ghostfolio and Kresus use, so that modified hosted versions stay open, with an additional permission for Apple's stores, named in every file's header, and DCO sign-off. The maintainer chose MIT: anyone may reuse the code, connectors and parsers included; contributors face no licensing ceremony; and distributing the apps through Apple, TestFlight included, raises no conflict, whereas the FSF reads Apple's terms as further restrictions that GPL-family licenses forbid.

## Consequences

- Closed forks and closed hosted modifications are allowed. The name isn't a registered trademark, and no policy asks forks to rename (ADR 0020).
- App builds still ship only permissive or MPL-licensed dependencies, checked in CI, because GPL-family code in the apps would bring the conflict back.
- There is no CLA and no required DCO: contributions come in under MIT.
- Released code stays MIT even if a later version changes license.
