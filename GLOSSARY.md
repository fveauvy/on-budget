# Shoestring

A self-hosted budgeting product: a household plans its money each month, reviews its spending and funds its goals across iPhone, Apple Watch, Mac and the web.

## Instance and people

**Instance**:
One self-hosted deployment of Shoestring, its server and database, belonging to one owner.
_Avoid_: server, deployment, tenant

**Owner**:
The person who self-hosts an instance and administers it: its households, members, backups and upgrades.
_Avoid_: admin, superuser, maintainer

**Maintainer**:
The person who publishes Shoestring: they decide the project's proposals, approve its pull requests and sign its TestFlight builds.
_Avoid_: owner (reserved for the instance role), author, admin

**Reference deployment**:
The maintainer's own way of running an instance, a NixOS host reachable only over Tailscale, whose defaults any self-hoster may adopt.
_Avoid_: production, prod, the server

**Ops repository**:
A private repository that declares one deployment's host and holds its encrypted secrets, apart from Shoestring's public repository.
_Avoid_: infra repo, deployment repo, private flake

**Person**:
Someone who signs in to an instance; their passkeys and devices belong to them, and they can be a member of several of its households.
_Avoid_: user, login, account, profile

**Household**:
A group of members whose accounts and plans live together in an instance, isolated from the instance's other households.
_Avoid_: family, foyer, tenant, workspace

**Membership**:
A person's place in one household, which makes them a member of it, such as the one gained by redeeming an invite.
_Avoid_: seat, enrollment, household role

**Member**:
A person in their role in one household, which gives access to that household's data; one person can be a member of several households of an instance, such as the owner in the test and production households.
_Avoid_: user, participant

**Invite**:
A single-use link or QR code, valid 72 hours, that the owner creates to add a person to a household; redeeming it adds a membership, after registering a passkey for someone new.
_Avoid_: invitation code, sign-up link

**Test household**:
A household filled with generated fictitious data and no bank connections, used to try features without touching real money.
_Avoid_: sandbox, staging, demo

## Spaces and sharing

**Space**:
The unit of access and sync inside a household: every record belongs to exactly one space, and a device syncs only the spaces its member may see.
_Avoid_: ledger, workspace, sync group, bucket, scope

**Personal space**:
A member's private space, holding their private accounts and transactions, their personal plan while it is private, their own money owed and their settings.
_Avoid_: private ledger, my space, vault

**Shared space**:
The household's space, synced by every member, holding joint accounts, the accounts and personal plans members chose to share, and the shared plan with its money owed.
_Avoid_: household ledger, joint space, common space

**Visibility**:
Whether an account or a plan is private (in the apps, only its member sees it) or shared (the whole household sees it); whoever runs the instance can read both.
_Avoid_: privacy, permission, access level, sharing mode

## Accounts

**Account**:
A place where money is tracked: a bank account fed by a bank connection or by imports, a card account, or a manual account.
_Avoid_: wallet, pot

**Account holder**:
The member in whose name an account is held, or every member of a joint account; only holders connect, renew or remove its bank connection.
_Avoid_: owner (reserved for the instance owner), titulaire

**Joint account**:
A bank account held by more than one member.
_Avoid_: shared account (sharing is about who sees something, not who holds it)

**Manual account**:
An account with no bank connection, whose transactions members enter themselves; cash is the typical one.
_Avoid_: offline account, wallet

**Card account**:
An account that gathers a deferred-debit card's payments until the bank debits their total from a bank account; that debit is a transfer.
_Avoid_: credit card, deferred card

**Transfer**:
Money moving between two accounts of the same household, such as a cash withdrawal into a cash account; it is never spending or income.
_Avoid_: movement, incoming payment (kept for money from outside the household)

**Cash count**:
A member's statement of how much a manual account really holds; the gap with the recorded balance becomes spending to categorize.
_Avoid_: adjustment, recount

## Transactions

**Transaction**:
Money going in or out of an account on a date: a booked or pending transaction from the bank, a provisional entry or a manual entry.
_Avoid_: operation, entry (kept for provisional, manual and shared entries)

**Booked transaction**:
A transaction the bank has booked, received through a refresh or an import; its amount and dates never change.
_Avoid_: settled, cleared, posted

**Pending transaction**:
A bank transaction not yet booked; it may change or vanish, and its booked twin replaces it.
_Avoid_: hold, authorization (except as the bank's own word)

**Provisional entry**:
A transaction recorded before the bank reports it, from an Apple Pay tap or a bank notification; it counts at once and is matched to its booked transaction later.
_Avoid_: draft, placeholder, shadow transaction

**Manual entry**:
A transaction a member types in, typically on a manual account such as cash.
_Avoid_: manual transaction, cash transaction

**Category line**:
One part of a transaction counted in one category; a transaction divided across categories has several, which sum to its amount.
_Avoid_: split, split line, sub-transaction

**Income**:
Incoming money that adds to a plan's money to allocate, such as a salary.
_Avoid_: revenue, earnings, inflow

**Refund**:
Incoming money in a category that lowers that category's spending instead of adding income, such as a returned purchase.
_Avoid_: credit, reimbursement (kept for money owed)

**Match**:
Linking a provisional or manual entry to the booked transaction it stands for, keeping the member's category; a booked transaction matches at most one entry.
_Avoid_: merge, reconcile, pair

**Payee**:
The cleaned-up, editable name of the other side of a transaction, such as a shop or an employer, derived from the bank label.
_Avoid_: merchant, beneficiary, counterparty (kept for money owed)

**Bank label**:
The raw description a bank sends with a transaction, kept unchanged.
_Avoid_: wording, libellé, memo

**Categorization rule**:
A member's rule that sets the category, payee or attribution of matching transactions when they arrive; a member's own choice always wins over a rule.
_Avoid_: automation, filter, auto-categorizer

**Review inbox**:
The new transactions and suggestions waiting for a member to confirm or categorize them.
_Avoid_: queue, unreviewed list

## Plans

**Plan**:
A budget that money is allocated in, period by period, with its own categories, allocations, limits, rollover rules and goals; a household has one personal plan per member and a shared plan.
_Avoid_: budget (as the name of this object), envelope set, workbook

**Personal plan**:
A member's own plan, funded by their income and kept in their personal space while it is private.
_Avoid_: individual budget, my budget

**Shared plan**:
The household's plan for shared spending and any joint account, kept in the shared space; every member can edit it, and its split decides who funds what.
_Avoid_: joint budget, household budget, common pot

**Period**:
The calendar month a plan runs on, bounded at midnight in the household time zone.
_Avoid_: cycle, pay period

**Category**:
A line of one plan that money is allocated to and that transactions count against; every category belongs to exactly one plan.
_Avoid_: envelope, bucket

**Fixed category**:
A category for charges known in advance, such as rent, subscriptions or a loan's installments; its money is held for its recurring items.
_Avoid_: fixed costs, essentials, bills

**Flexible category**:
A category for spending that varies from day to day, such as groceries or outings; safe to spend comes from these categories.
_Avoid_: discretionary, variable spending

**Money to allocate**:
What a plan can give roles to in a period: what the previous period left it, plus the income of the period.
_Avoid_: to be budgeted, ready to assign, available

**Allocation**:
Money given a role in a category for a period. Under zero-based budgeting, all of a plan's money to allocate is allocated.
_Avoid_: assignment, budgeted amount

**Reallocation**:
A change of allocation that takes an amount from one category and gives it to another in the same plan and period; it is recorded as an operation, so concurrent reallocations add up.
_Avoid_: money move, transfer (reserved for accounts), budget transfer

**Unallocated**:
The part of a plan's money to allocate that has no allocation yet; zero-based planning brings it to zero, and below zero the plan is over-allocated.
_Avoid_: to assign, to be budgeted, ready to assign, available

**Remaining**:
What a category still holds in a period: its allocation plus the money carried into it, minus its spending; it is negative when the category is overspent.
_Avoid_: available, balance, left

**Overspending**:
The amount by which a category's spending exceeds its allocation plus the money carried into it.
_Avoid_: deficit, overdraft (kept for accounts)

**Shortfall**:
The part of a fixed category's upcoming occurrences in a period that its remaining doesn't cover.
_Avoid_: gap, deficit

**Rollover rule**:
A category's rule for its unspent money at the end of a period: carry it over, move it to a goal, or reset it.
_Avoid_: carry-forward

**Safe to spend**:
The money left for flexible spending in a plan's current period; a member sees their personal plan's figure and the shared plan's.
_Avoid_: left to spend, available

**Daily budget**:
Safe to spend divided by the days left in the period.
_Avoid_: daily allowance

**Pace**:
How fast spending runs against an even spread, over the period, of what a plan's flexible categories, or a limit's category, hold: their allocations plus the money carried in.
_Avoid_: burn rate, velocity, rhythm

**Limit**:
A spending cap on a category that raises alerts on a threshold and on spending pace.
_Avoid_: cap, ceiling

**Alert**:
A notification the instance sends about money or a bank connection, such as a limit's threshold or pace crossed, a recurring item that changed, or a consent near expiry.
_Avoid_: warning, nudge, reminder (kept for money owed)

**Goal**:
Money a plan sets aside on purpose toward a target amount, such as a safety buffer or a big purchase.
_Avoid_: objective, savings target, pot

**Recurring item**:
An income or expense expected to repeat, detected from transactions or declared.
_Avoid_: subscription, bill

**Occurrence**:
One expected instance of a recurring item on a date: upcoming, paid, paid with a different amount, or missed.
_Avoid_: instance, event, installment (reserved for loans)

**Forecast**:
The expected occurrences of recurring items for the rest of the period and beyond, which feed safe to spend and projected balances.
_Avoid_: projection, prediction

## Shared money

**Split**:
A shared plan's rule for dividing its spending between members: 50/50, a fixed percentage, or in proportion to income.
_Avoid_: ratio, quota, share rule, distribution key

**Share**:
One member's or counterparty's portion of an amount under a split or a net-cost division. Not to be confused with sharing, which is about visibility.
_Avoid_: portion, part, quote-part

**Contribution**:
Money a member gives a shared plan for a period, as its split asks or to advance a counterparty's share, usually by a transfer into a joint account.
_Avoid_: deposit, top-up, funding transfer

**Shared entry**:
The record, in the shared space, of a payment made from a private account and attributed to the shared plan; it shows the payee, amount, date, payer and shares, never the account.
_Avoid_: shared expense, mirror, shadow transaction

**Member balance**:
What one member owes another inside a household when shared spending was paid unevenly against the split.
_Avoid_: debt (kept for counterparties), IOU, tab

**Attribution**:
Which plan a transaction counts in and, for shared spending, how it divides into shares; it defaults from the account and is confirmed in the review inbox.
_Avoid_: owner (for transactions), assignment, tagging

**Net cost**:
The member's own share of a payment they made for others; the category counts only the net cost, and the rest becomes an expected reimbursement.
_Avoid_: real cost, own part

## Money owed

**Counterparty**:
The person or company on the other side of a debt, a loan or an expected reimbursement.
_Avoid_: contact, creditor, debtor

**Debt**:
Money a member owes a counterparty and must pay back.
_Avoid_: IOU, liability

**Loan**:
A debt repaid in scheduled installments with interest, such as a consumer loan or a mortgage.
_Avoid_: credit, financing

**Installment**:
One scheduled payment of a loan, made of principal and interest.
_Avoid_: monthly payment, mensualité

**Expected reimbursement**:
Money a counterparty owes a member, such as a friend's share of a bill or a refund a company owes.
_Avoid_: receivable, IOU, expected refund

**Incoming payment**:
Money that reaches a household account from outside the household, such as a counterparty's settlement.
_Avoid_: transfer (kept for money between household accounts), deposit

**Reminder**:
A notification a member sets on a debt or an expected reimbursement, for a date and time they choose.
_Avoid_: alert, nudge

**Settlement**:
The payment that clears all or part of a debt, an expected reimbursement or a member balance.
_Avoid_: repayment, payback

## Bank data

**Bank connection**:
A member's link between one supported bank and Shoestring through their provider application: one consent, the accounts it covers, and its health (ok, expiring, needs renewal, bank down, rate-limited).
_Avoid_: link, integration, item, bank sync

**Data provider**:
The licensed account-information company that reads bank data on a member's behalf, such as Enable Banking.
_Avoid_: aggregator, AISP (outside legal text), bank API

**Provider application**:
A person's own application registered with the data provider, whose key the instance generates and keeps; one per person, never one per instance.
_Avoid_: provider app, API key, credentials, client

**Consent**:
A member's authorization, given at their bank with strong customer authentication, that lets the data provider read the covered accounts until it expires, usually after 180 days.
_Avoid_: session, mandate, authorization (in UI copy)

**Renewal**:
Giving a new consent for an existing bank connection, before or after the old one ends; its accounts reattach automatically.
_Avoid_: re-link, reconnect, re-auth

**Refresh**:
One fetch of an account's balances or transactions through a bank connection; background refreshes are capped by the bank's daily limit, refreshes a member asks for are not.
_Avoid_: sync, bank sync, poll, update

**Refresh slot**:
One of the times of day at which the instance refreshes a bank connection in the background.
_Avoid_: slot (alone), poll, cron

**Connector**:
The server code that turns one data source (a data provider, OFX, QIF or CSV) into transactions behind one interface; connectors are built in, never plugged in at runtime.
_Avoid_: plugin, adapter, integration, provider

**Import**:
Bringing transactions from a bank file (OFX, QIF or CSV) into an account; transactions already present are skipped.
_Avoid_: upload, load, sync

**Import profile**:
A saved description of one bank's CSV layout: separator, decimal mark, date format and columns.
_Avoid_: template, mapping, preset

## Money, time and formats

**Base currency**:
The one currency a household's plans count in.
_Avoid_: home currency, main currency, default currency

**Household time zone**:
The IANA time zone that defines today, periods, pacing and due dates for a household.
_Avoid_: server time, local time

**Regional preferences**:
A person's stored region, hour cycle and first weekday, each automatic or explicit, plus the web app's language; on Apple devices the app language follows each device's own setting.
_Avoid_: locale (for the stored setting), regional profile

## Apps, devices and security

**Device**:
One app installation, or one browser profile, that a person signed in on; it keeps one local copy per household, holds its own key or session, and can be revoked.
_Avoid_: client, installation, session

**Local copy**:
A device's database of one household's spaces, with the changes it hasn't synced yet; losing it costs a fresh download, never a synced change.
_Avoid_: cache, replica, client

**Sync**:
The exchange between a device and the instance: the device pushes its pending changes and pulls the changes of its spaces.
_Avoid_: refresh, replication, backup

**Glance**:
The small read model the instance builds for the watch and widgets: safe to spend, the daily budget and pace of the personal and shared plans; the iPhone relays it to the watch.
_Avoid_: snapshot, summary, widget data

**Instance address**:
The HTTPS hostname of an instance (`app.<domain>`), which is also its passkey identifier, so it is chosen once and never changes.
_Avoid_: server URL, public URL, domain, RP ID (outside authentication code)

**App lock**:
The Face ID, Touch ID or passkey check an app asks for when it opens or returns from the background; it is not a sign-in.
_Avoid_: biometric lock, biometric login, PIN

**Step-up**:
A fresh biometric or passkey check before a sensitive action: bank connections, members, sharing, export.
_Avoid_: re-auth, sudo mode

**Hide amounts**:
A display mode that hides figures from whoever looks at the screen; it doesn't change visibility.
_Avoid_: privacy mode, discreet mode

**Export**:
A versioned archive of everything one member may see in one household, without provider secrets; it serves portability and is not a backup.
_Avoid_: backup, dump, household export
