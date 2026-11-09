---
name: Shoestring
description: A self-hosted household budget read as a monthly cycle, health first and amounts second.
colors:
  ground: "#F4F6FA"
  sidebar: "#FFFFFF"
  surface: "#FFFFFF"
  surface-sunken: "#EEF1F6"
  hairline: "#E3E7EE"
  track: "#E7EAF0"
  text: "#121826"
  text-secondary: "#566072"
  text-tertiary: "#687083"
  accent: "#6E5CE6"
  accent-text: "#5B49D6"
  on-accent: "#FFFFFF"
  accent-soft: "rgba(110,92,230,.10)"
  accent-line: "rgba(110,92,230,.45)"
  health-ok: "#22A06B"
  health-warn: "#C27A05"
  health-bad: "#E5484D"
  goal: "#0F9FB4"
  health-ok-text: "#1A7D4C"
  health-warn-text: "#9A6106"
  health-bad-text: "#C93A3A"
  goal-text: "#0B7A8A"
  plan-line: "#8C93A1"
  neutral-fill-1: "#B8BFCB"
  neutral-fill-2: "#7E8798"
  scrim: "rgba(18,24,38,.30)"
  ground-dark: "#0F1115"
  sidebar-dark: "#0C0E12"
  surface-dark: "#171A20"
  surface-sunken-dark: "#1F232B"
  hairline-dark: "#2A2F38"
  track-dark: "#2A2F38"
  text-dark: "#ECEEF2"
  text-secondary-dark: "#A0A6B2"
  text-tertiary-dark: "#8A909C"
  accent-text-dark: "#A797FF"
  accent-soft-dark: "rgba(142,124,248,.16)"
  accent-line-dark: "rgba(167,151,255,.55)"
  health-ok-dark: "#4CB782"
  health-warn-dark: "#E8A33D"
  health-bad-dark: "#EB5B5B"
  goal-dark: "#2BB7CB"
  health-bad-text-dark: "#F07070"
  goal-text-dark: "#4CC6D8"
  plan-line-dark: "#7A808C"
  neutral-fill-1-dark: "#3A404B"
  neutral-fill-2-dark: "#646B78"
  scrim-dark: "rgba(0,0,0,.55)"
  watch-ground: "#000000"
  watch-surface: "#1C1C1F"
  watch-track: "#2A2B30"
  watch-text-secondary: "#A0A4AD"
typography:
  phone-large-title:
    fontFamily: "Inter, -apple-system, system-ui, sans-serif"
    fontSize: "34px"
    fontWeight: 700
    lineHeight: "41px"
    letterSpacing: "-0.02em"
    fontFeature: "\"tnum\", \"cv11\""
  phone-verdict:
    fontFamily: "Inter, -apple-system, system-ui, sans-serif"
    fontSize: "28px"
    fontWeight: 700
    lineHeight: "34px"
    letterSpacing: "-0.02em"
    fontFeature: "\"tnum\", \"cv11\""
  phone-title-2:
    fontFamily: "Inter, -apple-system, system-ui, sans-serif"
    fontSize: "22px"
    fontWeight: 700
    lineHeight: "28px"
    fontFeature: "\"tnum\", \"cv11\""
  phone-headline:
    fontFamily: "Inter, -apple-system, system-ui, sans-serif"
    fontSize: "17px"
    fontWeight: 600
    lineHeight: "22px"
    fontFeature: "\"tnum\", \"cv11\""
  phone-body:
    fontFamily: "Inter, -apple-system, system-ui, sans-serif"
    fontSize: "17px"
    fontWeight: 400
    lineHeight: "22px"
    fontFeature: "\"tnum\", \"cv11\""
  phone-subheadline:
    fontFamily: "Inter, -apple-system, system-ui, sans-serif"
    fontSize: "15px"
    fontWeight: 400
    lineHeight: "20px"
    fontFeature: "\"tnum\", \"cv11\""
  phone-footnote:
    fontFamily: "Inter, -apple-system, system-ui, sans-serif"
    fontSize: "13px"
    fontWeight: 400
    lineHeight: "18px"
    fontFeature: "\"tnum\", \"cv11\""
  phone-caption:
    fontFamily: "Inter, -apple-system, system-ui, sans-serif"
    fontSize: "12px"
    fontWeight: 400
    lineHeight: "16px"
    fontFeature: "\"tnum\", \"cv11\""
  desktop-amount:
    fontFamily: "Inter, -apple-system, system-ui, sans-serif"
    fontSize: "32px"
    fontWeight: 600
    lineHeight: "38px"
    letterSpacing: "-0.02em"
    fontFeature: "\"tnum\", \"cv11\""
  desktop-title:
    fontFamily: "Inter, -apple-system, system-ui, sans-serif"
    fontSize: "20px"
    fontWeight: 600
    lineHeight: "26px"
    fontFeature: "\"tnum\", \"cv11\""
  desktop-ui:
    fontFamily: "Inter, -apple-system, system-ui, sans-serif"
    fontSize: "13px"
    fontWeight: 400
    lineHeight: "18px"
    fontFeature: "\"tnum\", \"cv11\""
  desktop-label:
    fontFamily: "Inter, -apple-system, system-ui, sans-serif"
    fontSize: "12px"
    fontWeight: 600
    lineHeight: "16px"
    fontFeature: "\"tnum\", \"cv11\""
  watch-figure:
    fontFamily: "SF Compact Text, SF Compact, -apple-system, system-ui, sans-serif"
    fontSize: "40px"
    fontWeight: 600
    lineHeight: 1
    letterSpacing: "-0.02em"
  watch-body:
    fontFamily: "SF Compact Text, SF Compact, -apple-system, system-ui, sans-serif"
    fontSize: "13.5px"
    fontWeight: 400
    lineHeight: "18px"
rounded:
  key: "5px"
  chip: "8px"
  control: "10px"
  table: "14px"
  list: "16px"
  card: "20px"
  sheet: "24px"
  full: "999px"
spacing:
  "1": "4px"
  "2": "8px"
  "3": "12px"
  "4": "16px"
  "5": "20px"
  "6": "24px"
  "8": "32px"
components:
  button-primary:
    backgroundColor: "{colors.accent}"
    textColor: "{colors.on-accent}"
    typography: "{typography.desktop-ui}"
    rounded: "{rounded.control}"
    height: "32px"
    padding: "0 14px"
  button-secondary:
    backgroundColor: "{colors.surface}"
    textColor: "{colors.text}"
    rounded: "{rounded.control}"
    height: "32px"
    padding: "0 14px"
  button-quiet:
    backgroundColor: "{colors.surface-sunken}"
    textColor: "{colors.text}"
    rounded: "{rounded.control}"
    height: "32px"
    padding: "0 14px"
  button-primary-large:
    backgroundColor: "{colors.accent}"
    textColor: "{colors.on-accent}"
    typography: "{typography.phone-body}"
    rounded: "{rounded.table}"
    height: "48px"
  card:
    backgroundColor: "{colors.surface}"
    rounded: "{rounded.card}"
    padding: "14px 16px"
  list:
    backgroundColor: "{colors.surface}"
    rounded: "{rounded.list}"
    padding: "0 14px"
  chip:
    backgroundColor: "{colors.surface}"
    textColor: "{colors.text-secondary}"
    rounded: "{rounded.chip}"
    height: "26px"
    padding: "0 9px"
  tag-suggested:
    backgroundColor: "{colors.accent-soft}"
    textColor: "{colors.accent-text}"
    rounded: "{rounded.key}"
    padding: "2px 6px"
  nav-item:
    textColor: "{colors.text-secondary}"
    rounded: "9px"
    height: "32px"
    padding: "0 9px"
  nav-item-selected:
    backgroundColor: "{colors.accent-soft}"
    textColor: "{colors.accent-text}"
  key-cap:
    backgroundColor: "{colors.surface-sunken}"
    textColor: "{colors.text-secondary}"
    rounded: "{rounded.key}"
    padding: "3px 5px"
  sheet:
    backgroundColor: "{colors.surface}"
    rounded: "{rounded.sheet}"
    padding: "8px 20px 0"
  envelope-ring:
    size: "34px"
    rounded: "{rounded.full}"
  envelope-ring-small:
    size: "24px"
    rounded: "{rounded.full}"
---

# Design System: Shoestring

## Overview

**Creative North Star: "The Month as a Cycle"**

Shoestring draws every month as one loop: spending climbs against the plan's dashed line, a today marker cuts the chart, the gap reads in days, and a dotted projection runs to the date the money runs out. Every surface answers the same question in the same order: am I on pace, by how many days, and only then how many euros. The pace verdict (« Un peu vite », « 2 jours trop vite ») is the second type step on the phone, above any amount.

The structure and density come from Linear: a quiet sidebar, hairline-divided rows, a keyboard for every action. The finish bar is Flighty's: one confident chart, clean status, nothing half-drawn. The warmth is Copilot Money's: a cool light ground, white rounded cards with soft shadows, emoji identities and a faint violet glow behind the phone header. These are lineage, not assets: nothing is copied from them. The system refuses both the bank dashboard (a balance and a donut) and the budgeting spreadsheet.

One token source feeds three kits: the web app (React, Base UI and Tailwind 4, wrapped in Electron for the macOS desktop), the Expo iPhone app and the SwiftUI watch app. Token names here are platform-neutral so they generate Tailwind variables, React Native values and Swift constants alike. Light comes first; dark is a full peer except on the watch, which is dark only.

**Key Characteristics:**
- Pace before amounts: the verdict and the gap in days lead; euros follow.
- One violet accent, reserved for action and selection.
- Green, amber and red mean health and nothing else.
- Envelopes are the member's emoji inside a health ring.
- Light first: white cards, 16 to 24px corners, soft shadows. Dark: hairlines, no shadows.
- Tabular figures everywhere a number can change.

## Colors

A cool, nearly colourless ground carries one violet voice and a strict three-step health scale; every other hue is earned by meaning.

### Primary
- **Cycle Violet** (accent): the primary button fill, the selected tab and sidebar item, focus rings, checked radios, the edited plan field and the switch. **Violet Ink** (accent-text, lighter in dark) is the same voice for text links, selected labels and the suggestion label. **Violet Wash** (accent-soft) fills selected rows and nav items; **Violet Rule** (accent-line) outlines suggestion cards and the selected review row.

### Secondary
- **Lagoon Teal** (goal, goal-text): savings goals only: their rings, their share of the allocation bar and their percentage.

### Health scale
- **Margin Green** (health-ok): an envelope or month with room to spare.
- **Brisk Amber** (health-warn): a little fast; drives the verdict dot, the spent line, its area and the projection on the cycle chart.
- **Overrun Red** (health-bad): too fast.
- Each health colour has a darker text twin (`*-text`) for type on light surfaces; fills and strokes use the base value.

### Neutral
- **Cool Ground** (ground): the page behind cards; the phone and desktop canvas.
- **Paper White** (surface, sidebar): cards, lists, sheets, the desktop sidebar.
- **Sunken Mist** (surface-sunken): quiet buttons, key caps, skeleton bars, the offline pill.
- **Hairline** (hairline) and **Track** (track): dividers, card borders, empty ring and bar tracks.
- **Ink** (text), **Slate** (text-secondary), **Pewter** (text-tertiary): three text steps; Pewter also marks the "to review" dashed circle and a stale verdict dot.
- **Plan Grey** (plan-line): the plan's dashed line on the cycle chart.
- **Neutral Fills 1 and 2**: fixed charges and flexible spending in the allocation bar, so only goals carry hue there.
- **Watch Black** (watch-ground) with **Watch Graphite** surfaces and tracks: the watch's own true-black palette; health and accent values come from the dark set.

### Named Rules
**The One Voice Rule.** Violet marks only something you can act on or something selected, plus the faint glow behind the phone header (a 300px gradient from accent at 16%, 24% in dark, to transparent). Never use it for data, health or decoration.

**The Health Means Health Rule.** Green, amber and red appear only for health state: rings, pills, the verdict dot, the cycle chart's spending line and the gaps in days. A stale verdict drops to Pewter until it is recomputed.

**The Neutral Plan Rule.** Allocation shares are neutral greys; teal marks goals; the unassigned remainder is a dashed outline, never a colour.

## Typography

**Phone, web and desktop:** Inter (OFL), with system-ui fallback
**Watch:** SF Compact Text

**Character:** Inter is set tight at large sizes (-0.02em) and always with tabular figures and the alternate one (`tnum`, `cv11`), so amounts and day counts align and never jitter as they update.

### Hierarchy
- **Phone, iOS text styles** (map to Dynamic Type; never fixed): Large Title 700 34/41 for the month or screen name; the verdict at 700 28/34, the second step; Title 2 700 22/28; Headline 600 17/22 for section heads and sheet titles; Body 400 17/22 for row names; Subheadline 400 15/20 for suggestion text and secondary rows; Footnote 400 13/18 for meta lines and row details; Caption 400 12/16 for chart axes. Stat figures sit at 600 20/24.
- **Desktop:** UI 400 13/18 is the base for everything; group labels and column heads 600 12; bar titles 600 16; pane titles 600 20/26; the plan's month heading 700 22/28; the selected transaction's amount 600 32/38.
- **Watch:** the day's figure at SF Compact 600 40; verdict 600 16; envelope rows 13.5; meta 11. Complications show no amounts.

### Named Rules
**The Verdict First Rule.** On any summary surface, the pace verdict and the gap in days are set larger or earlier than any amount.

**The Tabular Rule.** Every figure that can change uses tabular numerals.

## Layout

The phone is a single column: text insets of 20px, card insets of 16px, a 300px glow behind the header, a four-tab bar (Aujourd'hui, Revue, Plan, Comptes) with a translucent blurred surface and the review count as a badge. Rows hold 44pt targets: list rows run 56 to 58px tall. Envelopes group by health ("Trop vite", "Un peu vite", "De la marge") under 13px semibold Slate labels.

The desktop follows Linear: a 240px sidebar (household, ⌘K search, four sections, envelopes and goals with their gaps), a 52px top bar with filter chips, a main list and a right detail pane (340 to 440px). A 38px hint bar at the foot of the main column lists the live shortcuts. Rows are 44 to 46px tall, divided by hairlines inside one rounded table.

Spacing runs on a 4px base (4, 8, 12, 16, 20, 24, 32); card padding sits at 12 to 18px.

## Elevation & Depth

Light mode lifts surfaces with soft, ambient shadows; dark mode is flat, separating surfaces by tone and hairlines alone. Overlays (sheets, toasts) take a deeper pop shadow in both modes.

### Shadow Vocabulary
- **Rest** (`box-shadow: 0 1px 2px rgba(18,24,38,.04), 0 6px 20px rgba(18,24,38,.06)`): cards, lists, tables, pills, secondary buttons, light mode only.
- **Pop** (`box-shadow: 0 18px 40px rgba(18,24,38,.16), 0 2px 6px rgba(18,24,38,.06)`; dark `0 18px 40px rgba(0,0,0,.5)`): bottom sheets and toasts.

### Named Rules
**The Hairline Dark Rule.** In dark mode the rest shadow is none; every card keeps its 1px hairline border.

## Shapes

Corners grow with the size of the surface: key caps and tags 5px, chips 8px, controls 10px, desktop tables, large buttons and toasts 14px, phone lists 16px, cards 20px, sheets 24px (top corners). Pills, badges, rings and status dots are fully round. Every card and list carries a 1px hairline border in both modes. Icons are 1.6px-stroke line drawings with round caps, 16px on desktop and 25px in the tab bar.

## Components

### Buttons
- **Shape:** gently rounded (10px), 32px tall, 600 13px label; large phone buttons are 48px tall, 14px corners, 17px label.
- **Primary:** Cycle Violet fill, white label; carries its key cap on desktop (white at 18%).
- **Secondary:** Paper White with hairline and rest shadow.
- **Quiet:** Sunken Mist, no border, no shadow; the dismissive choice (Ignorer).
- **Focus:** 2px violet outline, 2px offset.

### Chips, pills and tags
- **Chip:** 26px, 8px corners, hairline, Slate label; filters and envelope rules (Report, Remise à zéro).
- **Pill:** 26px fully round, semibold; carries a status dot or the "to assign" dashed circle.
- **Tag:** 11px semibold Violet Ink on Violet Wash; marks a suggested value (« suggestion »).

### Cards / Containers
- **Corner Style:** 20px cards, 16px lists, 14px desktop tables.
- **Background:** Paper White on Cool Ground.
- **Shadow Strategy:** rest shadow in light, none in dark.
- **Border:** 1px hairline; rows inside a list divide with hairlines.
- **Internal Padding:** 12 to 18px.

### Navigation
- **Phone:** four tabs, 25px line icons over 10px labels; selected tab in Violet Ink; the review count rides on the Revue tab as a badge.
- **Desktop sidebar:** 32px items, 9px corners, Slate at rest, Violet Wash with Violet Ink when selected; counts and gaps right-aligned; envelope items show the small ring.
- **Segmented control:** hairline group, 10px corners, selected segment in Violet Wash.

### Envelope ring (signature)
The member's emoji sits inside a ring whose arc is the share spent and whose colour is the envelope's health (teal for goals); the unfilled arc is Track. Sizes: 34px in phone rows, 24px in desktop rows and sidebars, 56px for large displays, 20px on the watch.

### Cycle chart (signature)
A card holding the month: a baseline, the plan as a 1.5px Plan Grey dashed diagonal (4/4), spending as a 2.4px health-coloured line over a 16% wash of the same colour, a today rule, a bracket with the gap in days (« +2 j », 13px bold), a dotted projection (2/4) to an open circle labelled with the run-out date (« à sec le 27 »), and an axis of 1st, today, last day. A three-stat strip follows: today's budget, what is left, the run-out date in health colour. On the watch, the cycle compresses to a bar with a today tick, and the complication to a ring around the gap in days.

### Suggestion card
A card outlined in Violet Rule with a semibold Violet Ink label, a one-sentence explanation of the match, and two equal buttons: quiet Ignorer, primary Approuver (with ↵ on desktop).

### Review row
Each transaction shows a dashed "to review" circle, merchant, envelope ring and name, the effect on that envelope's pace (« +6 j → +9 j » in health colour) and the amount. The selected desktop row takes Violet Wash and an inset Violet Rule.

### Keyboard-first review (desktop)
J and K move; 1 accepts, 2 picks another envelope, 3 splits, T marks a transfer, S skips; ⌘K opens commands. Key caps (11px, 5px corners, Sunken Mist) sit at the right of every action row and in the hint bar. Every action confirms through an inverted undo toast (Ink surface, light text, violet « Annuler » and ⌘Z), bottom-centre.

### Envelope sheet (phone)
A bottom sheet over a scrim, 24px top corners, a grabber, the transaction as headline, then a list of envelopes each with its ring, what remains and the pace effect; the suggested one is pre-selected with Violet Wash and a violet check. A full-width primary accept sits above two secondary actions.

### Recurring-charge tiles
A horizontal row of 136px cards: emoji, name, amount and due day. A paid charge carries a small Ink disc with a check in the top-right corner.

### Plan editor (desktop)
Envelopes grouped under fixed charges, flexible spending and goals, with last month, this plan and the end-of-month rule. The edited amount field takes a violet border and a 3px Violet Wash halo. The side pane shows what is left to assign, a neutral allocation bar and one-key suggestions to reach zero.

### States
- **Empty review:** a green-washed check, « Tout est revu », the next sync time, one violet link.
- **Offline, stale:** a Sunken Mist pill with the data age; the verdict and chart drop to Pewter.
- **Loading:** Sunken Mist skeleton bars and circles in the row rhythm.
- **Hidden amounts:** figures replaced by « •••• € » while the pace stays visible.

## Do's and Don'ts

### Do:
- **Do** lead every summary with the verdict and the gap in days, then the amounts.
- **Do** identify an envelope by the member's emoji inside its health ring.
- **Do** set every changing figure in tabular numerals (`tnum`).
- **Do** use iOS text styles on the phone so type follows Dynamic Type.
- **Do** give every desktop action a key and show it in a key cap.
- **Do** confirm reversible actions with the undo toast instead of a dialog.
- **Do** keep the pace visible when amounts are hidden or data is stale.

### Don't:
- **Don't** use violet for data, health or decoration; it marks action and selection only.
- **Don't** use green, amber or red outside health state.
- **Don't** give envelopes their own hues; health colours the ring, teal marks goals.
- **Don't** add shadows in dark mode.
- **Don't** build a balance-and-donut dashboard or a spreadsheet grid.
- **Don't** show amounts on watch complications.
