---
status: accepted
---

# Design tokens and component specs generate every platform

The design system's source of truth is the repository: DTCG token files in `design/tokens/`, in reference, system and component tiers, behind one resolver of appearance by platform, and one `design/components/<Name>/spec.md` per component, whose YAML front matter lists its variants. The resolver starts with the five permutations that have a consumer, light and dark on iOS and the web and dark on watchOS, and adds increased-contrast variants and macOS once an app uses them. Terrazzo, with small in-repo plugins, generates the React Native themes, the web's CSS variables, the Swift asset catalog and constants for the watch and widgets, an agent-facing `DESIGN.md`, and variant enums from each spec's front matter, in TypeScript for every component and in Swift for those the watch or widgets use, so an implementation that misses a variant fails to compile. Generated files are committed, and CI fails on drift, on any WCAG 2.2 AA contrast failure in a permutation, and on raw colors or sizes in component code. Why: components exist in React Native and, for the watch and widgets, again in SwiftUI, and only a generated contract keeps the two from drifting; Figma can't be the source without its Enterprise plan or a tokens plugin, and agents edit files.

## Consequences

- Figma and Claude Design work downstream: their prototypes are ported into components, and Figma's Code Connect maps only the React Native ones.
- The project owns a handful of generator plugins and a permutation check; Style Dictionary is the fallback generator.
- Apple builds use system fonts and SF Symbols, and the web uses system fonts and Lucide, since SF Symbols are licensed for Apple platforms only; an icon map links the two. SP-27 may replace system fonts with one bundled face.
