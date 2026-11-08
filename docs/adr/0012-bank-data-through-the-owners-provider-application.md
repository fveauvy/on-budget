---
status: accepted
---

# Bank data through the owner's own provider application

The owner registers their own provider application with the data provider, Enable Banking, in its free restricted mode, and connects only accounts they hold, joint accounts included. The instance generates the application's key, the owner pastes only its certificate into Enable Banking's Control Panel, and the key never leaves the instance. Other members bring their accounts in by importing bank files: OFX, QIF and CSV sit behind the same connector interface, which also covers banks the data provider can't read. Why: Enable Banking's terms cover only accounts linked by the person they belong to, and they forbid giving third parties access to its API. An owner reading their own accounts through their own application, on an instance they run, is the case the free mode is made for, so nothing waits for a written answer. Losing the free tier would cost latency, not data. Connectors are built into the server, never plugged in at runtime.

## Considered Options

- One provider application per person, members included, all run by the owner's instance. Members' applications would need Enable Banking's written agreement (SP-01). This option comes back only if a household needs members' bank connections and Enable Banking agrees.
- One instance-wide application for the household: it would break Enable Banking's terms.
- A contract with a licensed provider: out of proportion for one household.

## Consequences

- The data provider sends no webhooks, and instances accept no inbound connections (ADR 0003), so the worker refreshes on a schedule. Each consent gets two jittered background refreshes a day, and each account at most four accesses in any 24 hours, retries included. An access is one request to an account's balances or transactions endpoint, with each page counted, until SP-13 shows how each bank counts.
- Only two kinds of request carry the owner's IP address and user agent as PSU headers: a refresh the owner explicitly asks for, and the backfill right after a consent. Opening an app doesn't count.
- The provider key lives only in the database, encrypted per record and bound to the owner. Raw provider payloads are encrypted per household, with a retention limit. The key-encryption key stays outside the database and its backups, and only the worker loads it.
- A joint account the owner holds comes in once, through the owner's connection, and is shared with the household (ADR 0010).
- SP-13 records how long the certificate stays valid and whether the Control Panel accepts a replacement.
- One TypeScript parser reads bank files: in the apps, for a preview, and on the server, which parses them again, deduplicates and owns the result.
- Two assumptions await spikes: that Enable Banking accepts a callback on a tailnet-only host (SP-12, CONN-3), and that banks accept a tailnet address as the owner's IP (SP-13, CONN-4).
- No screen scraping, which needs stored bank passwords, and no FinanceKit, which isn't available in France.
