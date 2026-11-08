# Spec

What Shoestring does, as rules to build and test. It turns `PRODUCT.md` into behavior and leaves the design to [`docs/architecture.md`](docs/architecture.md) and the ADRs in `docs/adr/`. Terms follow `GLOSSARY.md`.

Rules marked (proposed) wait for the maintainer's confirmation; a marker covers only the clause it ends, or the bullet when it ends a label. Unmarked rules come from `PRODUCT.md`, the ADRs, the maintainer's decisions, regulation, or technical choices the maintainer delegated. Every pending decision is listed under [Open questions](#open-questions). Examples use euros; `app.<domain>` stands for the instance address. The MVP is built in the slices of [`docs/factory.md`](docs/factory.md#build-order)'s Build order.

## Scope

Each line ends with its source in `PRODUCT.md`.

### MVP

- An instance with one owner, households including a test household, and members added by invite; no public sign-up. _Users; Self-hosted, open source; Test household_
- Passkey sign-in, an app lock with Face ID or Touch ID, and step-up before sensitive actions. _Authentication_
- Bank connections to the supported banks (BoursoBank, Crédit Agricole Aquitaine, Crédit Mutuel du Sud-Ouest), read-only, each through the account holder's own provider application; Enable Banking is the recommended data provider, pending SP-01 and ADR 0012, and other members import files until it confirms its terms in writing. _Bank sync_
- Import of OFX, QIF and CSV files in every app, the fallback for every bank and every member. _Bank sync_
- Manual accounts, cash first: manual entries, cash counts, and withdrawals recognized as transfers. _Manual accounts_
- A personal plan per member and one shared plan with its split. _Plans in a household_
- Zero-based allocation on calendar-month periods. _Budget engine_
- Safe to spend and the daily budget for each plan. _Budget engine_
- A rollover rule on every category. _Budget engine_
- Limits with threshold and pace alerts. _Budget engine_
- The review inbox and categorization rules. _Budget engine; Principles 1 and 3_
- Recurring items, detected or declared, and the forecast. _Recurring items_
- Goals: a safety buffer and big purchases. _Goals_
- Money owed both ways, the net-cost rule and settlements; contributions and member balances. _Money owed; Plans in a household_
- Private or shared visibility for each account and personal plan. _Configurable sharing_
- Offline-first apps that sync once back online and merge concurrent edits predictably. _Offline-first_
- The iPhone app with its Apple Watch app, the Mac (the web app in the Dock) and the web app at full parity. _Apps_
- Light and dark themes from one enforced design system. _Themes; Design system_
- English and French, regional preferences for any European country, one base currency per household. _Localization; Regional settings_
- Each member's export of their data, and deletion. _Principle 4_

### Next (just after the MVP)

- Loans with a schedule: remaining principal, installments, interest. _Loans_
- Quick cash entries and provisional entries from Apple Pay taps on the iPhone, through App Intents. _Product Purpose: as little manual work as possible_
- Review on the watch, a limits page, more complication families, a Smart Stack widget and actions on forwarded alerts. _Apps_
- iPhone widgets for safe to spend and the review inbox. _Operating Context: daily short sessions_
- A watch widget the member sets to their personal plan or the shared plan. _Apps_
- Transfers paired across two members' private accounts. _Principle 1_
- Import profiles a member builds for another bank by mapping its columns. _Bank sync_

### Later

- Cash entry on the watch, a watch widget near shops, an interactive watch widget, goals on the wrist. _Apps_
- An OCI image and a compose bundle for self-hosters who don't run NixOS, once the server is stable (ADR 0018). _Self-hosted, open source_
- Accounts in a currency other than the base currency. _Regional settings_
- A "balances only" visibility level. _Configurable sharing_
- A contribution planner for a future joint account. _Users: joint accounts can be added later_
- Several people in one browser profile, and a memory-only mode for shared computers. _Apps_
- Importing an export into another household or instance. _Principle 4_
- A native Mac app once Expo supports macOS, with a watched import folder. _Stack_
- A share extension for bank files on the iPhone. _Bank sync_
- Contracted data providers, e-mailed bank exports and bank-app notifications as sources. _Bank sync_
- Reports, trends, and forecasts with what-if scenarios beyond three periods. _Product Purpose_
- More languages, e-mail, and sign-in through an external identity provider. _Localization; Authentication_

### Not planned

- Public sign-up, a service run for other people, an App Store listing or a push relay. _Users; Native app distribution_
- Initiating payments of any kind. _Bank sync: read-only_
- Bank screen scraping, and FinanceKit, which isn't offered in France. _Bank sync: secure and stable_
- Anything that needs an inbound connection from the internet, webhooks included, and a watch that reaches the instance without its iPhone. _Private network friendly_
- End-to-end encryption of budget data, which the instance must read to ingest, alert and push. _Principle 4_
- Right-to-left scripts. _Localization_
- A native Android app; the web app serves other devices. _Apps_
- Bill negotiation, subscription cancellation, credit scores, ads, affiliate offers, streaks and leaderboards. _Product Purpose; Principle 3_
- Telemetry. _Principle 4_

## Domain rules

### Periods and allocation

- A period runs from the first day of a month at 00:00 to the first day of the next month at 00:00, in the household time zone. All plans of a household share the same periods.
- A transaction counts in the period of its transaction date, the day the card was used, else of its booking date, else of its value date, else of the household-zone date on which the instance first saw it. A pending transaction's date is provisional, and its booked twin applies the rule again, so a pending payment that books in the next month stays in its month.
- Incoming money is either income, which adds to the plan's money to allocate, or a refund in a category, which lowers that category's spending.
- A plan's money to allocate in a period: the unallocated money of the previous period, plus income counted in the period, plus what reset rules return, minus the previous period's overspending (Rollover rules).
- Unallocated is the money to allocate minus the period's allocations. Zero-based planning brings it to zero; a negative figure is allowed and flagged as over-allocated.
- A member can hold income for the next period, so a salary paid on the 28th funds the next month.
- Allocation edits are operations: setting an allocation, which applies with a conflict notice if another member changed it meanwhile, or a reallocation. Concurrent reallocations add up.
- Members allocate the current and the next period. Past periods stay editable, and everything derived from them is recomputed.
- Every category is fixed, flexible, or a goal's own category. Deleting a category that holds money or history asks which category takes them over; archiving keeps the history.
- Each new plan starts with default categories, shown in each viewer's language until renamed.

### Safe to spend and daily budget

- A category's spending: the booked and pending transactions, provisional entries and manual entries in it, at their net cost and minus its refunds; a matched entry and its booked transaction count once, and transfers never count.
- Remaining = allocation + money carried in − spending.
- Safe to spend of a plan = the sum of its flexible categories' remaining, minus the overspending of its fixed categories, minus their shortfalls: the part of a fixed category's upcoming occurrences in the period that its remaining doesn't cover. Unallocated money and goals' categories don't count.
- Safe to spend runs to the end of the plan's current period, in the household time zone, and a payday only shapes income forecasts.
- Daily budget = safe to spend before today's spending ÷ the days left in the period, today included, rounded down to the minor unit. It stays fixed during the day, and the apps also show it minus today's spending.
- When safe to spend is zero or less, the daily budget shows zero and the plan is over.
- Pace of a plan = S ÷ ((A + C) × e), where S is its flexible categories' spending so far, A their allocations, C the money carried into them, and e the elapsed share of the period, today counted as elapsed. Each limit uses the same formula on its own category.
- A plan is on track, near when its pace exceeds 1 + the margin, or over when safe to spend is zero or less.

### Rollover rules

- When a period ends, each category's positive remaining follows its rollover rule:
  - carry over: it moves into the same category in the next period;
  - move to a goal: it moves into the chosen goal's category;
  - reset: it returns to the plan's money to allocate in the next period.
- A negative remaining never moves into a category: the overspent category starts the next period at its new allocation, and the overspending comes out of the next period's money to allocate.
- A goal's category always carries over.
- Defaults: flexible categories reset; fixed categories carry over, so non-monthly bills build up, and the next period's proposed allocation only tops them up to the forecast.
- Rollovers are computed, never stored, so editing a past period updates every later one.

### Limits and alerts

- A limit can be turned on for any category. It watches the category's allocation plus the money carried in, so a reallocation moves it. Limits start on for flexible categories and off for the others.
- A limit is on track, near once spending reaches its first threshold or its pace exceeds 1 + the margin, and over once spending exceeds it.
- Defaults: thresholds at 80 % and 100 %; a pace margin of 10 %, checked from the 4th day of the period.
- A threshold alert fires once per threshold, category and period; a pace alert at most once per category and week.
- Other alerts: an occurrence 10 % or more off its expected amount, or missed by 3 days; a bank account whose projected balance falls below zero before the period ends; a consent near expiry, and a bank connection that needs renewal (Bank connections).
- The instance evaluates the affected plans' alerts after every refresh and import, and after every pushed change to a plan's spending, allocations or attribution, at most once a minute per plan; an hourly sweep checks thresholds and pace. An alert on a personal plan goes to its member only, one on the shared plan to every member, and a consent alert to the account holder only.
- Delivery: APNs to the iPhone, which iOS forwards to a worn watch, and Web Push to installed web apps; each device writes the text in its own language. Time Sensitive: a limit exceeded, a projected overdraft, a bill or debt due today; the rest are Active. Quiet hours: from 21:00 to 08:00 in the device's zone, other alerts wait.
- With hide amounts on, no notification shows an amount or a payee. A member can mute each kind of alert, per plan.
- A test household's alerts and reminders reach no device until the owner turns them on.

### Review inbox and categorization rules

- A member's review inbox holds: new transactions, booked or pending, on accounts they hold, joint accounts included; provisional entries still unmatched after 7 days; cash-count gaps; suggested transfers, matches, settlements and suspected duplicates; newly detected recurring items.
- Each item arrives with a category, a payee and an attribution filled in, plus up to three suggested categories, from categorization rules and then from the payee's history.
- Confirming accepts category, payee and attribution together and records who confirmed. A transaction that other members may also edit leaves every inbox once one of them confirms it.
- On the iPhone, a swipe right confirms and a swipe left skips; skipped items come back at the end, and undo works for the session. The web and the Mac offer the same actions from the keyboard.
- From the inbox or the transaction, a member can divide it into category lines, which must sum to its amount, divide it with counterparties under the net-cost rule, or attribute it to the shared plan.
- Categorization rules match on payee, bank label, amount range and account, and set category, payee and attribution. They run on the instance when transactions arrive, in the member's order, the first matching rule setting each field. A member's own choice always wins: no rule overwrites it.
- After a member changes a category, the app offers a rule for that payee.
- A member's rules apply to the accounts they hold; the shared plan's rules apply to joint accounts.
- The payee comes from the bank label, cleaned of common bank prefixes, card numbers and dates; renaming a payee applies to its later transactions.
- The inbox is a daily or weekly ritual: when it isn't empty, a prompt comes at 20:00 in the device's zone each day, or once a week on a day and at a time the member picks; the member can move it or turn it off.

### Recurring items and the forecast

- A recurring item has a payee, an account, a plan and a category, an expected amount (fixed, or a range for variable bills), a frequency (weekly, monthly, every N months, yearly) and a next date. A member can declare one at any time: a salary, a rent, an installment.
- Detection: three booked transactions with the same payee, amounts within 10 % of their median and a regular interval (±3 days for monthly ones) produce a suggestion in the inbox, which the member confirms, edits or dismisses. Pending transactions never count.
- An occurrence is upcoming, paid, paid with a different amount, or missed. A transaction matches an occurrence when its payee matches, its amount is within 25 % of the expected amount, or of a variable bill's range, and its date within 5 days; the member can undo or move a match.
- One occurrence can be skipped or changed without changing the series.
- The forecast runs from today to the end of the third period ahead. It feeds safe to spend (fixed-category shortfalls), the proposed allocations, and each account's projected balance at the end of the period: its current balance plus the upcoming occurrences on it.
- Until loans arrive (Next), loan installments are recurring items in a fixed category.

### Goals

- A goal belongs to one plan: a safety buffer, a big purchase or any other target, with a target amount and an optional date.
- A goal is funded through its own category: allocations, remaining money moved to it, and spending from it, such as the purchase itself.
- Safety buffer: the target defaults to 3 months of the plan's average monthly income over the last 3 periods, or to months of spending if the member prefers; the goal shows how many months it covers.
- With a date, the suggested allocation per period is (target − saved) ÷ the periods left, rounded up to the minor unit; the goal is on track while saved keeps up with that schedule.
- A goal can show a linked manual savings account's balance next to its own figure; the plan never depends on that balance.
- A reached goal can be kept, raised or closed; closing returns its money to the plan's unallocated money.

### Accounts: bank, manual, joint, transfers, cash counts

- Accounts are bank accounts (fed by a bank connection, by imports, or both), card accounts and manual accounts. Each has one account holder, or several for a joint account.
- A joint account lives in the shared space, and its transactions count in the shared plan. When two holders connect the same joint account, the instance keeps one account fed by both consents.
- A connected account shows the bank's latest balance and its date; other accounts show the balance computed from their transactions.
- Transfer detection: an outgoing and an incoming transaction of the same amount on two accounts of the household, within 3 days, are suggested as one transfer, and a categorization rule can confirm such pairs. Only accounts one member can see are paired, so a transfer between two members' private accounts waits for Next.
- A transfer is never spending or income. Contributions and member balances: a transfer into a joint account pays a contribution, and one between two members can settle a member balance.
- Each member gets a private cash account when they join. A bank transaction that looks like a cash withdrawal, by its bank label or its card category code, is suggested as a transfer to the account holder's cash account.
- Joint-account cash: a cash withdrawal from a joint account is suggested as a transfer to the household's shared cash account, a manual account in the shared space whose entries and cash counts count in the shared plan; cash a member takes from a joint account for their own use is shared-plan spending in a category, not a transfer.
- A manual entry needs an amount and takes today's date by default, a category, a payee and a note; every app records one offline.
- Cash count: the member enters what the account holds, and the gap with the recorded balance becomes one transaction in the inbox, spending to categorize when cash is missing, income to categorize when there is more. The inbox suggests a cash count every 7 days.
- Savings that banks don't expose through PSD2 (livrets, PEL, life insurance) are manual accounts, fed by transfers and cash counts.
- A deferred-debit card is a card account: its payments count in the month the card was used, and the bank's monthly debit of their total is a transfer from the bank account to the card account.
- Removing a bank connection keeps its accounts and their history.

### Money owed, settlement and the net-cost rule

- A debt or an expected reimbursement has a counterparty, an amount, an optional due date, a note and an optional reminder; it is open, partly settled or settled. Money owed lives in its member's personal space.
- Net-cost rule: when a member pays for someone else, they divide the transaction into shares, by amount, in equal parts or by percentage, summing to the total. The member's own share, the net cost, counts in the category; each counterparty's share becomes an expected reimbursement.
- Shared-plan payments: for a payment that counts in the shared plan, such as one from a joint account, the shared plan's category counts the household's share, and each counterparty's share becomes an expected reimbursement of the shared plan, kept in the shared space, which every member sees and any member can settle when the payment reaches a joint account. Where a settlement paid into one member's own account goes follows open question 1.
- An expected refund from a company, such as health insurance, works the same way: the member marks the expected part when categorizing the payment.
- An incoming payment whose payee matches a counterparty and whose amount equals one open item, or several together, is suggested as their settlement. Settling an expected reimbursement or a member balance is neither income nor spending.
- A smaller payment settles part of an item, and a cash settlement is a manual entry on the cash account.
- When a reimbursement closes below its expected amount, the difference counts in the original category and period.
- Paying back a debt counts as the member's own spending in a category, and a debt due within the period appears in the forecast.
- A reminder notifies the member at its time: as a local notification on their iPhone, offline included, and by Web Push from the instance on their web devices while those are online. An item past its due date shows as overdue. The wording stays neutral: "{counterparty} owes you 12 €".

### Personal and shared plans and splits

- Each member has one personal plan; the household has one shared plan, which every member can edit.
- Attribution: a transaction counts in exactly one plan. It defaults to the holder's personal plan for a member's own account and to the shared plan for a joint account; a categorization rule or the member can change it.
- Split: 50/50 (equal shares with more members), a fixed percentage per member, or in proportion to income. Income split: each member chooses the figure it uses, one they enter or their personal plan's income in the previous period; the setup tells them that the others can work it out from the split they see; the figures are fixed when the period starts. A changed split applies from the current period.
- Shares are whole minor units given by largest remainder, so they sum to the amount; ties go to members in the order they joined.
- Contribution: each period, the split turns the shared plan's total allocation into what it asks of each member. Each personal plan allocates that amount, prefilled, to a fixed category for its contribution, whose allocation leaves the personal plan at once: nothing remains in it and nothing rolls over. A transfer into a joint account then pays the contribution and is never spending.
- The shared plan's income for a period is what its members allocate as contributions, plus income booked on a joint account.
- Shared entry: when a member pays shared spending from their own account and attributes it to the shared plan, the shared plan counts the full amount and the payer's personal plan counts nothing more, since their contribution already left it; each other member owes the payer their share as a member balance.
- Member balances: balances between two members net out; a transfer between them, or a cash handover either of them records, settles them once a member confirms it as a settlement; with a joint account, a transfer from it to the payer can settle a shared entry instead.
- For each period, each member sees the contribution asked, what they paid into a joint account, and every member balance.

### Sharing and visibility

- Visibility is private or shared, set per account and per personal plan. A member's accounts and personal plan start private. The shared plan is always shared, and so are joint accounts, so in a household of three a joint account of two members is shared with the third. A shared account or personal plan shows the household everything but what a private personal plan keeps (below); "balances only" stays reserved.
- A shared account shows the household its balance and transactions, and only its holders edit its transactions.
- A shared personal plan shows the household its categories, allocations and safe to spend; only its member edits it.
- Private data never reaches another member's device. A transaction of a shared account that counts in its holder's private personal plan shows the household its payee, amount and date, without its category, note or plan. A shared entry shows payee, amount, date, payer and shares, never the account. A partner can't tell that a private account exists.
- Changing visibility needs step-up and the instance, so it works only online. Making an account or a plan private again removes it from other members' devices at their next sync; shared entries already made stay.
- When they redeem an invite, and next to every visibility control, members read that private hides data from other members in the apps, not from whoever runs the instance.
- Hide amounts is separate from visibility: each device has its own setting, and the watch has one of its own, which the iPhone also applies to the alerts it forwards to the watch.

### Households, members, the test household and instance administration

- The owner bootstraps the instance from the host: a single-use link, valid one hour, creates the owner's person, membership and first household, and registers their passkey. Until then the instance serves only a "not initialized" page.
- Only the owner creates households, each with a name, a kind (regular or test, fixed at creation), a base currency and a household time zone. The owner becomes a member of each household they create.
- The owner adds people with invites, which are stored hashed, revocable and logged. A signed-in person who redeems an invite gains a membership rather than a second person.
- A person can be a member of several households and switches between them; data, alerts and pending changes never cross households. The watch, the widgets and the glance follow the household selected on the iPhone.
- The owner is the only administrative role; households have no admin.
- Removing a member (owner, step-up) ends the membership, revokes that person's devices for the household, revokes their consents where the data provider allows, and deletes their provider application's key when it was their last membership. Their personal space is deleted after 30 days, during which the owner can undo the removal. Members export their own data before they leave.
- A test household holds generated fictitious data and takes no bank connection. It shows a permanent banner, on the glance too, and imports only files its generator wrote, which carry a marker.
- The owner fills a test household from its settings in the owner's administration, with fictitious accounts, 12 months of transactions, recurring items and goals. It is where the owner tries new features; upgrades are rehearsed on a throwaway instance restored from a backup, since migrations are instance-wide.
- Instance administration (owner only, with step-up): households, invites, memberships, devices, backup status and the instance's audit events. Other members' bank connections appear there only as counts per health state, naming no bank, member or account; each member watches their own connections' health in their own settings. Host commands cover bootstrap, owner recovery, restore, backup verification, diagnostics and secret generation.
- A member who lost every passkey gets a single-use recovery link from the owner; the owner recovers through a host command.

### Bank connections, consent renewal and refresh

- Provider application, once per person: the instance generates the key pair and a long-lived certificate, whose validity SP-01 settles; the person registers an application with the data provider (with Enable Banking, a restricted-mode application) and pastes its ID; the instance checks that it is active, in production, allowed to read accounts and lists the instance's callback. Nobody uploads, downloads or sees the private key.
- Until the data provider confirms in writing that members' applications may run on an instance the owner administers (SP-01), only the owner's provider application is used; other members import files.
- Connecting a bank (account holder, step-up): pick a supported bank, give consent with strong customer authentication at the bank (a browser sheet in the native apps, a full-page redirect on the web, never an embedded web view), and come back through `https://app.<domain>/connect/enable-banking/callback`. A "Return to Shoestring" button and polling cover a bank app that doesn't hand back.
- The member then picks which accounts to add and their visibility, private by default. The instance backfills each account's full history at once.
- A consent lasts the bank's maximum, at most 180 days, minus a margin of one day, counted in UTC.
- Health: ok, expiring (14 days or less left), needs renewal, bank down or rate-limited, with the time of the last successful refresh.
- An access is one request to an account's balances or transactions endpoint, each page counted, until SP-13 shows how a bank counts. At most 4 accesses per account in any 24 hours, retries and catch-ups included, and at least 6 hours of back-off after a rate limit.
- Background refresh: 2 refresh slots per consent per day at jittered times; the first fetches balances and transactions, the second transactions only, each from 7 days before the last booked date, which leaves one access for a retry. A bank gets more slots, or balances in every slot, only once SP-13 has measured how it counts.
- Explicit refresh: pull-to-refresh or the refresh button fetches at once, only for the consents that member authorized, with their PSU headers and outside the background limit. The backfill right after a consent is user-present too, with the PSU headers of the member's return to the callback; where a bank refuses them (SP-13), it runs within the background limit and may get less than the full history. Opening an app syncs with the instance and may use a refresh slot left that day, without PSU headers. The watch never triggers a refresh.
- A joint account connected by both holders takes one fetch per slot across both consents.
- Pending transactions count at once. A booked transaction replaces a pending one when both carry the same bank reference; otherwise the pairing needs the same account and payee, a booked date within 10 days and the same amount, or any amount once the bank stops returning the pending one, as when a card hold books lower. The booked twin keeps the member's choices, and a pending transaction the bank stops returning without a twin is removed.
- Dedupe works by count, on keys that are never unique. Within one source, a transaction's key is its account, its bank reference (the data provider's entry reference, or a file's FITID) when present, and a fingerprint (date, amount, cleaned label, and the running balance when present), plus its rank among identical keys in the batch; a batch adds only the transactions whose key and rank aren't already there, so two identical payments on one day both stay, and a reference the bank reuses for two transactions keeps both. Across sources, such as a file over a bank connection, the fingerprint without the running balance and the rank decide, and a row that matches only by date and amount goes to the inbox as a suspected duplicate.
- An amount with more decimals than its currency allows holds its transaction back for review; it is never rounded.
- Renewal works before or after expiry: a new consent for the same bank, the accounts reattach by themselves, the old consent is deleted. The instance alerts the account holder on each of their devices 14 days and 3 days before expiry and on the day. Other members see a neutral "needs renewal" on shared accounts; when a bank ends many consents at once, each affected holder's alert says the bank ended it.
- A consent that covers no account explains that new accounts must first be added in the data provider's control panel.

### File import

- Formats: OFX 1.x and 2.x, QIF, and CSV through an import profile. Each supported bank ships one; profiles for other banks come later (Next).
- Entry points: a file picker and drag-and-drop in the web app, the Dock app included; the document picker and "Open in" on the iPhone.
- The app reads the file on the device and previews it, offline too: count, date range, new and duplicate transactions. The instance reads it again and owns the result, which replaces the preview at the next sync.
- The target account is matched from the file's account identifier, picked by the member, or created.
- Transactions already present, from an earlier file or a bank connection, are skipped by the dedupe rule (Bank connections); importing a file twice adds nothing, two identical payments in one file both import, and a row that overlaps a bank connection only by date and amount waits in the inbox as a suspected duplicate.
- The import profile fixes separator, decimal mark, date format and encoding (UTF-8 or Windows-1252); a QIF file with ambiguous dates asks once.
- Imports don't count against the daily refresh limit, and imported transactions enter the review inbox.

### Currencies

- Each household has one base currency, any ISO 4217 code, chosen at creation and changeable only while the household has no transactions.
- In the MVP every account is in the base currency, and a bank account or a file in another currency is refused with an explanation.
- A card payment abroad that the bank books in the base currency keeps its original amount for display.
- Amounts are exact minor units, formatted by each person's regional preferences, signed by the formatter, and never told apart by color alone.

## Key flows

### Owner bootstrap and first run

1. The owner deploys the instance and runs the bootstrap host command with the first household's name; it prints a link and a QR code, valid one hour.
2. On a device that reaches the instance (on the tailnet, in the reference deployment), the owner opens the link on `app.<domain>`, registers a passkey, and sets the household's base currency, its time zone (prefilled from the browser) and their regional preferences.
3. The owner opens the web app, or installs the iPhone app: the maintainer's household from the maintainer's TestFlight internal group, other owners from their own build. They sign in through the browser sheet with the passkey and turn on the app lock; the app suggests a watch complication.
4. The owner sets up their provider application, connects a bank, and picks accounts and their visibility; the backfill runs.
5. A first review proposes transfers, recurring items and categories from the history; the owner confirms or edits them.
6. The app proposes a first plan from the last three months: fixed categories from recurring items, flexible ones from average spending, a safety buffer goal. The owner allocates until nothing is unallocated.
7. Optionally, the owner creates a test household and fills it with generated data.

### Adding a member

1. The owner gives the person's devices access to the instance's network (the tailnet, in the reference deployment), then creates an invite for the household, with step-up, and shows its QR code or sends its link. For the maintainer's household, the maintainer also adds the person to App Store Connect and the internal TestFlight group (ADR 0019).
2. Within 72 hours, the person opens it, reads that private data is hidden from other members but not from whoever runs the instance, registers a passkey and becomes a member; their personal space, personal plan and cash account are created.
3. The member sets their regional preferences, opens the web app or installs the native apps, and signs in.
4. If the household now has two members, the shared plan asks for its split; its next allocation prefills each member's contribution.
5. The member imports bank files, or sets up their own provider application once the data provider allows it, and decides each account's visibility.

### Connecting a bank

1. In the bank connections settings, the account holder picks a supported bank, with step-up.
2. The app opens the bank's consent page; the member passes strong customer authentication, often in the bank's own app.
3. The bank returns to the instance's callback; the instance completes the consent, and the app comes back by itself or through "Return to Shoestring".
4. The member picks accounts and their visibility, private by default.
5. The backfill runs, new transactions fill the review inbox, and the connection shows its health and expiry date.

### The monthly allocation

1. On the first day of a period, the home screen leads to the allocation of each plan the member can edit.
2. Rollover rules have run: carried amounts sit in their categories; reset money and any overspending show in the money to allocate.
3. The app proposes allocations: fixed categories topped up to the forecast, flexible ones at last period's allocation, goals at their scheduled amount, the contribution the split asks.
4. The member adjusts and reallocates until unallocated is zero, holding money for the next period if a salary arrives late.
5. Members allocate the shared plan together, often on the Mac or the web; each personal plan then shows its contribution.

### The daily review

1. The home screen shows safe to spend and the daily budget of the personal plan and the shared plan, their pace, and the inbox count.
2. The member opens the inbox and confirms each item with one swipe, or changes its category, attribution or shares.
3. Changing a category offers a rule, a cash withdrawal offers a transfer, and a payment for others offers a net-cost division.
4. Once the inbox is empty, the home screen and the watch's glance show the updated figures.

### An alert

1. A background refresh brings a payment that takes the Groceries category past 80 % of its limit.
2. The instance evaluates limits and pace and alerts the plan's member, or every member for the shared plan; each iPhone writes "Groceries: 80 % of the limit used, 12 days left" in its language, without amounts when hide amounts is on.
3. If the iPhone is locked and the watch worn, iOS forwards the alert to the watch.
4. Tapping it opens the category, where the member can reallocate. The 80 % alert won't come back this period.

### Settling an expected reimbursement

1. The member pays 120 € for a dinner for four; in the inbox, they divide it: their net cost is 30 €, and three counterparties owe 30 € each.
2. The category counts 30 €, and three expected reimbursements open, each with an optional reminder.
3. A 30 € incoming payment from one counterparty arrives and is suggested as their settlement; the member confirms it.
4. That item closes; the others stay open, and their reminders fire if set.

### A cash count

1. The member withdraws 100 €; the inbox suggests a transfer to their cash account, and they confirm it.
2. During the week, they record two cash entries in the app.
3. The inbox suggests a cash count; they count 42 € against 60 € recorded.
4. An 18 € item appears in the inbox, which they categorize or divide into category lines.

### Renewing consent

1. 14 days before expiry, the instance alerts the account holder on each of their devices, and the connection shows "expiring".
2. The holder taps the alert and passes strong customer authentication at the bank again.
3. The accounts reattach by themselves, without duplicates, the old consent is deleted, and the health returns to ok.
4. If the bank ends a consent early, the next refresh marks it "needs renewal" and alerts the holder; other members see only a neutral badge on shared accounts.

## Apps

### iPhone

- Every MVP feature: home, review inbox, plans (allocation, categories, limits, rollover rules, goals, split), accounts and transactions (search, edit, category lines, attribution), money owed, bank connections and imports, sharing, the household switcher, settings, and the owner's administration.
- Sign-in through a browser sheet with a passkey; the app lock asks for Face ID or Touch ID on cold start and after 5 minutes in the background, and step-up uses the same check.
- Alerts, consent alerts included, through APNs; reminders as local notifications that the iPhone schedules, so they fire offline.
- Relays the glance of the selected household to the watch.
- The maintainer's household installs it from the maintainer's TestFlight internal group; other self-hosters use their instance's web app or build their own (ADR 0019).

### Apple Watch

- The MVP watch only reads: safe to spend of the personal plan and the shared plan, with the daily budget, days left and pace; one complication family that shows each plan's state and pace without amounts; forwarded alerts, without actions; hide amounts; English and French; dark only.
- The watch app runs on watchOS 26.0 and later. On watchOS 26, complications refresh only while the watch app runs; on watchOS 27, the iPhone's relay also updates a complication on the active face.
- Its data is the glance the iPhone relays for the household selected there, and every surface shows the glance's age.
- The watch holds no credentials. It shows no amount without a watch passcode, hides amounts in Always On and complications, and relies on wrist detection.
- Never on the wrist: allocation, settings, bank connections, splits.

### Mac

- The web app added to the Dock from Safari: every web feature with desktop layouts (a sidebar, then a sidebar with an inspector), keyboard shortcuts for search, a new entry and refresh, drag-and-drop import, and Web Push.
- If SP-08 shows the Dock app losing its data, the iPad app on the Mac takes its place.

### Web

- Every MVP feature of the iPhone app, offline included, in Safari 26 or later and current Chrome, Edge and Firefox; installable, with a URL per screen and complete keyboard use.
- One person per browser profile. Their data stays in the browser, signed in for 30 days of use; signing out sends pending changes, warns about any it can't send, and deletes that person's Shoestring data from the profile. Partners who share a computer use separate browser profiles, each with its own Dock app.
- The app lock and step-up use a passkey; alerts and reminders come through Web Push, which iOS offers to Home Screen web apps only.
- Not on the web: Face ID and Touch ID (the app lock uses a passkey) and the watch.

## Non-functional requirements

### Offline-first and sync

- Every app reads and writes offline: allocations, reviews, rules, manual entries, cash counts, money owed, settings and import previews. Bank connections, renewals, explicit refreshes, invites, visibility changes, administration and exports need the instance.
- Pending changes go out within seconds of coming back online; each app shows how many changes haven't synced, and warns after 24 hours without reaching the instance.
- A change reaches the household's other online devices that may see it within 5 seconds.
- Merges are predictable: per field, a member's choice beats a rule, which beats an import, and then the edit that reaches the instance last wins, except that an edit made more than 24 hours before it arrives loses to a change the field received meanwhile; reallocations add up; category lines are replaced as a whole and must sum to the amount; the first match or settlement of a transaction wins; a delete beats an edit unless restored. A refused change disappears, and the app says why.
- A wrong device clock never blocks sync; the app warns when it is off by more than a few minutes.
- After a restore from backup, devices send again the changes they hadn't sent. What the instance had acknowledged after the restored backup is lost, at most the hour since the last off-site backup (Self-hosting operations), and each app says how many of its own changes were.
- Apps of the current and previous minor release keep working for at least 90 days; older apps show "update needed" and keep their pending changes.
- Each device keeps one local copy per household; losing it costs a fresh download, never a change already sent.

### Security and privacy

- Passkeys only, no passwords; the instance address is the passkey identifier and never changes.
- The app lock and step-up as specified above; each device has its own sign-in, revocable from any other device, and reusing a replaced refresh token revokes the device.
- Data a member may not see never reaches their devices. The owner sees a household only as one of its members, plus the instance administration above, and the docs and the invite page say that whoever runs the host can read everything.
- Provider keys stay on the instance, encrypted, and never reach an app or a log; raw provider data is encrypted and deleted after 90 days.
- Logs never hold transaction data, tokens or bank identifiers. An append-only audit log keeps sign-ins, failures, administration, sharing changes, imports and exports for 13 months; the owner sees the instance's events, and the events about a member's own data (imports, exports, visibility changes, bank connections) show only to that member.
- Sign-in, invites, bootstrap and recovery links and step-up are rate-limited.
- Each member's export and deletion on request; no telemetry; no third-party scripts in the web app.
- Data protection: each instance relies on the GDPR household exemption (Art. 2(2)(c)), so the apps have no consent step for data processing and no processing notice. The safeguards stay: raw provider data encrypted and deleted after 90 days, each member's export and deletion, and the invite page's notice about whoever runs the instance. Self-hosters who use Shoestring beyond their household handle their own compliance.
- No feature needs an inbound connection from the internet, and every feature works on an instance reachable only over a private network: consent returns are browser navigations, and push goes out to APNs and Web Push. The reference deployment is reachable only over Tailscale (ADR 0003).

### i18n and regional formats

- English is the source language and every release ships French, which addresses the reader as "vous"; a missing French string fails the build.
- Regional preferences are automatic or explicit, and an explicit value wins on every device. Apple apps follow each device's app language; the web app has its own language choice and, for automatic values, borrows what the person's Apple devices report, else asks once at first run.
- Amounts, dates and times go through one formatting path in every app, so a figure reads the same on the iPhone, the watch and the web; date and time inputs always get the hour cycle and the first day of the week.
- Today, periods, pacing and due dates follow the household time zone; reminders and quiet hours follow the device's zone, and the apps say when the two differ.
- Built-in category names follow each viewer's language until renamed; layouts take 40 % longer text without clipping.

### Accessibility

- Contrast of 4.5:1 for text and 3:1 for controls, focus rings and chart marks, in the light and dark appearances (WCAG 2.2 AA); increased-contrast variants come once an app uses them (ADR 0014).
- Text scales to 200 % on the iPhone and the web, and to 140 % on the watch; the web reflows at 320 CSS px.
- VoiceOver reads amounts, dates and states correctly in English and French and reaches every action; the web works fully from the keyboard, with visible focus.
- Budget states and money direction never rely on color alone; Reduce Motion replaces motion with crossfades.
- Touch targets measure at least 44 pt on Apple devices and 24 CSS px on the web, 44 px on touch screens.

### Performance

Measured in the lab: the iPhone figures with XCTest metrics on a pinned simulator in CI as a proxy, and on a current iPhone before each release (a `ready-for-human` check); the web figure with Playwright under a 4× CPU throttle, over 10 runs.

- iPhone: the first frame within 400 ms on a current iPhone; at most 10 ms of hitches per second while scrolling 500 inbox rows; a response to every tap within 100 ms.
- Web: interaction to next paint at or under 200 ms at the 75th percentile of the runs.
- Safe to spend, the daily budget and counts update within a second of a local change.
- Watch: the cached glance within 1 s; with the iPhone in reach, a glance at most 2 minutes old within 5 s of opening; on watchOS 27, a face complication updated within 15 minutes of a change at the median, while the iPhone is online; on watchOS 26, complications updated each time the watch app opens.
- A device downloads a household with two years of history within 30 s on a home connection.

### Self-hosting operations

- Installed with the project's NixOS module on a host that needs to publish nothing to the internet, under an HTTPS hostname with a DNS-01 certificate.
- Hourly encrypted backups; with an off-site copy, at most one hour of data is lost. The reference deployment keeps two off-site copies, and the diagnostics command warns when an instance has none. A dump before every migration; a verify command; documented restores.
- Upgrades apply forward-only migrations at start. A server refuses a schema whose floor is above its own version, and an unsupported PostgreSQL major; going back to the previous release needs a restore only when the release notes say it raised the floor (ADR 0018).
- Health checks at `/up` and `/api/v1/meta` and a diagnostics command; an emergency kit, printed when the secrets are generated, holds what a rebuild needs that no backup holds: the key that protects stored provider keys and the backup repositories' passwords.
- No e-mail: invites and recovery travel as links and QR codes, alerts as push and local notifications.

## Acceptance criteria

### Instance, households, members and sign-in

- With an empty database the instance serves only the "not initialized" page; a bootstrap link works once and fails after one hour.
- An invite works once and fails after 72 hours or once revoked; no flow creates a person without an invite or the bootstrap.
- An owner who is a member of a test and a production household sees only the selected household's data, and a change made offline in one never lands in the other.
- Switching the iPhone to the test household switches the watch's glance to it at the next relay and the watch's complications once the watch app opens; the test household's alerts reach no device until the owner turns them on.
- A test household refuses a bank connection. It shows its banner on every screen, the watch included, and refuses any file its generator didn't write.
- With a partner's private account connected, the owner's administration counts one more connection in its health counts and names neither the bank nor the account, and the owner's audit view shows no event about it.
- Sign-in on the iPhone and the web uses a passkey on `app.<domain>`, and no screen asks for a password; changing a visibility, connecting a bank, inviting or removing a member and exporting each ask for a fresh check.

### Periods, allocation and rollover rules

- In a Europe/Paris household, a manual entry made at 2026-10-31T23:30Z counts in November. A card payment with transaction date 2026-10-31, booked on 2026-11-02, counts in October.
- Two members, offline, each reallocate 50 € from Outings to Groceries, which holds 100 €; once both sync, Groceries holds 200 € and Outings 100 € less.
- With 2,000 € of income all allocated, unallocated shows zero; allocating 50 € more shows −50 €, flagged as over-allocated.
- With 40 € remaining in a flexible category: reset adds 40 € to the next period's money to allocate, carry over starts the category's next period with 40 € carried in, and move to a goal adds 40 € to that goal.
- A category overspent by 25 € starts the next period at its allocation, with 25 € less to allocate; editing an October transaction in December updates November and December.

### Safe to spend and daily budget

- On 21 November, with 300 € of safe to spend and nothing spent yet that day, the daily budget is 30.00 € (10 days left). After a 12 € purchase, it stays 30.00 € with 18.00 € still to spend that day; on 22 November it is 32.00 €.
- A fixed category with 50 € remaining and an 80 € occurrence still due this period lowers safe to spend by 30 €; unallocated money and goals never change it.
- 10.00 € of safe to spend with 3 days left gives a daily budget of 3.33 €.
- For the same sync state, the iPhone, the web and the watch show the same figures for both plans.

### Limits and alerts

- With a 400 € limit and thresholds at 80 % and 100 %, alerts fire when spending reaches 320 € and 400 €, once each per period.
- A manual entry that takes a category past a threshold alerts its plan's member within two minutes of reaching the instance, without waiting for a refresh.
- On day 10 of a 30-day period, 160 € spent against a 400 € limit, 20 % ahead of pace, fires one pace alert that week.
- An alert on a private personal plan reaches no other member; with hide amounts on, no notification on any device shows an amount or a payee.
- An alert that isn't Time Sensitive raised at 23:00 arrives at 08:00.

### Review inbox, categorization rules and recurring items

- A new booked transaction reaches the account holder's inbox at the first sync after its refresh, with category, payee and attribution filled in; confirm, skip and undo work by swipe on the iPhone and by keyboard on the web.
- No rule changes a category a member set by hand, and category lines that don't add up to the amount are refused.
- Confirming a joint-account transaction on one holder's device removes it from the other holder's inbox after sync.
- Three monthly 15.99 € payments to one payee produce one recurring suggestion; once confirmed, next month's occurrence appears in the forecast and in the account's projected balance.
- A payment 10 % or more above its expected amount marks its occurrence as paid with a different amount and raises one alert; pending transactions never create or match a recurring item.

### Goals, accounts and cash

- A 1,200 € big purchase due in 6 periods, with nothing saved, suggests 200 € per period; a safety buffer shows how many months it covers and never counts in safe to spend.
- A confirmed 100 € withdrawal raises the cash account by 100 € and counts as spending in no plan; a transfer into the joint account counts as spending in neither plan.
- A 100 € withdrawal from the joint account is suggested as a transfer to the shared cash account and counts as spending in neither plan, and a later cash count of the shared cash account puts its gap in the shared plan.
- A cash count of 42 € against 60 € recorded creates one 18 € item to categorize.
- A manual entry made offline on the web shows once on the iPhone after both devices sync.
- A deferred-debit card payment made on 28 October and debited on 5 November counts in October, and the 5 November debit is a transfer.

### Money owed, shared plans and sharing

- A 120 € payment divided into 30 € for the member and 30 € for each of three counterparties counts 30 € in its category and opens three 30 € expected reimbursements; a 30 € incoming payment from one of them, confirmed as that settlement, closes that item and adds no income.
- That incoming payment is suggested as the settlement, and a 20 € payment against a 30 € item leaves 10 € open.
- A 90 € payment from the joint account, divided into 60 € for the household and 30 € for a friend, counts 60 € in the shared plan's category and opens a 30 € expected reimbursement in the shared space, which every member sees and any member can settle when the friend's payment reaches the joint account.
- A reminder fires at its time on the member's iPhone, offline included, and reaches their web devices by Web Push while those are online.
- With a 50/50 split and no joint account, A pays a 1,200 € rent from a private account and attributes it to the shared plan: the shared plan counts 1,200 €, A's personal plan nothing more, and B owes A 600 €; a 600 € transfer from B to A, confirmed as a settlement, brings the member balance to zero.
- Previous-period incomes of 3,000 € and 2,000 € give a 60/40 split, and a 1,800 € shared allocation then asks contributions of 1,080 € and 720 €, after each member saw that the others can work out their income from the split; at 50/50, the shares of 100.01 € are 50.01 € and 50.00 €.
- A new account and a new personal plan are private, and none of a private account's data reaches the partner's local copy; a shared account's transaction that counts in its holder's private plan reaches the partner without its category, note or plan; the partner sees a shared entry's payee, amount, date, payer and shares and nothing of its account; making a shared account private removes it from the partner's devices at their next sync.

### Bank connections and file import

- Against a fake data provider, counting every page as an access, background refreshes never carry PSU headers, never exceed 4 accesses per account in any 24 hours, retries included, and pause 6 hours after a rate limit; pull-to-refresh fetches only the consents of the member who asked.
- A booked transaction that replaces a pending one keeps the member's category and creates no duplicate; a 100 € card hold that books as 43 € is replaced by the 43 € transaction, and a pending transaction the bank drops without a twin counts nowhere.
- Renewal reattaches every account without duplicates, and the 14-day, 3-day and same-day alerts reach only the account holder's devices.
- Connecting a bank and renewing a consent both work on an instance reachable only over Tailscale, from the iPhone and from the web.
- Importing the same OFX file twice, or two overlapping CSV exports, adds each transaction once; two identical payments on the same day in one file both import; a bank reference that the bank reuses for two transactions keeps both; a file overlapping a bank connection adds no duplicate, and its rows that match only by date and amount wait in the inbox as suspected duplicates.
- A CSV with `;` separators, decimal commas and dd/mm/yyyy dates imports through its bank's import profile, and an amount with three decimals is held for review.
- The import preview works offline on the iPhone and the web, and after sync the instance's result replaces it without duplicates.

### Apps, watch, localization and export

- Every MVP screen exists on the iPhone and the web, and the web app reloads offline with its data and pending changes.
- The watch shows both plans' safe to spend, daily budget, days left and pace with the glance's age; with the iPhone out of reach it still shows the cached glance.
- The watch app runs on watchOS 26.0, and opening it brings its complications up to date with the glance it shows.
- No amount appears in Always On or in complications, whatever the watch's settings.
- Every string exists in English and French. English with region FR shows `-1 234,50 €` and `07/11/2026`; with region CH and base currency CHF, an amount shows as `CHF 1'234.50`; an explicit 24-hour or 12-hour choice gives that clock in times and pickers on every device.
- A member's export holds every record they may see in the household, as one versioned archive of NDJSON and CSV files, without provider keys or consent identifiers.

## Open questions

Each question becomes one GitHub issue labelled `needs-info` once these docs are public; its answer lands as an ADR or a `PRODUCT.md` edit, and the pull request that records it deletes the matching (proposed) markers. Answers that need device or household details stay off GitHub. Repository and tooling questions live in [`docs/factory.md`](docs/factory.md#open-questions).

Maintainer decisions still pending:

1. A friend's settlement of shared-plan spending: where does it go when it reaches one member's own account rather than a joint account (Money owed)?
2. Loans (Next), from `PRODUCT.md`'s undecided list: how installments split into principal and interest in the plan.

Outside the project:

3. Enable Banking's written answer (SP-01): members' provider applications run by the owner's instance, how many accounts one application may cover, the automation clause, how the 4-a-day limit is counted, and the certificate's validity and replacement.
