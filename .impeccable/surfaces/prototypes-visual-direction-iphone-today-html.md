---
version: 1
slug: "prototypes-visual-direction-iphone-today-html"
primary_target: "prototypes/visual-direction/iphone-today.html"
related_targets: ["prototypes/visual-direction/iphone-review.html","prototypes/visual-direction/desktop-review.html","prototypes/visual-direction/desktop-plan.html","prototypes/visual-direction/watch.html","prototypes/visual-direction/states.html"]
---

# Visual direction prototypes

Scope: throwaway, code-led HTML prototypes that settle Shoestring's visual direction before any app code exists. Visitor mode: Operate on every screen. Screens: iPhone Today and Review, desktop Review and Plan (the monthly allocation), the watch glance and a complication; light and dark (the watch is dark only).

Audience and job: the owner, and later a partner, check their pace in short daily sessions on the iPhone and the watch, review new transactions, and give every euro a role on a big screen when income arrives. Proof: synthetic household data (labelled as prototypes), with the pace verdict read before any amount. Constraints: iOS conventions on the phone (four tabs, sheets, 44 pt targets), standard web components on desktop, French copy, tabular figures, AA contrast; nothing may feel like a bank app or a spreadsheet.

Chosen direction: B « Cycle », pinned by the maintainer in round two. Linear sets the structure, density and keyboard-first review; Flighty sets the finish bar. Memorable moment: the month drawn as a cycle, spending against the plan's line, with the gap in days.

Unresolved: the month journal page (opens from the cycle chart, after the MVP); hue rules for envelopes beyond health colours; motion beyond state changes.

## Direction contract

THESIS: The month is a cycle. Spending is read as progress against the plan's line, health first, amounts second; it refuses the bank dashboard (balance plus donut) and the budgeting spreadsheet.

OWN-WORLD: Linear's structure, warmed with Copilot Money's cues. Light first: a cool ground (#F4F6FA), white cards with 16–24px corners and soft shadows; dark keeps hairlines and no shadows. One violet accent (#6E5CE6) for action and selection, plus a faint violet glow behind the phone header. Health colours (green, amber, red) only for state: rings, pills and gaps. Each envelope is the member's emoji inside a ring whose colour is its health; goals use teal. Inter with iOS text styles on the phone and 13px UI on desktop; SF Compact on the watch.

STORY: The member reads the verdict (« Un peu vite », « 2 jours trop vite ») and the cycle chart, then today's budget and the envelopes that drive the pace. Suggestion cards settle matches (a friend's repayment, a transfer) with Ignorer or Approuver. Review runs by keyboard or sheet, each transaction showing its effect on the pace. Recurring charges sit in tiles, ticked once paid. On payday, envelopes fill until nothing is left to assign.

FIRST VIEWPORT: iPhone Today. A large title « Novembre » with the date and sync age, then the verdict as the second type step. The cycle chart (dashed plan line, spent area, today marker, a +2 j bracket, a dotted projection to « à sec le 27 »). A three-stat strip (today's budget, what is left, the run-out date). Health groups of envelopes with progress circles. The primary action is the Review tab and its badge.

FORM: maintainer-pinned after round two: B « Cycle », a Linear-led canon at Flighty's finish level, outside the ordered list, then warmed with Copilot Money's cues at the maintainer's request. Round-one roll superseded: seed:a6a104e8efc987e3.

FINISH: unreviewed and undocumented is unfinished; this build ends with the finish review, the verdict, DESIGN.md, and every shipping raster carrying its provenance
