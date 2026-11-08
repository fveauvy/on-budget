---
status: accepted
---

# The watch shows a glance relayed by the iPhone

The watch app runs no sync engine and holds no credentials. The instance builds the glance for the household selected on the iPhone (safe to spend, the daily budget and pace of the personal and shared plans, stamped with the instance epoch and sync cursor), the iPhone's native `WatchRelay` module fetches it and passes it on over WatchConnectivity, and the watch keeps the copy with the highest (epoch, cursor), a new epoch always replacing it. In the MVP the watch only reads. When watch actions come, `WatchRelay` posts each one with a client-generated action ID to one idempotent endpoint, `POST /api/v1/watch/actions`, keeps unsent actions in a small file and retries them with the same ID; the instance turns each into its named mutation and applies it once, so Swift carries no outbox, mutation-ID sequence or clock. Why: an instance on a private network is out of the watch's reach, since Tailscale, which the reference deployment uses (ADR 0003), has no watch app; watchOS grants a few seconds of background time about four times an hour, and only with a complication on the face, which is too little to sync; and iOS can wake the iPhone app for the watch with no JavaScript running, while background JavaScript runs when iOS chooses, often overnight.

## Considered Options

- A second sync client in Swift inside `WatchRelay`, with its own database, outbox, mutation IDs and clock (`watchos-app.md` §2.5): it reuses `/api/v1/push`, but it is a second implementation of the push side of the protocol, in a second language, for a handful of confirmations.

## Consequences

- Apple calls WatchConnectivity "an opportunistic optimization", so the watch is only as fresh as the iPhone's last relay, and every watch surface shows the glance's age.
- The watch app's floor is watchOS 26.0, because a household watch can't run watchOS 27. Complication transfers reach WidgetKit complications only from watchOS 27, so on watchOS 26 complications refresh only while the watch app runs.
- Alerts target the iPhone, and iOS forwards them while the iPhone is locked and the watch is worn. `WatchRelay` mirrors the watch's hide amounts setting into the app group, and the notification service extension drops amounts when either device's setting is on.
- With no sign-in on the watch, the app lock doesn't apply there: the watch shows amounts only behind its passcode and wrist detection, hides them in Always On and in complications, and offers hide amounts.
- There is no direct mode: the watch never calls the instance. The glance JSON, and later the watch actions endpoint, are the only contracts between the server and Swift, pinned by golden vectors and covered by the client support window (ADR 0018).
