---
status: accepted
---

# Native apps only through the maintainer's TestFlight internal group

The native apps reach devices only through the maintainer's TestFlight internal group: one app record, the iOS app with its embedded watch app; members join as App Store Connect users with the Developer or Marketing role limited to that app; and Xcode Cloud archives each release tag, which the maintainer adds to the internal group once the reference instance runs that release. There is no App Store or unlisted listing, external TestFlight group, Developer ID build or push relay. Other self-hosters use their instance's web app, or build the native apps from source with their own Apple developer account, setting the team ID, bundle ID, app group, URL scheme and APNs key from untracked configuration. This is the maintainer's choice, and the instance model calls for it: APNs keys stay with the publisher's team, so an instance can push only to builds its own owner signed, and a relay would be a service run for other people (ADR 0001); App Review would need a demo backend it can reach, which no Tailscale-only instance is, and expects financial apps from a legal entity, while the internal group needs no review.

## Consequences

- Nothing to prepare for App Review: no demo mode, no DSA trader self-assessment, no guideline 5.1.1(ix) question.
- TestFlight builds expire after 90 days, so a release ships at least that often.
- Each instance lists the native apps it accepts, by OAuth client ID and callback scheme, in its own configuration.
- Bundle IDs, the app group and Keychain access groups come from the product name under a reverse-DNS prefix the publisher controls, and the URL scheme is `shoestring` (ADR 0020); none comes from an instance address, and all are fixed before the first TestFlight build, since renaming them strands installed data.
