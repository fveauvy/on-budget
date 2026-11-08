---
status: accepted
---

# One message catalog for every platform

All text the apps show lives in `l10n/messages/{en,fr}.json`, in a subset of ICU MessageFormat: plain arguments, at most one plural and one select per message, and no number or date arguments, since values are formatted before they are passed. An in-repo generator emits typed FormatJS modules for the shared UI and the server, and Apple String Catalogs with Swift accessors for the watch, the widgets and the notification service extension, plus each target's Info.plist strings. English is the source, and French is required in the same pull request. Why: the same text appears in TypeScript and in Swift targets, so one key means one translation everywhere, the maintainer and agents review plain diffs of two files, and the subset maps onto Apple's catalogs.

## Consequences

- Xcode's string extraction isn't used, and Apple's device variants and inflection are reached only through extra keys in the Swift targets.
- Push alerts carry an event type and raw values, and the notification service extension and the web service worker render their text from the catalog. The server never formats a notification, so declarative Web Push isn't used, and category names and amounts never reach APNs in clear text.
- No translation platform in v1; one can import the ICU files later.
