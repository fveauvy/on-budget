---
status: proposed
---

# Bank data through each person's own provider application

Each person who wants bank data registers their own provider application with the data provider, Enable Banking, in its free restricted mode. The instance generates the application's key, the person pastes only its certificate into Enable Banking's Control Panel, and the key never leaves the instance. There is no instance-wide provider application: a person's application reads only the bank connections that person created. Why: Enable Banking's terms cover only accounts linked by the person they belong to and forbid giving third parties access to its API, so one application for a household would break them, while a contract with a licensed provider is out of proportion for one household. OFX, QIF and CSV import sit behind the same connector interface, so a member without a provider application, or a bank the data provider can't read, still gets data, and losing the free tier would cost latency, not data. Connectors are built into the server, never plugged in at runtime. This stays proposed until Enable Banking answers SP-01 in writing and the maintainer accepts it.

## Consequences

- The data provider sends no webhooks and instances accept no inbound connections (ADR 0003), so the worker refreshes on a schedule: two jittered background refreshes per consent a day, and at most four accesses per account in any 24 hours, retries included, where an access is one request to an account's balances or transactions endpoint, each page counted, until SP-13 shows how a bank counts. Only a refresh a member explicitly asks for, and the backfill right after their consent, carry their IP address and user agent as PSU headers, and only for consents they gave; opening an app doesn't count.
- Provider keys live only in the database, encrypted per record and bound to their person, and raw provider payloads are encrypted per household with a retention limit. The key-encryption key stays outside the database and its backups, and only the worker loads it.
- Members other than the owner use file import until Enable Banking confirms in writing that their application may run on an instance the owner administers (SP-01 (CONN-1)). The same answer settles the certificate's validity and whether an application can replace it.
- One TypeScript parser reads bank files in the apps, for a preview, and on the server, which parses them again, deduplicates and owns the result.
- Two assumptions await spikes SP-12 (CONN-3) and SP-13 (CONN-4): that Enable Banking accepts a callback on a tailnet-only host, and that banks accept a tailnet address as the member's IP.
- No screen scraping, which needs stored bank passwords, and no FinanceKit, which isn't available in France.
