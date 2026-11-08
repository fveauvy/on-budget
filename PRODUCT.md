# Product

<!-- impeccable:product-schema 1 -->

## Platform

adaptive

## Stack

Delegated: the owner asked for the technologies to be chosen during the research phase (`docs/research/`); decisions are recorded as ADRs in `docs/adr/`. Constraints the owner set:

- One UI codebase shared by the iOS, macOS and web apps ("same code everywhere").
- The Apple Watch app is native SwiftUI.
- One design system feeds every app.
- Self-hosted: each instance (server + database) belongs to one owner. The maintainer's instance runs on a dedicated EU VPS.
- Infrastructure: a NixOS module in the project's flake; the reference deployment is a separate private flake, deployed with `just` recipes and reachable only over Tailscale, modeled on dreamsofcode-io/nixos-vps (ADR 0003).
- UI: Expo (React Native) for the iPhone, web and Mac apps; the Mac starts as the web app in the Dock or the iPad app on the Mac (ADR 0004).
- Backend: TypeScript, chosen under the owner's delegation so the server shares domain code with the clients.
- Address: each instance lives at an HTTPS hostname its owner controls, such as a subdomain of a domain they already own (`app.<domain>` in the docs). That hostname is the permanent passkey identifier and doesn't depend on the product name. Every instance works on a private network; the reference deployment is reachable only over Tailscale.

## Users

- **The owner**: self-hosts Shoestring for their household and administers the instance. Wants to plan their budget easily and become accountable for their spending; uses the product every day on iPhone, Apple Watch and Mac. The maintainer runs the first instance.
- **Household members**: people the owner adds, typically a partner. Joint accounts are optional and can be added later.
- **Self-hosters**: anyone who deploys the open-source project for their own household.

There is no public sign-up and no service run for other people.

## Product Purpose

Make a budget easy to plan and hard to drift from, with as little manual work as possible. Bank transactions flow in on their own; recurring income and expenses are anticipated; limits raise alerts before overspending; savings goals (a financial-safety buffer, a big purchase) are funded on purpose. The code is open source so that anyone can self-host their own budget.

Success: the owner opens the app daily, always knows what they can still spend, stays within plan, and sees goals filling up.

## Positioning

An open-source, self-hosted product across iPhone, Apple Watch, Mac and the web that joins four mechanisms in a single loop: zero-based allocation, a live safe-to-spend figure, category limits with alerts, and a short review ritual for new transactions. Built for European banks, French banks first, and for couples who choose how much they share. To be sharpened against `docs/research/budgeting-products.md`.

## Operating Context

- Daily short sessions on iPhone or Apple Watch: review new transactions, check what is left to spend.
- Each month, as income arrives: give every euro a role (fixed charges, discretionary spending, goals).
- Planning and settings on a big screen: the Mac app or the web app on any computer.
- Two people may edit the same shared budget, sometimes offline, each on several devices.
- The owner connects their own banks through their own Enable Banking application; other members import bank files. Every app keeps working offline and syncs on reconnect.
- The owner administers the instance: households (for example a test household and a production household), members, bank connections, backups and upgrades.

## Capabilities and Constraints

- **Apps**: iOS (iPhone) with an Apple Watch app, macOS desktop, and a web app at full parity with the native apps, offline included.
- **Apple Watch**: glance-only in the MVP, with review on the wrist later.
- **Self-hosted, open source**: one instance holds the households its owner creates, each isolated from the others; the owner adds members; no public sign-up.
- **Private network friendly**: every feature works when the instance is reachable only over a private network such as Tailscale; nothing requires inbound connections from the internet.
- **Data protection**: each instance relies on the GDPR household exemption, so the apps have no consent step for data processing and no processing notice; self-hosters who use Shoestring beyond their household handle their own compliance.
- **Test household**: the owner can fill a household with generated fictitious data, without bank connections, to try new features. Upgrades are rehearsed on a throwaway instance restored from a backup.
- **Bank sync**: first supported banks are BoursoBank, Crédit Agricole Aquitaine and Crédit Mutuel du Sud-Ouest (Arkéa group). Read-only: transaction history is all the product needs. The owner connects their own accounts through Enable Banking (ADR 0012); other members, and banks it can't read, use file import. Must be secure and stable.
- **Budget engine**, all four confirmed by the owner:
  - plans run on calendar months;
  - zero-based allocation: every euro gets a role;
  - safe to spend, shown two ways: the amount left for flexible spending until the end of the month, and a daily budget (that amount divided by the days left);
  - category limits with alerts on threshold and on spending pace;
  - review inbox: new transactions to confirm or categorize, as a daily or weekly ritual;
  - end of period: each category has its own rule for unspent money (carry over, move to a goal, or reset).
- **Recurring items** are detected and forecast.
- **Goals**: financial-safety buffer, saving for a big purchase.
- **Manual accounts**, cash first: members record cash spending when they think of it and recount from time to time, the gap becoming spending to categorize. Cash withdrawals are recognized as transfers from the bank account to the cash account.
- **Money owed, both ways**: debts to friends or companies and expected reimbursements from them, each with an amount, an optional due date, a note and a reminder. When a member pays for someone else, the category counts only the member's share; the rest becomes an expected reimbursement, settled when the matching payment arrives. Money owed always belongs to one member; on shared spending, friends owe the member who divides the payment.
- **Loans** with a schedule (remaining principal, installments, interest), just after the MVP; until then, installments are recurring items. Each loan belongs to one member; reports count its interest as spending and its principal as debt reduction.
- **Plans in a household**: each member has a personal plan, and a shared plan covers shared spending and any joint account. Each shared plan sets how its spending splits between members: 50/50, a fixed percentage, or in proportion to income.
- **Configurable sharing**: each person decides which accounts and budgets are private or shared with their household.
- **Offline-first** on every app; sync through the instance's API and database on reconnect; concurrent edits merge predictably.
- **Authentication** with Face ID / Touch ID on top of sign-in.
- **Themes**: a theme system with light and dark modes.
- **Design system**: strict and enforced across every app. Foundations first (typefaces, type scale, margins and spacing, color palette, surface types, elevation, and the like), then a component library, documented, with each component shared by the apps. Prototyping may happen in Figma or Claude Design.
- **Localization**: English and French now; more languages later; no right-to-left scripts.
- **Regional settings** for any European country: date and time formats, 12- or 24-hour clock, time zones, currencies, number formatting, and other locale-sensitive data.
- **Currencies**: one base currency per household for now.
- **Native app distribution**: the maintainer's household installs the native apps through the maintainer's TestFlight; other self-hosters use their instance's web app or build the native apps with their own Apple developer account. No App Store listing and no push relay.

## Brand Commitments

- Name: **Shoestring**, from "on a shoestring": on a tight budget. It replaces the working name on-budget. The product isn't sold, so it needs no registered trademark, domain or store listing (ADR 0020).
- Open source under the MIT license.
- `assets/icon.svg` is a T3 Code placeholder, not a brand asset.

## Evidence on Hand

- No user research, usage data, testimonials or brand assets yet; future work must not fabricate them.
- Research notes in `docs/research/` (in progress).

## Product Principles

1. **Semi-automatic by default**: the product does the bookkeeping (sync, categorization, recurring detection); people make the decisions.
2. **Every euro has a role**: the plan comes first, and safe-to-spend is derived from it.
3. **Accountability is a ritual**: a short daily review of new spending against limits.
4. **Your data, your server**: self-hosted, with no public sign-up. Inside a household, data is private by default and shared by choice, and the apps are biometric-locked.
5. **One product everywhere**: the same design system and components on every app, each working offline and syncing predictably.
