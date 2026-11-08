# Software factory for the chosen stack

> **Status (2026-11-08).** Background research, adopted by [`docs/factory.md`](../factory.md), which trims its tool set and drops its channels; `factory.md` wins where they differ. It uses on-budget, the working name, for what is now Shoestring ([ADR 0020](../adr/0020-shoestring-as-the-product-name.md)).

> Research note, 2026-11-07, revised the same day after a fact-check. Versions are the latest stable releases that day, read from the npm registry, GitHub releases and an evaluation of `nixpkgs-unstable`, unless marked. Confidence: **H** high, **M** medium, **L** low. "Secondary" marks a non-primary source, "judgment" my own assessment, "measured" a check run for this note. It replaces the stack-dependent parts of [factory-ci-quality.md](factory-ci-quality.md), which assumed SwiftUI apps, and builds on [cross-platform-ui-stack.md](cross-platform-ui-stack.md) (strategy B), [design-system.md](design-system.md), [i18n-l10n.md](i18n-l10n.md), [sync-engine.md](sync-engine.md), [web-client-sync.md](web-client-sync.md), [watchos-app.md](watchos-app.md), [native-apps-auth.md](native-apps-auth.md), [connectors-and-license.md](connectors-and-license.md), [self-hosting.md](self-hosting.md) and [nixos-vps-infra.md](nixos-vps-infra.md). It edits none of them; §1 lists where it departs from them.

## Question

What CI/CD, quality guard rails and agentic-development setup should on-budget use for its chosen stack: Expo (React Native) for the iPhone, web and Mac apps; a SwiftUI watch app and Swift extensions; a TypeScript backend on PostgreSQL; a Nix flake with a NixOS module and an OCI image? The repository is public on GitHub Free, one maintainer works with AI agents (Claude Code and Codex), and no Mac is always on, so there is no self-hosted runner.

## TL;DR

1. **One toolchain definition, one entry point.** The flake's devShell pins every CLI tool (Node 26.13 LTS, pnpm 12, PostgreSQL 18.6, squawk, gitleaks, lefthook, nixfmt, statix, deadnix, SwiftFormat, SwiftLint, Maestro, just, deploy-rs, skopeo). `pnpm-lock.yaml` pins the JavaScript tools, and `.xcode-version` pins Xcode's major.minor. Hooks, agents and CI call only `just` recipes. Drop mise: a second pin file for the same tools only adds drift (judgment). **M-H**
2. **pnpm 12 workspaces, no Turborepo or Nx yet, TypeScript 7 project references.** Expo SDK 58 supports pnpm's isolated installs, configures Metro for monorepos by itself, and needs Node 22.13+, 24.3+ or 26+ and Xcode 26.4+. It doesn't support macOS yet, so the Mac stays the web app or the iPad app for at least one more SDK. The committed `app/ios/` holds the app, watch, widget, NSE and App Intents targets. CocoaPods is SDK 58's default, but React Native 0.89 targets SwiftPM and plans to drop CocoaPods, so the Podfile is temporary. TypeScript 7 has no API yet, so the `typescript` name installs TypeScript 6 for the tools that import it. Keep `pnpm-lock.yaml` a single YAML document (`pmOnFail: ignore`, no `configDependencies`), or Dependabot creates no alerts; pnpm then enforces no version of itself, so every install outside Nix pins the devshell's. **M**
3. **Biome 2.6 formats and lints everything it can, design-system bans included.** Biome's and ESLint's color-literal rules both miss Unistyles' `StyleSheet.create(theme => …)`: ESLint's reads only an object argument, and Biome's skips a `StyleSheet` imported from `react-native-unistyles`. So GritQL plugins carry the bans. Measured: a short one flags hex and functional colors and numeric spacing in that form, nested variants included, but it misses named colors, template literals and many size props, so it must deny by default before the design system counts as strict (F9). Two more plugins flag `style` props outside the design system and literal JSX text. If the React Compiler is on, Biome's nursery `useReactCompiler` reports its diagnostics as warnings; `eslint-config-expo` returns only if F9 finds gaps, and even without type-aware rules it needs TypeScript 6 under the `typescript` name. No Prettier. Swift: SwiftFormat and SwiftLint. Nix: nixfmt, statix and deadnix through treefmt. SQL: squawk. Commits: Conventional PR titles. **M-H**
4. **Tests by layer.**
   - Vitest 5.1 for pure TypeScript (domain, sync core, server, generators), with fast-check 4.10's model-based runner and scheduler for the sync simulator.
   - Jest 30 with jest-expo 58 and React Native Testing Library 14 for components, because Expo documents only Jest.
   - Server integration tests on a real PostgreSQL 18 started from the devshell, the same way locally and in CI.
   - Maestro 2.12 for iOS end-to-end: Detox officially covers React Native 0.77 to 0.84 only.
   - Playwright 1.65 for web end-to-end, screenshots and axe-core 4.14; Storybook 10.6 tests through its Vitest addon (its test runner is superseded).
   - NixOS VM tests on x64 runners, which have KVM; the watch through `xcodebuild test`.

   **M**
5. **CI is free and unlimited on this public repo, macOS included,** within 20 concurrent jobs, 5 of them macOS. The macOS job uses the `xcode-27` label, generally available since 2026-10-15, because the watch's watchOS 27 floor needs Xcode 27; Expo SDK 58 alone would accept `macos-26`. It runs on PRs only when native code or native dependencies change, and always on `main`. One `gate` job is the only required check. Caches: Magic Nix Cache (free, on the Actions cache) and the pnpm store, saved by `main` pushes within the 10 GB limit; release jobs set `cache-mode: none`. **H** on the plan facts, **M** on the layout.
6. **Security rails.** CodeQL default setup for JavaScript/TypeScript and Actions workflows. Swift has no CodeQL path: it needs a traced build, which CodeQL 2.27.3 dropped on macOS 27 and with any Xcode 27, so Swift SAST is an owner decision. Dependabot for npm, Swift, Nix and Actions, with 7-day cooldowns, the minimum zizmor accepts. OpenSSF Scorecard (advisory). Secret scanning, push protection and gitleaks. `actions.lock` in place of SHA pins in workflow files (reusable-workflow calls keep SHAs), zizmor and actionlint. No `pull_request_target`, and no repository secrets: only environment secrets bound to `main` or `v*` tags. **H**
7. **Agents act through a machine account, not as the owner.** GitHub's terms allow one free machine account. With write access only, it can't edit rulesets, but it can approve and merge PRs it neither wrote nor last pushed (Dependabot's, contributors', release-please's). So the `main` ruleset requires code-owner review, and `CODEOWNERS` names only the owner. Fine-grained tokens can't reach a personal account's repo as a collaborator, so it uses a classic token whose only repo is this one. **M-H**
8. **Release: release-please, then one tag `vX.Y.Z` for everything.**
   - Merging the release PR makes release-please create the tag and a draft GitHub Release (`draft`, `force-tag-creation`). The tag workflow builds the Nix OCI image natively on amd64 and arm64, pushes both by digest to GHCR, joins and attests them, attaches the compose bundle and digests, and publishes the draft. The image and bundle start once the server is stable (ADR 0018).
   - Xcode Cloud archives the iPhone app with its watch app and extensions and sends it to the internal TestFlight group, so no signing secret enters GitHub. The fallback is `xcodebuild` with an App Store Connect team key in a tag-bound environment.
   - EAS (free tier: 15 iOS builds, low priority, 45-minute limit) and fastlane aren't needed.

   **M**
9. **Deploys stay in the private ops repo:** weekly flake-input updates gated by VM tests, `just deploy` by hand first, then deploy-rs 1.0 over Tailscale from CI (OIDC, `tag:ci`). On a private repo that needs GitHub Pro for environment secrets. **M**
10. **Agentic factory.**
    - Short root and nested AGENTS.md files: Codex reads at most 32 KiB.
    - Hooks load the devshell (Claude Code's `SessionStart` writes `CLAUDE_ENV_FILE`), format on edit, and run fast checks on stop. Codex skips a new or changed repo hook until the owner trusts it again.
    - Ten repo skills live in `.agents/skills` (read by Codex), linked from `.claude/skills` (read by Claude Code).
    - MobileBuildMCP, Xcode's `mcpbridge`, agent-device and Playwright MCP drive simulators and browsers.
    - The installed skills chain wayfinder → to-spec → to-tickets → triage → implement (tdd, code-review) → ship, and §7's definition of done closes each ticket.

    **M**
11. **Cost: $0 a month** beyond the Apple Developer Program ($99 a year). CI on the public repo is free; the private ops repo's weekly VM tests fit in the Free plan's $12 a month of included compute, about 2,000 Linux minutes. Xcode Cloud's 25 included hours cover releases. Optional: GitHub Pro ($4 a month) for CI deploys from the private ops repo. **H** on prices, estimates on usage.

## 1. What carries over from factory-ci-quality.md

| Topic | Verdict | Now |
|---|---|---|
| GitHub Free, public repo, hosted runners, no self-hosted runner | Keep | Rechecked in §5 |
| macOS label | Change | `xcode-27` is required, not optional, and generally available since 2026-10-15 (§5). The watch floor is watchOS 27.0 ([watchos-app.md](watchos-app.md) §3.2), and only that image has the watchOS 27 SDK and simulator ([image readme](https://github.com/actions/runner-images/blob/main/images/macos/xcode-27-arm64-Readme.md), image 20261026). **H** |
| Xcode Cloud for TestFlight archives; signing facts; 90-day builds; monotonic build number | Keep | The post-clone script now installs Node 26 and the devshell's exact pnpm, then runs `pnpm install` and `pod install` (spike F1) |
| SwiftFormat, SwiftLint, Swift 6 mode, warnings as errors | Keep, smaller scope | Watch app, extensions and the two Swift Expo modules only |
| swift-snapshot-testing | iOS widgets only | Watch views: `ImageRenderer` diffs ([watchos-app.md](watchos-app.md) §3.5) |
| `mise run …` entry point (also in design-system.md and i18n-l10n.md) | Replace | Nix devshell plus `just` (§2), same task names: `just check`, `just ds-build`, `just l10n` |
| git-cliff release notes | Replace | release-please (§6) |
| Kamal and Docker deploy | Superseded | [nixos-vps-infra.md](nixos-vps-infra.md) |
| Rulesets, single `gate`, squash, linear history | Keep | Plus owner approval through a machine account (§5) |
| CodeQL "once the code has an OSI license" | Now possible | MIT is decided; Swift has no CodeQL build mode on Xcode 27 (§5) |
| Dependabot over Renovate | Keep | Plus `nix`, cooldowns, two caveats (§5) |
| Agent hooks, `stop_hook_active` guard, deny rules | Keep | Plus a devshell hook and repo skills (§7) |
| JSON `.xcproj` | Still no | CocoaPods' Xcodeproj gem can't read it yet, and Expo SDK 58 builds with CocoaPods; revisit once an SDK builds with SwiftPM (§2) |
| Jest for logic ([cross-platform-ui-stack.md](cross-platform-ui-stack.md) §4.6) | Narrow | Vitest for pure TypeScript; Jest only where React Native renders |
| ESLint `no-restricted-syntax` token rules ([design-system.md](design-system.md) §6) | Replace | Biome GritQL plugins, measured, then widened (§3) |

## 2. Monorepo and tooling

**Package manager** (versions of 2026-11-07):

| | pnpm 12 (pick; 12.18 on npm, 12.9 in nixpkgs) | Bun 1.4 | npm workspaces |
|---|---|---|---|
| Expo | Supported, isolated installs since SDK 54 ([monorepos](https://docs.expo.dev/guides/monorepos/)) | Supported (same page) | Supported (same page) |
| Phantom dependencies | Blocked by isolated `node_modules` | Not checked | Hoisted by default ([monorepos](https://docs.expo.dev/guides/monorepos/)) |
| Supply-chain defaults | `minimumReleaseAge` 1,440 min and `blockExoticSubdeps` on by default; `trustPolicy: no-downgrade` opt-in; build scripts blocked unless listed in `allowBuilds` ([resolution](https://pnpm.io/settings/dependency-resolution), [build](https://pnpm.io/settings/build)) | Not checked | Not checked |
| Dependabot | pnpm 7 to 12, except two-document lockfiles ([ecosystems](https://docs.github.com/en/code-security/dependabot/ecosystems-supported-by-dependabot/supported-ecosystems-and-repositories)) | Version updates only, no security updates | Full |
| Nix packaging | `fetchPnpmDeps` with a hash that changes with every lockfile change; nixpkgs-unstable has no hash-free pnpm importer (measured) | Not checked | `importNpmLock`, no hash ([self-hosting.md](self-hosting.md) §2.3) |

- **The two-document trap.** pnpm 12 writes a leading "boot" document into `pnpm-lock.yaml` when the project pins pnpm through `devEngines.packageManager` or `packageManager`, and always when it has `configDependencies` ([lockfile](https://pnpm.io/lockfile)). Then "Dependabot alerts and Dependabot security updates are not created for those dependencies" ([ecosystems](https://docs.github.com/en/code-security/dependabot/ecosystems-supported-by-dependabot/supported-ecosystems-and-repositories)). Setting `pmOnFail: ignore` stops the version record ([lockfile](https://pnpm.io/lockfile), [CLI settings](https://pnpm.io/settings/cli)), and the project adds no `configDependencies`. **H**
- **Then nothing pins pnpm outside Nix.** With `ignore`, pnpm will "skip the check entirely" ([CLI settings](https://pnpm.io/settings/cli)). The devshell pins it, the Xcode Cloud script installs that exact version, and a CI check compares the two (judgment). Dependabot runs a pnpm of its own choice; its docs don't say which. **M**

```yaml
# pnpm-workspace.yaml (sketch)
packages: [app, server, "packages/*", "tools/*"]
catalog: { react: 19.3.0, react-native: 0.88.1 }  # one React Native, as Expo requires
pmOnFail: ignore              # single-document lockfile; Nix pins pnpm; no configDependencies
trustPolicy: no-downgrade
minimumReleaseAge: 1440       # the default, restated; Dependabot's 7-day cooldown sits above it (§5)
allowBuilds: { esbuild: true } # a map; anything unlisted stays blocked, and strictDepBuilds fails the install
```

```jsonc
// root package.json (sketch): the TypeScript team's side-by-side layout
"devDependencies": {
  "typescript": "npm:@typescript/typescript6@^6.0.2", // for tools that import it; its binary is tsc6
  "@typescript/native": "npm:typescript@^7.0.2"       // tsc, for tsc -b
}
```

- **Orchestrator: neither Turborepo 2.11 nor Nx 24, for now** (judgment). The JavaScript code is small; `pnpm -r --filter "...[origin/main]"` runs affected packages, `tsc -b` is incremental, and Vitest selects related tests. Turborepo earns its place when the Linux JavaScript job passes about 10 minutes; its remote cache is free on Vercel's plans ([Vercel](https://vercel.com/changelog/free-vercel-remote-cache)), a third party. **M**
- **TypeScript 7.0.2** (2026-07-08) is the Go port. `tsc --build` builds referenced projects in parallel (`--builders`); `strict` is on by default and `types` defaults to `[]`. It ships no programmatic API ("We expect TypeScript 7.1 to ship with a new (and different) API"), and "some tools like typescript-eslint expect to import from `typescript` directly", so the TypeScript team aliases `typescript` to TypeScript 6 and installs TypeScript 7 as `@typescript/native` ([announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)). Dropping type-aware rules doesn't avoid this: `eslint-config-expo` 58.0.4 depends on `@typescript-eslint/parser`, whose 8.72.1 release peer-requires `typescript` `>=4.8.4 <6.1.0` ([eslint-config-expo](https://registry.npmjs.org/eslint-config-expo/58.0.4), [parser](https://registry.npmjs.org/@typescript-eslint/parser/latest), [versions](https://typescript-eslint.io/users/dependency-versions/)). So: the alias layout above, TypeScript 7 for `tsc -b`, no type-aware ESLint, and no `typescript: 7.0.2` anywhere. The aliases sit in the root `package.json`, since pnpm's catalog docs don't say whether an entry may be an `npm:` alias (unverified). **H**
- **Project references** (judgment). A root solution `tsconfig.json` references every package. Packages are `composite` and export their TypeScript source through `exports`, so Metro, Vite, Vitest and the server bundler read source and `tsc -b` only typechecks. The Expo base config (`module: preserve`, `moduleResolution: bundler`, `customConditions: ["react-native"]`; [expo@58 tsconfig.base.json](https://unpkg.com/expo@58.0.23/tsconfig.base.json)) uses nothing TypeScript 7 removed. The shared base adds `@tsconfig/strictest` 2.0.8's flags: `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride`, `noPropertyAccessFromIndexSignature`, `noImplicitReturns` and the unused checks ([strictest](https://unpkg.com/@tsconfig/strictest@2.0.8/tsconfig.json)), plus `verbatimModuleSyntax`. **M**
- **Node 26 LTS.** Node 26.13.0 "Lithium" became LTS on 2026-10-28 and is supported to 2029-04-30; Node 24 entered maintenance on 2026-10-20 ([schedule](https://github.com/nodejs/Release/blob/main/schedule.json), [releases](https://nodejs.org/dist/index.json)). Expo SDK 58 accepts 22.13+, 24.3+ or 26+ and no odd releases ([SDK 58](https://expo.dev/changelog/sdk-58)). nixpkgs has `nodejs_26` 26.13.0 in both nixos-26.05 and unstable (measured). Use 26 everywhere: devshell, CI, the server package and the Xcode Cloud script. **H**

**Who pins what** (judgment, versions measured in nixpkgs-unstable on 2026-11-07):

| Pin source | Holds | Why |
|---|---|---|
| `flake.lock` → `devShells.default` (nixpkgs-unstable input for tools) | nodejs_26 26.13.0, pnpm 12.9, postgresql_18 18.6, squawk 2.70, gitleaks 8.30.1, lefthook 2.2.2, nixfmt 1.5.0, statix (fork, 2026-07-17), deadnix 1.3.2, swiftformat 0.64.3, swiftlint 0.65.1, maestro 2.12.0, xcbeautify 3.2.1, actionlint 1.7.12, zizmor 1.32.0, just 1.59, deploy-rs 1.0.0, skopeo 1.24.1 | The infra already needs Nix; CI and laptops get the same binaries; SwiftLint, SwiftFormat and Maestro build for aarch64-darwin and Linux (measured) |
| `flake.lock` → server package (stable channel) | The NixOS release's Node 26 and PostgreSQL; `pnpm_12` (12.9.0), named in `fetchPnpmDeps` and the build | Matches what self-hosters run. nixos-26.05 lags on tools: its `pnpm` is `pnpm_11` 11.27 ([all-packages.nix](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/pkgs/top-level/all-packages.nix)), and the variants "support different sets of lock file versions" ([nixpkgs docs](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/doc/languages-frameworks/javascript.section.md)); Biome is 2.4.15 (measured) |
| `pnpm-lock.yaml` | Biome, TypeScript, Vitest, Jest, Playwright, Storybook, Terrazzo, FormatJS, knip, commitlint | npm moves faster than nixpkgs (pnpm 12.18 vs 12.9, Biome 2.6.6 vs 2.6.5) |
| `.xcode-version` + `xcode-select` | Xcode major.minor (`27.1`), never a patch, beta or RC | Nix can't provide Xcode; the image has it and replaces patch versions in place (§5) |
| `.node-version`, `.pnpm-version` | Mirror the devshell's Node and pnpm for editors and the Xcode Cloud script | A CI check compares both with the devshell, since `pmOnFail: ignore` turns off pnpm's own check |

**Expo SDK 58 in this monorepo** (facts from [SDK 58](https://expo.dev/changelog/sdk-58), [versions](https://docs.expo.dev/versions/latest/), [monorepos](https://docs.expo.dev/guides/monorepos/); layout is judgment). **M-H**
- Metro is configured automatically for monorepos since SDK 52, autolinking's module resolution is on in monorepos since SDK 55, and isolated pnpm installs work since SDK 54. "Duplicate React Native versions in a single monorepo are not supported", so a catalog pins one version and CI runs `pnpm why --depth=10 react-native`.
- SDK 58 targets iOS 16.4+, needs Xcode 26.4+, builds with the iOS 27 SDK, and generates a `SceneDelegate` because Xcode 27 builds need the UIScene life cycle. "CocoaPods remains the default and supported path". React Native's plan is "0.89: Target Swift Package Manager as the default, with an opt-out to CocoaPods", and "We plan to remove CocoaPods support in a later release" ([RN 0.88](https://reactnative.dev/blog/2026/10/19/react-native-0.88)). So the Podfile, the `Pods/` cache, `pod install` in Xcode Cloud and the CocoaPods reason to avoid the JSON `.xcproj` are temporary; the `upgrade-expo-sdk` skill carries the move to SwiftPM.
- SDK 58 "does not support macOS yet. Support will return when react-native-macos is updated to at least React Native 0.86", and react-native-macos's latest release is 0.83.0 (2026-09-30, [npm](https://registry.npmjs.org/react-native-macos)). So the move to react-native-macos is at least one SDK away, and CI needs no macOS-native lane yet.
- Run one `expo prebuild`, then commit `app/ios/` ([cross-platform-ui-stack.md](cross-platform-ui-stack.md) §4.7). Add the watch app, watch widgets, iOS widgets, NSE and App Intents targets in Xcode, linking local Swift packages from `apple/Packages/`. The Podfile covers the app target only; `Podfile.lock` is committed and `Pods/` is not. Agents never run `expo prebuild` again: a deny rule blocks it (§7), and SDK upgrades port the native diff by hand (`upgrade-expo-sdk` skill).
- Dependabot must not bump `expo`, `expo-*`, `react`, `react-native` or Expo-managed libraries: SDK upgrades move them together. CI runs `npx expo install --check` (judgment).
- SDK 58 Router defaults `web.output` to `server`; set `single` for the offline PWA ([web-client-sync.md](web-client-sync.md) §3.4).

```text
app/              Expo app (iPhone, web; Mac as the web app or the iPad app); app/ios/ committed; app/modules/ Swift Expo modules (Security, WatchRelay)
server/           TypeScript API, worker, CLI; migrations/ as plain SQL
packages/         domain, sync-core, contract (schemas, golden vectors), design-system, strings (generated from l10n/)
apple/            Packages/ (DesignSystem, Strings, WatchBridge: generated or Swift-only) and Targets/ (watch, widgets, NSE, intents)
design/ l10n/     DTCG tokens and component specs; the ICU catalog
e2e/              Maestro flows, Playwright specs, the test-household seed
nix/              packages, modules, VM tests, ops template (nixos-vps-infra.md)
tools/            generators: tokens, strings, spec-codegen, test household
```

## 3. Static checks

**Biome 2.6.6 versus ESLint 10.14 plus Prettier 3.9.**
- Biome formats and lints JavaScript, TypeScript, JSX, JSON, CSS, GraphQL, Markdown and YAML; HTML and Vue/Svelte/Astro are partial ([language support](https://biomejs.dev/internals/language-support/)). Measured: 2.6.6 reformatted a Markdown list and table and a YAML file. **H**
- Its `reactNative` domain, enabled by the `react-native` dependency, carries `noReactNativeRawText`, `noReactNativeDeepImports`, `useReactNativePlatformComponents` and `noReactNativeLiteralColors` ([domains](https://biomejs.dev/linter/domains/)). The last names `react-native/no-color-literals` as its source ([rule](https://biomejs.dev/linter/rules/no-react-native-literal-colors/)) but differs from it. Biome's walks every descendant of the call, so it would cover the function form and nested variants, but it accepts `StyleSheet` only when "imported from `react-native`/`react-native-web` or is an unresolved global"; Unistyles' `StyleSheet` comes from `react-native-unistyles`, so the rule skips it. It also checks only string values of properties whose name contains `color`, and has no options ([source at 2.6.6](https://github.com/biomejs/biome/blob/%40biomejs/biome%402.6.6/crates/biome_js_analyze/src/lint/style/no_react_native_literal_colors.rs)). ESLint's rule reads only an object argument (`node.arguments[0].properties`), so it misses the function form ([source](https://github.com/Intellicode/eslint-plugin-react-native/blob/master/lib/util/stylesheet.js), [design-system.md](design-system.md) TL;DR 5). **H**
- GritQL plugins "can match specific code patterns, report customized diagnostics, and suggest fixable rewrites". They "run on every file" unless an `includes` list scopes them, and they can't see the semantic model. JavaScript plugins can ("the file being analyzed, the semantic model, CST navigation, and more"), but they are a "Developer Preview" in which "Breaking changes can happen in patch releases" ([plugins](https://biomejs.dev/linter/plugins/)). **H**
- **Measured with Biome 2.6.6:**
  - The plugin below flagged all six raw values in a sample: two colors, `padding: 12` and `borderRadius: 8`, plus `fontSize: 32` and an `hsl()` shadow color inside a nested variant. It passed every `theme.*` reference.
  - **But it allows by default.** A fact-checker's run of Biome 2.6.6, not repeated for this revision, found that it flags no named color (`'white'`, `'transparent'`), no template-literal color, and no number under `minHeight`, `maxWidth`, `borderWidth` or `letterSpacing`. Its key regex, tested alone with Python's `re`, also misses `rowGap`, `columnGap`, `minWidth`, `maxHeight`, the per-corner radii such as `borderTopLeftRadius`, `start`, `end`, the `inset*` props and quoted keys, all of them React Native style props ([layout](https://reactnative.dev/docs/layout-props), [view](https://reactnative.dev/docs/view-style-props)). A negative literal is a unary expression, not a `number()` (judgment).
  - A second plugin, ``<$tag style={$v} $... />`` and its open-tag twin, flagged both `style` props of a sample (an inline object and `styles.title`).
  - A third, `JsxText()` matching a letter, flagged `<Text>Safe to spend</Text>` and passed `{m('budget.safeToSpend')}`.
  - Capture groups in GritQL regexes bind variables and abort the plugin ("regex pattern matched 1 variables, but expected 0"), so write `(?:…)`.
  - Biome 2.6 deprecates `linter.rules.recommended` in favor of `preset`.

```grit
language js
`StyleSheet.create($arg)` where {
  $arg <: contains bubble or {
    pair(key=$k, value=$v) where { $v <: string(), $v <: r"^['\"](?:#|rgba?\(|hsla?\(|oklch\().*",
      register_diagnostic(span=$v, message="Design system: use a theme color token") },
    pair(key=$k, value=$v) where { $v <: number(),
      $k <: r"^(?:margin|padding|gap|fontSize|lineHeight|borderRadius|top|left|right|bottom|width|height)[A-Za-z]*$",
      register_diagnostic(span=$v, message="Design system: use a space, radius or type token") }
  }
}
```

**Widen it before calling the design system strict** (judgment; spike F9 writes and tests the new version):
- Colors: flag every string or template literal under a key that contains `color`, the test Biome's own rule uses, so named colors fail too.
- Sizes: flag every number and negated number except `0` under any key outside a short unitless list (`flex`, `flexGrow`, `flexShrink`, `zIndex`, `opacity`, `aspectRatio`), and every string length except a percentage. A style prop nobody listed is then denied by default.
- Fixtures: each miss above, ternaries and quoted keys must fail; `0`, percentages, `-theme.space(1)` and every token form must pass.
- Scope with `includes`: the raw-value plugins skip `packages/design-system/`, and the JSX-text plugin skips stories and tests.
- A JavaScript plugin could also check where `StyleSheet` comes from; wait until that API leaves developer preview.

- **ESLint's remaining job** was the React Compiler rules: "React Compiler lint rules are included by default with `eslint-config-expo`" since SDK 55 ([React Compiler](https://docs.expo.dev/guides/react-compiler/)). Biome 2.5.8 added `useReactCompiler`, which "runs React Compiler in lint mode and reports the actionable diagnostics it emits"; it is nursery, "experimental and the behavior can change at any time" ([rule](https://biomejs.dev/linter/rules/use-react-compiler/)). If the compiler is on, enable it as a warning, like `noFloatingPromises`. Keep `eslint-config-expo` 58 in CI and the stop hook, without type information and with the TypeScript 6 alias (§2), only if F9 shows Biome missing diagnostics that matter (judgment). Oxlint's ESLint-compatible JS plugins are "currently in alpha" and lack type-aware rules ([oxlint](https://oxc.rs/docs/guide/usage/linter/js-plugins.html)), and oxfmt is still 0.77, so no. **M**
- **Type-aware rules** come from `tsc` and from Biome's `types` domain (`noFloatingPromises`), which is still nursery ([rule](https://biomejs.dev/linter/rules/no-floating-promises/)). Enable it as a warning and revisit when it leaves nursery. **M**

**The other rails** (versions as pinned in §2):
- **Swift:** SwiftFormat `--lint` and SwiftLint `--strict`, with [design-system.md](design-system.md) §6's raw-value custom rules, over `apple/` and `app/modules/`. nixpkgs builds SwiftLint for Linux, so they can run on the Linux hygiene job (whether its Linux build runs every rule: spike F9). Periphery stays weekly and advisory ([factory-ci-quality.md](factory-ci-quality.md)).
- **Nix:** treefmt (2.6) maps `*.nix` to nixfmt 1.5.0 (RFC 166 style), `*.swift` to SwiftFormat and the rest to Biome, so the edit hook and CI share one formatter call. statix now comes from the Molybdenum Software fork that nixpkgs packages; the original moved to tangled ([statix](https://github.com/oppiliappan/statix)). deadnix 1.3.2. All run in `nix flake check`. **H**
- **SQL:** squawk 2.71.0 ([releases](https://github.com/sbdchd/squawk)) on every new file in `server/migrations/`, with expand/contract migrations only ([self-hosting.md](self-hosting.md) §3.1). A check regenerates migrations from the Drizzle schema and fails if a new one appears (judgment). **M**
- **Markdown:** Biome formats it; markdownlint-cli2 0.24.0 is optional (advisory). `typos` 1.51 checks spelling, which the `ship` skill looks for.
- **Commits:** squash-merge only; the PR title is a Conventional Commit, checked by an inline script that reads the title from `env` (no third-party action, no template injection). lefthook's `commit-msg` runs commitlint 21 locally, as advice: the squash title is what release-please reads.
- **Repo-wide:** knip 6.44 for unused files and dependencies (advisory), `expo-doctor` and `expo install --check` on dependency changes, generated-file drift checks (tokens, strings, spec enums) by regeneration plus `git diff --exit-code`, and a gitleaks rule for IBAN-shaped strings next to its defaults, so recorded bank fixtures can't leak account identifiers (judgment).

## 4. Tests

| Layer | Tool (2026-11-07) | Runs on | When | Gate |
|---|---|---|---|---|
| Domain math, mutators, sync core, server units, generators | Vitest 5.1.0, coverage v8 with `thresholds` (floor on `packages/domain`) | Linux | Stop hook (related tests), PR | Blocking |
| Sync simulator | fast-check 4.10.2 `fc.commands` + `fc.scheduledModelRun` + `fc.scheduler`, seeded; failing seeds kept as cases | Linux | PR (fixed seed count), nightly (100,000+ seeds, [sync-engine.md](sync-engine.md)) | PR blocking, nightly advisory |
| React Native components | Jest 30.5 + jest-expo 58.0 + @testing-library/react-native 14.1 | Linux | PR | Blocking |
| Story tests, web component a11y | Storybook 10.6.6 (`react-native-web-vite`) + `@storybook/addon-vitest` in Vitest browser mode (Playwright); a11y addon | Linux | PR | Blocking once F7 passes |
| Server integration | Vitest against PostgreSQL 18.6 from the devshell: one throwaway cluster per worktree on a Unix socket, one database per test file cloned from a template (judgment) | Linux, laptops | PR | Blocking |
| Connector contract kit | Fake provider server, synthetic golden files ([connectors-and-license.md](connectors-and-license.md) §5.4) | Linux | PR; nightly sandbox smoke | PR blocking |
| Web end-to-end | Playwright 1.65.0 against the exported web app and a local server seeded with the test household: offline, multi-tab and sign-in flows in Chromium, WebKit and Firefox | Linux (Playwright image `v1.65.0-noble`) | PR | Blocking |
| Web visual and a11y | `toHaveScreenshot` baselines in the repo; @axe-core/playwright 4.14.0 | Linux, same image | PR | Blocking once baselines settle |
| iOS components (images) | React Native Harness 1.7.0 on one pinned simulator, or the XCUITest fallback ([cross-platform-ui-stack.md](cross-platform-ui-stack.md) §4.6) | macOS `xcode-27` | Native PRs, main | Advisory, then blocking |
| iOS end-to-end | Maestro CLI 2.12.0 flows from the devshell (2.12.1 shipped 2026-11-02 and arrives once nixpkgs packages it) on the Debug build (Java 17+ required; the image ships Java 21) | macOS `xcode-27` | Native PRs (smoke), main (full) | Smoke blocking |
| iOS a11y, launch, hitches | XCUITest `performAccessibilityAudit()`, launch and hitch metrics | macOS | main | Advisory |
| Watch, widgets, Swift modules | `swift test` on packages; `xcodebuild test` on the watch scheme, with the destination pinned to watchOS 27.0, the image's only watch runtime; `ImageRenderer` diffs | macOS | Native PRs, main | Blocking |
| NixOS module | `nix flake check`: `runNixOSTest` smoke, backup-restore and upgrade tests ([nixos-vps-infra.md](nixos-vps-infra.md)) | Linux x64 (KVM) | PR, main | Blocking |
| Strings, tokens, specs | `just l10n` (ICU subset, `formatjs verify` on JSON and generated `.xcstrings`), `just ds-check` (regeneration diff, 16-permutation contrast), spec parity | Linux | PR | Blocking |

- **Why Jest for components.** Expo's guide uses jest-expo and says "In SDK 58 and later, `jest-expo` requires Jest 30", and it doesn't mention Vitest ([unit testing](https://docs.expo.dev/develop/unit-testing/)). Vitest for React Native exists as community packages, such as `@srsholmes/vitest-react-native` 0.3.0. RNTL 14 makes `render` and `fireEvent` async and needs React 19 and React Native 0.78+ ([v14 migration](https://oss.callstack.com/react-native-testing-library/docs/start/migration-v14)). **H**
- **Maestro over Detox.** Detox is "Fully compatible" with React Native 0.77 to 0.84, and "Newer RN versions might work … but they've not been thoroughly tested" ([Detox](https://github.com/wix/Detox)); its latest stable release is 20.51.4 (2026-06-16), and 21.0 is in alpha: 21.0.0-alpha.1 (2026-09-04), while the `rc` tag still points to a 2023 build ([npm](https://registry.npmjs.org/detox)). Maestro is black-box, so it doesn't depend on React Native's version. Its CLI needs "Java version 17 or higher" ([install](https://docs.maestro.dev/maestro-cli/how-to-install-maestro-cli)). E2E sign-in can't drive the passkey sheet: spike F6. **M-H**
- **The simulator** ([sync-engine.md](sync-engine.md) testing strategy, ported to TypeScript). `fc.scheduler` "has been specifically designed for detecting and testing race conditions" ([race conditions](https://fast-check.dev/docs/advanced/race-conditions/)), and `fc.scheduledModelRun` runs command sequences against a model under that scheduler ([model-based](https://fast-check.dev/docs/advanced/model-based-testing/)). Two to four clients, the real server mutators and an in-memory store run in one process, with the invariants from sync-engine.md. **H** on the tool, **M** on the fit.
- **Visual regression.** Playwright baselines live in the repo and change only in a dedicated PR. Chromatic's free plan gives "5,000 billed snapshots" a month in Chrome only; it is "Free for open source" on request, without stated criteria ([pricing](https://www.chromatic.com/pricing)). It's unnecessary while baselines live in git. Storybook's own test runner "has been superseded by the Vitest addon" ([test runner](https://storybook.js.org/docs/writing-tests/integrations/test-runner)). The addon requires a Vite-based framework ([addon](https://storybook.js.org/docs/writing-tests/integrations/vitest-addon)), and `react-native-web-vite` is one, but that pairing is unverified (spike F7). **M**
- **NixOS VM tests need KVM.** GitHub-hosted Linux runners "support hardware acceleration for Android SDK tools" ([runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)); since 2024-04 that covers standard runners, after a udev rule grants the runner user `/dev/kvm` ([changelog](https://github.blog/changelog/2024-04-02-github-actions-hardware-accelerated-android-virtualization-now-available/)). `cachix/install-nix-action` v31.11.3 enables KVM "on supported machines" ([self-hosting.md](self-hosting.md) §2.3). Whether `ubuntu-24.04-arm` exposes KVM is unverified (one third-party report says no), so VM tests run on x64 and the arm64 runner only builds. **H** for x64.
- **Fixtures** come from one seeded generator for the test household (Faker 10.6). It feeds E2E runs, screenshots, local runs and the test household of `PRODUCT.md`, so no test touches real bank data (judgment). There is no public demo instance: it would be a service run for other people (ADR 0001), and the reference deployment is tailnet-only (ADR 0003).

## 5. CI on GitHub Actions: public repo, Free plan

**Plan facts** (rechecked 2026-11-07). **H**
- "Use of the standard GitHub-hosted runners is free and unlimited on public repositories", macOS included ([runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)).
  - Linux x64 and arm64: 4 CPUs, 16 GB (private repos such as the ops repo: 2 CPUs, 8 GB).
  - macOS arm64: 3 M1 cores, 7 GB of RAM, a 14 GB SSD (`macos-26`, `xcode-27`).
  - `ubuntu-slim`: 1 CPU in a container, with a 15-minute job limit; fine for `gate`.
- Free plan limits ([limits](https://docs.github.com/en/actions/reference/limits)): 20 concurrent jobs, 5 of them macOS; 6 hours per job; 10 GB of Actions cache per repository; `GITHUB_TOKEN` at 1,000 API requests per hour.
- `xcode-27` (and `xcode-27-xlarge`, Apple silicon only) is generally available since 2026-10-15, and "If you already use these labels, you don't need to change your workflows" ([changelog](https://github.blog/changelog/2026-10-15-xcode-27-runner-image-generally-available/)); the runner docs list it with no preview note. The "preview" badge in the runner-images README and the pinned issue [#14404](https://github.com/actions/runner-images/issues/14404) are stale. Its base has been macOS 27 since 2026-09-16 (#14404).
- Image 20261026 carries Xcode 27.0, 27.1 (default) and a 27.2 Release Candidate, plus Node 24.22, CocoaPods 1.17 and Java 21. Its installed simulator runtimes are iOS 27.0, iOS 27.1 (iPhone Duo only) and watchOS 27.0; 27.2 exists only as the RC's SDKs ([readme](https://github.com/actions/runner-images/blob/main/images/macos/xcode-27-arm64-Readme.md)). So the iPhone and watch test destinations pin `OS=27.0`.
- `macos-26` tops out at Xcode 26.6 with the watchOS 26.5 SDK ([readme](https://github.com/actions/runner-images/blob/main/images/macos/macos-26-arm64-Readme.md)), so the watchOS 27 floor forces `xcode-27`. If the floor dropped to watchOS 26, `macos-26` would suffice for Expo SDK 58 (Xcode 26.4+).
- **Xcode selection.** The image policy keeps one Xcode major per image with "all minor versions of the supported major version"; "when a new patch version is released, the previous patch version will be replaced", and beta and RC builds come "as-is" ([runner-images](https://github.com/actions/runner-images)). So `.xcode-version` holds major.minor (`27.1`). The `apple` job selects `/Applications/Xcode_27.1.app`, which follows patch replacements, falls back to the default Xcode only if it has the same major.minor, and otherwise fails with a message naming the missing version. It resolves the link first and refuses a beta or RC: today `/Applications/Xcode_27.2.app` points to `Xcode_27.2_Release_Candidate.app` (judgment on the script).

**Caching** (judgment on choices, facts as cited):
- **Platform rules** ([dependency caching](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching)). Only `push`, `schedule`, `workflow_dispatch` and a few other triggers "can create or overwrite caches in the default branch's scope"; other runs on the default branch get read-only access. `pull_request` caches are "scoped to the merge ref (`refs/pull/.../merge`)" and restore only on re-runs of that PR. So `main` pushes save the caches that PRs restore. A `cache-mode` key (`read`, `write`, `write-only`, `none`), set per workflow or job, is "enforced with scoped cache tokens". **H**
- **Nix:** Magic Nix Cache, "a totally free and zero-configuration binary cache" on the Actions cache, works on Linux and macOS; it has "No secrets", so fork PRs benefit, and it may hit the cache's rate limits on big rebuilds ([README](https://github.com/DeterminateSystems/magic-nix-cache-action); action v15, 2026-09-09). It runs its daemon from the `main` branch unless a `source-*` input pins it, so pin `source-tag`. Cachix gives "a free 5 GB limit for open source projects" ([pricing](https://www.cachix.org/pricing)) and is the upgrade path, since self-hosters could pull from a public cache. **M-H**
- **pnpm:** cache `pnpm store path` keyed on the lockfile hash. **CocoaPods:** cache `Pods/` keyed on `Podfile.lock`, until SwiftPM replaces it (§2). **TypeScript:** cache the `.tsbuildinfo` files.
- **Budget the 10 GB.** GitHub removes entries not accessed in 7 days and, once the repository is full, deletes the least recently used first. Nix paths for three platforms, the pnpm store, `Pods/` and `.tsbuildinfo` share it, so key by OS and architecture, save from `main` only, and measure the footprint in F3.
- **Release jobs set `cache-mode: none`.** zizmor's `cache-poisoning` audit flags "Potentially compromised reads from caches, within release workflows", says "release workflows should not read from the GitHub Actions cache", and recommends that mode ([audits](https://docs.zizmor.sh/audits/)). The platform then keeps release builds clean, not a convention.

**Gate and rulesets.**
- Path-filter jobs with an in-repo `changes` job: `git diff --name-only` against the merge base, emitted as outputs. A `gate` job on `ubuntu-slim` `needs` every job and fails on `failure` or `cancelled`; it is the only required check, so skipped jobs never block ([factory-ci-quality.md](factory-ci-quality.md)). **H**
- Ruleset on `main`:
  - require a PR, with 1 approval, dismissal of stale approvals, approval of the latest push, and review from Code Owners. `CODEOWNERS` holds one line, `* @<owner>`, so only the owner's approval counts. Without it, the machine account could approve any PR it didn't push last, since collaborators can "Submit a review on a pull request that affects the mergeability of the pull request" ([permission levels](https://docs.github.com/en/account-and-profile/reference/permission-levels-for-a-personal-account-repository));
  - require `gate`, conversation resolution and linear history; squash only;
  - block force pushes and deletion;
  - also require code scanning results, which can block merges on alerts of a chosen severity ([rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets));
  - bypass: only the owner's admin role, "For pull requests only", so the owner can merge a PR of their own ([creating rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository)).
- Tag ruleset on `v*`: restrict creation, update and deletion; bypass: the owner and the release GitHub App. GitHub Apps are eligible bypass actors (same page); whether a personal account's repo accepts one is unverified. **M**
- Leave "Require branches to be up to date" off: without a merge queue (organization repos only), strict mode reruns the macOS job after every merge, and `main` CI catches semantic conflicts (judgment; owner question 2).

**Agent identity** (the change from factory-ci-quality.md, where agents used the owner's token). **M-H**
- A machine account "set up by an individual human"; "You may maintain no more than one free machine account in addition to your free Personal Account" ([terms](https://docs.github.com/en/site-policy/github-terms/github-terms-of-service)).
- It joins as a collaborator with write access. It can't edit rulesets or approve a PR it pushed last. It can still "Create, merge, and close pull requests", approve PRs that others pushed (Dependabot's, contributors', release-please's), and "Create and edit releases" ([permission levels](https://docs.github.com/en/account-and-profile/reference/permission-levels-for-a-personal-account-repository)). Code-owner review is what makes the owner's approval a real gate; the tag ruleset and attestations guard releases.
- Fine-grained tokens can't serve "repositories where the user is an outside or repository collaborator" ([tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)). So the machine account uses a classic `repo` token, and its only repository is this one.
- Agents get that token through `GH_TOKEN` and git credentials. The owner's own token stays out of agent sessions.

**Security tooling** (H on facts):

| Rail | Setup | Note |
|---|---|---|
| CodeQL | Default setup: JavaScript/TypeScript (TypeScript 2.6 to 7.0) and GitHub Actions workflows ([languages](https://codeql.github.com/docs/codeql-overview/supported-languages-and-frameworks/)). Eligible because the repo "is publicly visible" ([default setup](https://docs.github.com/en/code-security/code-scanning/enabling-code-scanning/configuring-default-setup-for-code-scanning)) | Deselect Swift. It supports only the `autobuild` and `manual` build modes ([build options](https://docs.github.com/en/code-security/reference/code-scanning/codeql/build-options-for-compiled-languages)), and default setup would use `autobuild`; CodeQL 2.27.3 dropped both "on macOS 27 with any Xcode version, and on macOS 26 when Xcode 27 is selected" ([2.27.3](https://github.com/github/codeql-cli-binaries/releases/tag/v2.27.3)). Support also stops at "Swift 5.4-6.3", and Xcode 27 ships 6.4. So the watch app and Swift modules have no CodeQL path: owner question 7 |
| Dependabot | `npm` (pnpm), `swift` (`apple/Packages/*/Package.swift`), `nix` (`flake.lock`), `github-actions`, weekly, grouped; `cooldown.default-days: 7` in all four ecosystems, since zizmor's `dependabot-cooldown` flags less than its default of 7 days, and without the key Dependabot applies 3 days to version updates ([options](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/dependabot-options-reference), [audits](https://docs.zizmor.sh/audits/)). Security updates skip cooldowns, but pnpm applies `minimumReleaseAge` to every dependency it resolves, so an urgent fix younger than a day goes into `minimumReleaseAgeExclude` by exact version (`<name>@<version>`) in its PR ([pnpm](https://pnpm.io/settings/dependency-resolution)); whether Dependabot's own pnpm run honours the limit is unverified | `nix`: `flake.lock` version updates only, no `flake.nix` refs, and it "does not currently support private repositories". Expo-managed packages are ignored. Security updates need the single-document lockfile (§2) |
| Renovate 44.222 | Not now | Its `nix` manager is beta and opt-in ([nix](https://docs.renovatebot.com/modules/manager/nix/)). Post-upgrade commands, which could recompute the Nix pnpm hash, are "blocked by default" and need an admin `allowedCommands` list ([options](https://docs.renovatebot.com/configuration-options/#postupgradetasks)). Revisit if hash churn hurts (F8) |
| OpenSSF Scorecard | Action v2.5.0, weekly and on `main`, `publish_results: true` | Runs with the default `GITHUB_TOKEN` and reads rulesets with it ([action](https://github.com/ossf/scorecard-action)). Advisory |
| Secrets | GitHub secret scanning and push protection (on); gitleaks 8.30.1 in lefthook (`--staged`) and CI (diff on PRs, full history nightly), run from the devshell, not an action | Plus the IBAN rule (§3) |
| Actions supply chain | `.github/workflows/actions.lock`, generated by `gh actions-lock`, pins every action, "including transitive dependencies". "For repositories that adopt a lockfile, SHA pinning within workflow files is no longer necessary", and the lockfile satisfies the "Require actions to be pinned to a full-length commit SHA" policy for the workflows onboarded to it ([lockfiles](https://docs.github.com/en/actions/concepts/security/workflow-dependency-lockfiles)). It "doesn't pin calls to reusable workflows", so `jobs.<id>.uses` keeps full SHAs, and it doesn't cover what an action downloads at run time, so Magic Nix Cache's daemon is pinned with `source-tag` (judgment) | Dependabot reads the lockfile's structured record instead of version comments |
| Workflow lint | zizmor 1.32.0 (`dangerous-triggers`, `template-injection`, `artipacked`, `excessive-permissions`, `dependabot-cooldown`, and `secrets-outside-env` under its auditor persona) and actionlint 1.7.12 | Blocking |
| Disclosure | Private vulnerability reporting, open to public repos ([docs](https://docs.github.com/en/code-security/security-advisories/working-with-repository-security-advisories/configuring-private-vulnerability-reporting-for-a-repository)), plus `SECURITY.md` | — |

**Hardening rules** (each from GitHub's [secure use](https://docs.github.com/en/actions/reference/security/secure-use) page or zizmor; their combination is judgment):
- No `pull_request_target`: workflows on that trigger "must not explicitly check out untrusted code". Fork PRs run on `pull_request`, and "secrets are not passed to the runner when a workflow is triggered from a forked repository"; nor are they "available to workflows triggered by Dependabot events" ([secrets](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets)). Turn on approval for all outside contributors.
- No repository secrets at all: "Any user with write access to your repository has read access to all secrets configured in your repository" (secure use), and the machine account has write access. Every secret sits in an environment bound to `main` (`release`: the release App key; `nightly`: bank-sandbox credentials) or to `v*` tags (`testflight`, fallback only).
- `permissions: {}` at the top of each workflow, minimal per job. The default token is read-only, and Actions may not create or approve PRs. Checkout uses `persist-credentials: false`.
- Untrusted text (PR titles, branch names) reaches scripts only through `env:`, never `${{ }}` inside `run:`.
- OIDC instead of long-lived keys wherever possible: attestations, Scorecard, the Tailscale join.

## 6. CD

**iPhone, watch and extensions to TestFlight.**

| Path | Signing material | Cost and limits | Fit |
|---|---|---|---|
| **Xcode Cloud** archive on tag `v*` (pick) | Apple-managed; nothing in GitHub ([factory-ci-quality.md](factory-ci-quality.md) §2) | 25 compute hours a month included; the post-clone script installs Node 26 and the devshell's exact pnpm, then JS and Pods, on every action (judgment: about 25 minutes per archive, so 8 releases use about 3.3 hours) | Builds the committed workspace with every target; logs live in App Store Connect |
| GitHub Actions: `xcodebuild archive` + `-exportArchive` upload with `-allowProvisioningUpdates` and an App Store Connect team key | The `.p8` key, Admin role in practice, in the `testflight` environment bound to `v*` tags | Free macOS minutes | Fallback if F1 fails; the key can do nearly anything in the developer account, so the tag ruleset is mandatory |
| fastlane `match`/`pilot` | Certificates in storage plus a passphrase | Ruby toolchain | Not needed: cloud signing covers App Store distribution |
| EAS Build + Submit | Credentials held by Expo | Free: "15 Android and 15 iOS builds", "Low priority", a "45-minute build timeout"; Starter $19 a month ([pricing](https://expo.dev/pricing)) | Not needed with a committed `ios/` ([cross-platform-ui-stack.md](cross-platform-ui-stack.md) §4.6) |

- **Internal TestFlight only.** The internal group needs no review, and builds expire after 90 days, so tag at least every 90 days ([native-apps-auth.md](native-apps-auth.md) §8). The iPad build on Apple-silicon Macs is a TestFlight group setting, so the Mac path needs no extra pipeline ([cross-platform-ui-stack.md](cross-platform-ui-stack.md) §4.2). **H**
- **Self-hosters' own builds** (PRODUCT.md; ADR 0019: "setting the team ID, bundle ID, app group, URL scheme and APNs key from untracked configuration"; the layout is judgment). Every target in `app/ios/` reads the team ID, bundle-ID prefix, app group, Keychain access group and URL scheme from one xcconfig. A committed example holds placeholders under `example.invalid`; each builder writes an untracked local copy, and the maintainer's values reach Xcode Cloud as workflow environment variables, so no instance or account is hard-coded. CI builds the simulator app from the placeholders. No Associated Domains entitlement is needed: the apps sign in through the browser sheet with a custom-scheme callback, never the native passkey sheet (ADR 0011).
- **Versions** (judgment): one SemVer for the product (server, web, apps), as [self-hosting.md](self-hosting.md) §3.5 chose.
  - release-please updates `package.json` files and a shared `Version.xcconfig` holding `MARKETING_VERSION`, through `x-release-please-version` annotations on any text file ([customizing](https://github.com/googleapis/release-please/blob/main/docs/customizing.md)).
  - The build number is Xcode Cloud's monotonic `CI_BUILD_NUMBER`, applied to every target in `ci_pre_xcodebuild.sh`. The watch's `CFBundleVersion` must match the app's, or App Store Connect rejects the upload (error 90379 in [apple-targets #147](https://github.com/EvanBacon/expo-apple-targets/issues/147); secondary).
  - Don't use app.json's version fields: they stop syncing once `ios/` is committed.
  - TestFlight "What to Test" comes from the release-please changelog, in `ci_post_xcodebuild.sh`.
- **No over-the-air updates** ([cross-platform-ui-stack.md](cross-platform-ui-stack.md) §4.8), so every JavaScript fix ships through this path.

**Release automation: release-please 17.11 (action v5.0.0) over changesets 3.0.** **M**
- on-budget ships one product and no npm packages. release-please derives the version and changelog from the Conventional Commit titles that squash merges leave on `main`, so agents write no release files.
- changesets needs a file in each PR and suits independently versioned packages.
- Release PRs made with `GITHUB_TOKEN` "will not trigger future GitHub actions workflows" ([action](https://github.com/googleapis/release-please-action)), so `gate` would never report on them. So release-please runs with a GitHub App token, from the `release` environment.
- release-please itself creates the tag and the GitHub Release when its PR merges. `skip-github-release` would leave no tag, since "Release-Please still requires releases to be tagged". So it creates a draft (`draft: true`) with `force-tag-creation: true`, because GitHub otherwise creates a draft's tag only when the draft is published ([manifest options](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md)). The tag workflow attaches its artifacts with `gh release upload` and then publishes the draft.
- The installed `ship` skill writes changesets "where required" and skips them "when the repository's release policy permits it", so AGENTS.md states that policy.

**Server, web app and image.**
- The web app ships inside the server package, which serves it from the store ([nixos-vps-infra.md](nixos-vps-infra.md) Ingress), so the web needs no deploy of its own.
- On tag ([self-hosting.md](self-hosting.md) §2.2, §2.6): `buildLayeredImage` on `ubuntu-24.04` and `ubuntu-24.04-arm`; `skopeo` push by digest; `docker buildx imagetools create` for the index with tags `X.Y.Z`, `X.Y`, `X` and `latest`; `actions/attest` v4.2.2 with `push-to-registry`; then it uploads the compose bundle and the digests to release-please's draft and publishes it. `main` publishes `edge` the same way. The GHCR package is made public once. The image, `edge` and the compose bundle start once the server is stable (ADR 0018); until then the tag workflow only publishes the draft. **M-H**
- Spike F5: an `expo export` of the web app inside a Nix derivation, offline, with Metro's caches.

**The private ops repo** ([nixos-vps-infra.md](nixos-vps-infra.md) CI/CD).
- **Phase 1:** a scheduled job runs `nix flake update on-budget` and the host VM test in one workflow, then opens the PR; Dependabot's `nix` support is no fallback here, since it "does not currently support private repositories". Private repos get 2-CPU, 8 GB runners, which have KVM ([KVM changelog](https://github.blog/changelog/2024-04-02-github-actions-hardware-accelerated-android-virtualization-now-available/)), and draw on the Free plan's included compute (Costs). Then the owner runs `just deploy` by hand: from a Mac, `just deploy-here` builds on the VPS (`deploy --remote-build`).
- **Phase 2:** a `deploy` job joins the tailnet with `tailscale/github-action` v4.4.0 through workload identity federation (`id-token: write`, Tailscale 1.90.1+, ephemeral `tag:ci` nodes; [action](https://github.com/tailscale/github-action)), then runs deploy-rs 1.0.0 (2026-10-19, its first crates.io release) with magic rollback.
  - That job needs environment secrets, which private repos get only with GitHub Pro.
  - The pull-based alternative, comin 0.14.0, polls the repo from the host and deploys itself, so CI holds no SSH key ([comin](https://github.com/nlewo/comin)). Its rollback behaviour is undocumented on that page.

  **M**

## 7. Agentic factory

**AGENTS.md** (Claude Code reads it through the root `CLAUDE.md`'s `@AGENTS.md`; Codex reads root-to-leaf files up to 32 KiB, [factory-ci-quality.md](factory-ci-quality.md) §4).
- The root file stays under about 4 KiB. It holds the public-repository rule, the `just` commands, the layout, the invariants (money in integer minor units, glossary terms, tokens only, strings in en and fr, expand/contract migrations, no real bank data) and the definition of done.
- Nested files sit in `app/`, `app/ios/`, `apple/`, `server/`, `packages/sync-core/` and `nix/`, each with its own commands and pitfalls: never run `expo prebuild`; never edit generated files; give every module change a VM test; never change `stateVersion`.
- A "Release policy" line says: no changesets; release-please from PR titles.
- A hooks line says that Codex skips new or changed repo hooks until the owner trusts them in `/hooks`, so a PR that edits `.codex/hooks.json` says so.

**Hooks.** The Claude Code facts are from its [hooks](https://code.claude.com/docs/en/hooks) page, the Codex facts from its [hooks](https://learn.chatgpt.com/docs/hooks) page. **H** on mechanics, judgment on the scripts.
- **Claude Code.**
  - `SessionStart` hooks can write `export` lines to `CLAUDE_ENV_FILE`, and "Each Bash command runs the file's contents as shell code before the command itself". So the hook puts the devshell's `PATH` and `PG*` variables into every agent command.
  - `PostToolUse` on `Edit|Write` runs `treefmt` on the file and shows errors to Claude with exit 2.
  - `Stop` runs `just check-fast` with the `stop_hook_active` guard. That recipe runs `tsc -b`, `biome ci --changed`, `vitest related`, Jest's related tests, and `swift test` or squawk when those paths changed, within a 3-minute budget.
- **Codex.** Repo hooks live in `.codex/hooks.json` and load only once the project is trusted. Each hook needs trust too: Codex "records trust against the hook's current hash", and "new or changed hooks are marked for review and skipped until trusted" through `/hooks`. So any edit to `.codex/hooks.json`, by an agent or a merge, silently drops the format and stop hooks until the owner trusts them again. `--dangerously-bypass-hook-trust` is meant for "one-off automation that already vets hook sources", not agent sessions (judgment). Claude Code, by contrast, normally picks up hook edits through its file watcher. The `PostToolUse` matcher can use `apply_patch`, `Edit` or `Write`. `Stop` continues on exit 2 or `"decision": "block"`. There is no documented env persistence, so Codex starts inside the devshell (direnv `use flake`), and `just` recipes re-enter it when `IN_NIX_SHELL` is unset.
- **Deny rules** (best effort; CI and the ruleset are the real gate): `git commit --no-verify`, `git push --force`, pushes to `main`, `gh pr merge --admin`, `gh api …/rulesets`, `gh secret`, `expo prebuild`, `pod deintegrate`, and `nix flake update` outside the dependency skill.

```json
{ "hooks": {
    "SessionStart": [{ "hooks": [{ "type": "command", "command": "${CLAUDE_PROJECT_DIR}/scripts/agent/env.sh" }] }],
    "PostToolUse": [{ "matcher": "Edit|Write", "hooks": [{ "type": "command", "command": "${CLAUDE_PROJECT_DIR}/scripts/agent/format.sh" }] }],
    "Stop": [{ "hooks": [{ "type": "command", "command": "${CLAUDE_PROJECT_DIR}/scripts/agent/stop.sh", "timeout": 300 }] }] },
  "permissions": { "deny": ["Bash(git commit --no-verify*)", "Bash(git push --force*)", "Bash(gh pr merge --admin*)",
                            "Bash(*expo prebuild*)", "Bash(gh api *rulesets*)", "Bash(gh secret *)"] } }
```

**Skills location.** Claude Code loads project skills from `.claude/skills/<name>/SKILL.md` ([skills](https://code.claude.com/docs/en/skills)). Codex "scans `.agents/skills` in every directory from your current working directory up to the repository root" ([Codex skills](https://learn.chatgpt.com/docs/build-skills)). Both follow the Agent Skills standard. So keep one copy in `.agents/skills/` and make `.claude/skills` a symlink to it (judgment). **H**

**MCP servers and simulator control.**

| Tool (2026-11-07) | Use | Note |
|---|---|---|
| MobileBuildMCP 2.7.1 (Sentry, MIT; formerly XcodeBuildMCP) | Build, test, run and log against simulators, from an MCP server or its CLI; ships an agent skill | Sentry error telemetry is on unless opted out ([README](https://github.com/getsentry/XcodeBuildMCP)) |
| Xcode `xcrun mcpbridge` | Previews, the debugger, simulator taps with Xcode open | Interactive only ([factory-ci-quality.md](factory-ci-quality.md) §4) |
| agent-device 0.21.36 (Callstack, MIT) | "CLI, MCP server, and typed Node.js API to control mobile, web, TV, and desktop apps on devices and simulators" ([npm](https://www.npmjs.com/package/agent-device)) | Taps, text input, snapshots, screenshots |
| Playwright MCP 0.0.84 | Drives the web app | Pre-1.0 |
| Storybook MCP addon 10.6.6 | Component docs and story tests for agents | Storybook calls its AI features a preview |
| mcp-nixos | NixOS options and packages for the pinned channel | Per [nixos-vps-infra.md](nixos-vps-infra.md) |
| Expo MCP (optional) | Docs, screenshots, taps, logs, EAS | Needs an Expo account. Doc search "requires an Expo Services paid plan". Local tools need `EXPO_UNSTABLE_MCP_SERVER=1`, iOS simulators only ([Expo MCP](https://docs.expo.dev/eas/ai/mcp/)) |

How an agent drives the iPhone app (judgment):
1. `just ios-run <screen>` boots the pinned simulator, builds the Debug app with MobileBuildMCP's CLI or `xcodebuild`, installs it, seeds the test household, and opens a deep link.
2. agent-device or a Maestro flow taps through the screen.
3. `xcrun simctl io booted screenshot` captures it in light and dark and in en and fr.
4. The paths go in the PR.

The web is the same loop with Playwright. Expo notes that Xcode 27 replaces Simulator.app with Device Hub ([SDK 58](https://expo.dev/changelog/sdk-58)); `simctl` remains the scripting surface.

**Installed skills in one flow** (read from their SKILL.md files):
1. `wayfinder` charts fog into a `wayfinder:map` issue with decision tickets. Research tickets call `research`; prototype tickets call `prototype`; grilling tickets call `grilling` and `domain-modeling`; task tickets do the blocking chore.
2. `to-spec` turns a settled discussion into a spec issue labelled `ready-for-agent`.
3. `to-tickets` cuts it into vertical slices with native blocking links.
4. `triage` moves issues through the five labels of `docs/agents/triage-labels.md`.
5. `implement` takes one ticket, uses `tdd` at agreed seams, runs typecheck and tests, then `code-review` (standards and spec, in parallel sub-agents).
6. `ship` validates, makes Conventional commits and opens the PR.
7. CI's `gate` runs, the owner approves, and the squash title feeds release-please.

**Definition of done** (judgment):
- The PR links its issue.
- Tests were written red first, at the seams agreed in the spec.
- `just check` passes locally and `gate` passes in CI.
- UI changes carry screenshots in light and dark and in en and fr, and use tokens and design-system components only.
- New strings exist in en and fr, and the owner reviews the French.
- Schema changes are expand/contract and pass squawk.
- Glossary terms are used, and ADRs or notes are updated when a decision changed.
- The diff holds no secrets, personal data or real bank data.
- A PR that changes `.codex/hooks.json` says so, and the owner trusts the changed hooks in Codex's `/hooks` after the merge.
- The PR title is a Conventional Commit, and the owner approved as code owner.

## Recommended pipeline

| Stage | Trigger | Jobs (runner) | Blocking |
|---|---|---|---|
| **PR** | `pull_request` (`cancel-in-progress` per ref) | `changes` and `gate` (`ubuntu-slim`) | `gate` is the only required check |
| | | `hygiene` (`ubuntu-24.04`): treefmt, Biome (`useReactCompiler` as a warning if the compiler is on), SwiftLint, zizmor, actionlint, gitleaks diff, PR title, typos | Yes |
| | | `js`: `tsc -b`, Vitest (units, a fixed property-test seed set), Jest components, Storybook Vitest + a11y, `just l10n`, `just ds-check`, spec parity, `expo install --check` and knip on dependency changes | Yes (knip advisory) |
| | | `server`: Postgres integration, squawk, migration drift, connector contract kit | Yes |
| | | `web`: Playwright end-to-end, screenshots and axe (Playwright image) | Yes |
| | | `nix` (x64, KVM): `nix flake check` (packages, VM tests, nixfmt, statix, deadnix), image build without push | Yes |
| | | `apple` (`xcode-27`), when `app/ios/`, `app/modules/`, `apple/` or native dependencies change, or label `ios-ci`: Xcode from `.xcode-version`, `pod install`, Debug build for the simulator with placeholder signing values, `swift test`, watch `xcodebuild test` on watchOS 27.0, Maestro smoke, Harness snapshots | Yes (snapshots advisory first) |
| **main** | `push` | All of the above, `apple` always: full Maestro suite, XCUITest audits and metrics; image `edge` pushed for amd64 and arm64 once the server is stable; release-please updates its PR | Red `main` means fix first (policy) |
| **Nightly** | `schedule` | Simulator sweep of 100,000+ seeds, flake repetition, full-history gitleaks, bank-sandbox smoke (`nightly` env); weekly Scorecard and Periphery | Advisory, opens an issue |
| **Release** | release-please PR merged → tag `vX.Y.Z` and a draft release | GitHub, with `cache-mode: none`: multi-arch image by digest, index, attestation, compose bundle, then the draft is published. Xcode Cloud: archive, internal TestFlight group | Yes |
| **Ops repo** | weekly `nix flake update` job; `main` push or dispatch | Host VM test on 2-CPU private runners (included compute); `just deploy` by hand, later deploy-rs over Tailscale | Yes |

```yaml
gate:
  if: always()
  needs: [changes, hygiene, js, server, web, nix, apple]
  runs-on: ubuntu-slim
  steps:
    - if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')
      run: exit 1
```

## Guard-rail list

| Guard rail | Tool (version, 2026-11-07) | Where it runs | Blocking? |
|---|---|---|---|
| Toolchain pins | Nix flake devShell (§2), `pnpm-lock.yaml`, `.xcode-version` (major.minor), `.node-version` and `.pnpm-version` checked against the devshell | Everywhere | Yes (CI uses them) |
| Format | treefmt 2.6 → Biome 2.6.6, nixfmt 1.5.0, SwiftFormat 0.64.3 | Edit hook, lefthook, CI | Yes |
| TS/JS lint | Biome 2.6.6 (recommended, `reactNative`, deny-by-default GritQL design-system and i18n plugins, `useReactCompiler` as a warning if the compiler is on); `eslint-config-expo` 58 only if F9 keeps it | Stop hook, CI | Yes |
| Typecheck | TypeScript 7.0.2 `tsc -b`, strictest flags | Stop hook, CI | Yes |
| Swift lint | SwiftLint 0.65.1 `--strict`, token custom rules | lefthook, CI | Yes |
| Nix lint | statix (nixpkgs fork), deadnix 1.3.2 | `nix flake check` | Yes |
| Migrations | squawk 2.71.0, Drizzle drift check | CI | Yes |
| Unit and property tests | Vitest 5.1.0, fast-check 4.10.2, coverage floor on `packages/domain` | Stop hook, CI, nightly | Yes; nightly advisory |
| Component tests | Jest 30.5 + jest-expo 58.0.14 + RNTL 14.1.0 | Stop hook, CI | Yes |
| Server integration | Vitest + PostgreSQL 18.6 from the devshell | CI, laptops | Yes |
| Web E2E, visual, a11y | Playwright 1.65.0, axe-core 4.14.0, Storybook 10.6.6 addon-vitest | CI Linux | Yes once baselines settle |
| iOS E2E and images | Maestro 2.12.0 (devshell); React Native Harness 1.7.0 | CI macOS `xcode-27` | Smoke yes; images advisory first |
| Watch and Swift | `swift test`, `xcodebuild test` (Xcode from `.xcode-version`, watchOS 27.0 simulator) | CI macOS | Yes |
| NixOS module | `runNixOSTest` VM tests | CI Linux x64 | Yes |
| Strings and tokens | `just l10n` (FormatJS CLI 6.18), `just ds-check` (Terrazzo 2.10) | Stop hook, CI | Yes |
| Dead code | knip 6.44.2; Periphery (Swift) | CI, weekly | Advisory |
| Secrets | Secret scanning and push protection; gitleaks 8.30.1 with an IBAN rule | GitHub, lefthook, CI | Yes |
| Code scanning | CodeQL default setup (JS/TS, Actions; Swift deselected, owner question 7) plus the ruleset rule | GitHub | Yes on high and critical |
| Dependencies | Dependabot (npm, swift, nix, github-actions) with 7-day cooldowns; pnpm `minimumReleaseAge`, `trustPolicy`, `allowBuilds` | GitHub, install | PRs; install-time blocks |
| Workflows | `actions.lock` (full SHAs only for reusable-workflow calls); zizmor 1.32.0; actionlint 1.7.12 | GitHub, CI | Yes |
| Posture | OpenSSF Scorecard action v2.5.0 | Weekly | Advisory |
| Commits | Conventional PR title (inline check); commitlint 21 in lefthook | CI, local | Title yes |
| Merge | Ruleset: PR, code-owner approval (`CODEOWNERS`: the owner only), `gate`, code scanning, linear, squash; `v*` tag ruleset | GitHub | Yes |
| Agent identity | Machine account, write role, classic token; owner token out of agent sessions | GitHub, laptops | Yes |
| Agent behaviour | Claude Code and Codex hooks (Codex hooks trusted again after each change); deny rules | Laptops | Best effort |
| Release integrity | Tag ruleset, environment-bound secrets, release jobs with `cache-mode: none`, attestations | GitHub | Yes |

## Skills and hooks to create

| Skill (`.agents/skills/…`) | Does | Done when |
|---|---|---|
| `run-app` | Boots the pinned simulator or a browser, seeds the test household, opens a screen by deep link, takes screenshots (light and dark, en and fr) | Screenshot paths returned; used by `implement` for UI tickets |
| `add-component` | From `design/components/<Name>/spec.md`: spec-codegen, the RN component with `.ios`/`.web` variants, one story per `tests` entry, snapshots, the watch SwiftUI subset if the spec lists watchOS | `just ds-check` passes, and stories and snapshots exist |
| `add-migration` | Schema change → `drizzle-kit generate` → expand/contract review → squawk → upgrade fixture and VM upgrade test → "migrations: yes" in the release notes | Server and nix jobs pass |
| `add-mutation` | A named, versioned mutation in `packages/domain`, golden vectors, a simulator command, the server handler, a test from an N−1 client | Simulator passes |
| `add-connector` | A connector package on [connectors-and-license.md](connectors-and-license.md)'s interface: manifest, contract kit against the fake provider, synthetic golden files, the capture scrubber | Contract kit passes; no real data |
| `add-strings` | Keys in `en.json` and `fr.json` per the glossary and style guide, then `just l10n`, then an EN/FR table in the PR | The owner approves the French |
| `test-household` | Generates or refreshes seeded fictitious data for local runs, E2E and screenshots | Same seed gives the same data |
| `fix-dependency-pr` | On a Dependabot PR: recompute the Nix pnpm hash, run `expo install --check`, and push one commit whose message contains `[dependabot skip]`, since "Dependabot will stop rebasing a pull request once extra commits have been pushed to it" and that string lets it force-push over them ([Dependabot PRs](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/managing-pull-requests-for-dependency-updates)); recompute after each rebase | `gate` passes |
| `upgrade-expo-sdk` | `just expo-scratch-prebuild` (a prebuild in a throwaway copy, so the deny rule still guards `app/`), diff it against `app/ios/`, port the native changes, including the move from CocoaPods to SwiftPM once the SDK's React Native defaults to it (Podfile, `Pods/` cache, Xcode Cloud script), run the full iOS suite | HITL: the owner tests on devices |
| `release` | Checks the release PR, the 90-day TestFlight clock, the watch checklist on paired devices ([watchos-app.md](watchos-app.md) §3.5), the ops-repo bump | HITL checklist complete |

| Hook | Where | Runs |
|---|---|---|
| `SessionStart` (Claude Code) | `.claude/settings.json` | `scripts/agent/env.sh`: devshell `PATH` and `PG*` into `CLAUDE_ENV_FILE` |
| `PostToolUse` `Edit\|Write` (Claude Code) and `apply_patch\|Edit\|Write` (Codex) | `.claude/settings.json`, `.codex/hooks.json` | `treefmt <file>`; exit 2 with errors |
| `Stop` (both) | same | `just check-fast`; one retry, then stop (`stop_hook_active`) |
| Git `pre-commit` | `lefthook.yml` | treefmt on staged files, `gitleaks git --pre-commit --staged` |
| Git `commit-msg` | `lefthook.yml` | commitlint (advisory) |

Codex runs its two hooks only while their current hashes are trusted, so the owner reviews them in `/hooks` after any change (§7).

## Setup order

1. **Owner-only (about 2 hours).**
   - Create the machine account and add it as a collaborator; mint its classic token.
   - Create the release GitHub App.
   - Enable private vulnerability reporting, approval for all outside contributors, read-only workflow tokens and the SHA-pinning policy.
   - Create the `release`, `nightly` and `testflight` environments.
   - Set up the Apple Developer Program, the App Store Connect records and Xcode Cloud's connection.
2. **Local loop.** The flake devShell and `justfile`; the pnpm workspace with a catalog and supply-chain settings; `tsconfig` bases; Biome and its GritQL plugins; treefmt; lefthook; AGENTS.md (root and nested); hooks and deny rules, after which the owner trusts the Codex hooks in `/hooks`; the `.agents/skills` symlink; MCP registrations.
3. **PR CI.** `changes`, `hygiene`, `js`, `server`, `nix` and `gate`; the rulesets with code-owner review and a `CODEOWNERS` that names the owner; `dependabot.yml` with 7-day cooldowns; CodeQL default setup without Swift; zizmor; `actions.lock`. Measure job times and cache sizes.
4. **Native.** One `expo prebuild`; commit `app/ios/`; add the watch, widget, NSE and App Intents targets; the signing xcconfig with placeholders; the `apple` job with its Xcode selection script; the Maestro smoke flow; spikes F1, F6 and F7.
5. **Release.** release-please with the App token, `draft` and `force-tag-creation`; the multi-arch image, attestation and GHCR once the server is stable (ADR 0018); the Xcode Cloud workflow on `v*`; the first internal TestFlight build.
6. **Ops repo.** The template, the host VM test, manual `just deploy`; CI deploys only after the owner decides on Pro or comin.
7. **Hardening.** Visual baselines; coverage floors; nightly simulator sweeps; Scorecard; Periphery; Swift SAST if owner question 7 asks for it.

## Costs

| Item | Monthly cost | Note |
|---|---|---|
| GitHub Actions (Linux and macOS standard runners) | $0 | Free and unlimited on public repos; 5 concurrent macOS jobs |
| GitHub Actions, private ops repo | $0 | The Free plan includes "$12 USD" of compute a month ([limits](https://docs.github.com/en/actions/reference/limits)); 2-CPU Linux runners cost $0.006 a minute ([pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing)), so about 2,000 minutes, and a weekly VM test uses a small part (estimate) |
| Actions cache, GHCR, attestations, CodeQL, secret scanning, Scorecard | $0 | 10 GB cache; public packages free ([self-hosting.md](self-hosting.md) §2.6) |
| Apple Developer Program | $99 a year | Already needed for TestFlight |
| Xcode Cloud | $0 | About 3.3 of 25 included hours a month at 8 releases (estimate) |
| Machine account | $0 | One free machine account per person |
| Cachix (optional) | $0 | 5 GB free for open source |
| Chromatic, EAS, Expo MCP doc search, Turborepo remote cache | Not used | Free tiers exist; Expo doc search is paid |
| GitHub Pro (ops repo CI deploys) | $4 a month, optional | Or manual `just deploy`, or comin |

## Open questions and spikes

**Owner decisions**
1. Agent identity: the machine account (recommended), or move the repo to a free organization. That unlocks the merge queue, fine-grained tokens for members and GitHub App bypasses, but the URL changes.
2. Strict "up to date before merge" (slower, safer) or fix-forward on `main` (recommended while there is no merge queue).
3. Does every release build TestFlight, or only app-changing ones? Either way, at least every 90 days.
4. Magic Nix Cache only, or also a public Cachix cache that self-hosters can use?
5. React Compiler on or off? If on, Biome's nursery `useReactCompiler` reports its diagnostics as warnings, and the thin ESLint layer returns only if F9 finds gaps.
6. CI deploys for the ops repo: GitHub Pro, comin, or manual only.
7. Swift SAST. CodeQL can't analyze Swift built with Xcode 27 (§5), and the watch needs Xcode 27. Either accept no Swift SAST for now, relying on SwiftLint, tests, the pinned contracts that are Swift's only link to the server (ADR 0007), and the owner's review of the `Security` module, the most sensitive Swift code (recommended, judgment), and recheck at each CodeQL release; or evaluate another scanner, none of which this note checked.

**Spikes (half a day to 2 days each)**
- **F1** Xcode Cloud archives the committed `app/ios/` from this pnpm monorepo, with the watch and extensions, through a post-clone script. Measure compute minutes. Test that the tag release-please creates through the API triggers it and the tag workflow, and that build numbers match across targets. (Part of cross-platform-ui-stack.md S4.)
- **F2** The `apple` job on `xcode-27`: clean and cached durations, peak memory on 7 GB and disk on the 14 GB SSD, the Xcode selection script against the image's next patch or RC, and React Native's precompiled iOS core (opt-in since 0.81 per [Expo](https://expo.dev/blog/precompiled-react-native-for-ios); its default in 0.88 is unverified).
- **F3** Nix on macOS runners: install plus cache restore time, against `setup-node` and Homebrew tools; the cache footprint of every job against the 10 GB budget.
- **F4** TypeScript 7 across the workspace, starting from the alias layout (§2): Expo types, `expo-router` typed routes, Storybook docgen, drizzle-kit, FormatJS extraction, knip; list which tools import `typescript`, and whether a catalog can hold the aliases.
- **F5** `expo export --platform web` inside a Nix derivation (sandbox, caches, telemetry off), then the server package and the image.
- **F6** E2E sign-in: a Debug-only test identity path that Maestro and Playwright can use without weakening Release builds.
- **F7** Storybook addon-vitest with `react-native-web-vite` and Unistyles; React Native Harness snapshot stability on the hosted simulator.
- **F8** Hash churn: how often Dependabot PRs break `fetchPnpmDeps`, and whether `fix-dependency-pr` suffices, counting the recompute after each Dependabot rebase, or self-hosted Renovate is worth it.
- **F9** The deny-by-default GritQL plugins (§3): fixtures for every miss listed there as well as false positives, `includes` scoping, cost across the full codebase; `useReactCompiler` against `eslint-config-expo`'s compiler rules; SwiftLint's Linux build on the hygiene job.
- **F10** Dropped: CodeQL has no Swift build mode on Xcode 27, so Swift SAST is owner question 7.
- **F11** `pmOnFail: ignore` yields a single-document lockfile and restores Dependabot alerts, with the devshell's pnpm 12.9 and with whichever pnpm Dependabot runs.

**Could not verify**
- KVM on `ubuntu-24.04-arm` (one third-party report says no).
- Whether a personal account's repo accepts a GitHub App as a tag-ruleset bypass actor.
- `@storybook/addon-vitest` support for `react-native-web-vite`; React Native Harness's image-snapshot API beyond cross-platform-ui-stack.md's description.
- Maestro version pinning in its install script; comin's rollback behaviour; the precompiled iOS core default in React Native 0.88.
- Whether Xcode Cloud overrides `CFBundleVersion` for every target by itself.
- Every per-job minute figure; all are estimates until spike F2.
- The Biome 2.6.6 misses of §3 come from a fact-checker's run that this revision didn't repeat; the deny-by-default plugins are unwritten (F9).
- Whether pnpm catalogs accept `npm:` aliases; which pnpm version Dependabot runs, and whether its run honours `minimumReleaseAge`.
- When GitHub introduced `cache-mode`: its page gives no date.
- Whether `useReactCompiler` reports what `eslint-config-expo`'s compiler rules report.
- Any SAST tool that analyzes Swift 6.4 built with Xcode 27.

## Sources

**Repo notes:** [factory-ci-quality.md](factory-ci-quality.md), [cross-platform-ui-stack.md](cross-platform-ui-stack.md), [design-system.md](design-system.md), [i18n-l10n.md](i18n-l10n.md), [sync-engine.md](sync-engine.md), [web-client-sync.md](web-client-sync.md), [watchos-app.md](watchos-app.md), [native-apps-auth.md](native-apps-auth.md), [connectors-and-license.md](connectors-and-license.md), [self-hosting.md](self-hosting.md), [nixos-vps-infra.md](nixos-vps-infra.md); ADRs [0001](../adr/0001-self-hosted-single-owner-instances.md), [0002](../adr/0002-one-ui-codebase-native-watch.md), [0003](../adr/0003-nixos-just-tailscale-infrastructure.md), [0007](../adr/0007-typescript-server-on-node-with-hono-drizzle-and-postgresql.md), [0011](../adr/0011-passkeys-on-the-instance-sign-in-page.md), [0018](../adr/0018-forward-only-upgrades-with-a-client-support-window.md), [0019](../adr/0019-native-apps-through-the-testflight-internal-group-only.md).

**GitHub:** [hosted runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners), [limits](https://docs.github.com/en/actions/reference/limits), [runner pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing), [Xcode 27 image GA](https://github.blog/changelog/2026-10-15-xcode-27-runner-image-generally-available/), [macos-26 readme](https://github.com/actions/runner-images/blob/main/images/macos/macos-26-arm64-Readme.md), [dependency caching](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching), [personal repo permission levels](https://docs.github.com/en/account-and-profile/reference/permission-levels-for-a-personal-account-repository), [Dependabot PRs](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/managing-pull-requests-for-dependency-updates), [CodeQL 2.27.3](https://github.com/github/codeql-cli-binaries/releases/tag/v2.27.3), [CodeQL build options](https://docs.github.com/en/code-security/reference/code-scanning/codeql/build-options-for-compiled-languages), [KVM changelog](https://github.blog/changelog/2024-04-02-github-actions-hardware-accelerated-android-virtualization-now-available/), [runner-images](https://github.com/actions/runner-images), [xcode-27 readme](https://github.com/actions/runner-images/blob/main/images/macos/xcode-27-arm64-Readme.md), [#14404](https://github.com/actions/runner-images/issues/14404), [secure use](https://docs.github.com/en/actions/reference/security/secure-use), [using secrets](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets), [workflow lockfiles](https://docs.github.com/en/actions/concepts/security/workflow-dependency-lockfiles), [ruleset rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets), [creating rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository), [Dependabot ecosystems](https://docs.github.com/en/code-security/dependabot/ecosystems-supported-by-dependabot/supported-ecosystems-and-repositories), [Dependabot options](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/dependabot-options-reference), [CodeQL languages](https://codeql.github.com/docs/codeql-overview/supported-languages-and-frameworks/), [CodeQL default setup](https://docs.github.com/en/code-security/code-scanning/enabling-code-scanning/configuring-default-setup-for-code-scanning), [private vulnerability reporting](https://docs.github.com/en/code-security/security-advisories/working-with-repository-security-advisories/configuring-private-vulnerability-reporting-for-a-repository), [personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens), [terms of service](https://docs.github.com/en/site-policy/github-terms/github-terms-of-service), [Scorecard action](https://github.com/ossf/scorecard-action).

**Expo and React Native:** [monorepos](https://docs.expo.dev/guides/monorepos/), [SDK 58](https://expo.dev/changelog/sdk-58), [SDK versions](https://docs.expo.dev/versions/latest/), [unit testing](https://docs.expo.dev/develop/unit-testing/), [React Compiler](https://docs.expo.dev/guides/react-compiler/), [Expo MCP](https://docs.expo.dev/eas/ai/mcp/), [pricing](https://expo.dev/pricing), [precompiled iOS](https://expo.dev/blog/precompiled-react-native-for-ios), [expo@58 tsconfig.base.json](https://unpkg.com/expo@58.0.23/tsconfig.base.json), [React Native 0.88](https://reactnative.dev/blog/2026/10/19/react-native-0.88), [RNTL v14 migration](https://oss.callstack.com/react-native-testing-library/docs/start/migration-v14), [React Native Harness](https://react-native-harness.dev/), [Detox](https://github.com/wix/Detox) and [its npm record](https://registry.npmjs.org/detox), [react-native-macos on npm](https://registry.npmjs.org/react-native-macos), React Native [layout props](https://reactnative.dev/docs/layout-props) and [view style props](https://reactnative.dev/docs/view-style-props), [Maestro install](https://docs.maestro.dev/maestro-cli/how-to-install-maestro-cli), [Maestro releases](https://github.com/mobile-dev-inc/Maestro/releases), secondary: [apple-targets #147](https://github.com/EvanBacon/expo-apple-targets/issues/147).

**JavaScript tooling:** [Node schedule](https://github.com/nodejs/Release/blob/main/schedule.json), [Node releases](https://nodejs.org/dist/index.json), pnpm [lockfile](https://pnpm.io/lockfile), [resolution settings](https://pnpm.io/settings/dependency-resolution), [build settings](https://pnpm.io/settings/build), [CLI settings](https://pnpm.io/settings/cli), [TypeScript 7.0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/), [typescript-eslint versions](https://typescript-eslint.io/users/dependency-versions/), [eslint-config-expo 58.0.4](https://registry.npmjs.org/eslint-config-expo/58.0.4), [@typescript-eslint/parser](https://registry.npmjs.org/@typescript-eslint/parser/latest), [pnpm catalogs](https://pnpm.io/catalogs), [@tsconfig/strictest](https://unpkg.com/@tsconfig/strictest@2.0.8/tsconfig.json), Biome [plugins](https://biomejs.dev/linter/plugins/), [language support](https://biomejs.dev/internals/language-support/), [domains](https://biomejs.dev/linter/domains/), [noReactNativeLiteralColors](https://biomejs.dev/linter/rules/no-react-native-literal-colors/) and [its source at 2.6.6](https://github.com/biomejs/biome/blob/%40biomejs/biome%402.6.6/crates/biome_js_analyze/src/lint/style/no_react_native_literal_colors.rs), [useReactCompiler](https://biomejs.dev/linter/rules/use-react-compiler/), [eslint-plugin-react-native's stylesheet helper](https://github.com/Intellicode/eslint-plugin-react-native/blob/master/lib/util/stylesheet.js), [noFloatingPromises](https://biomejs.dev/linter/rules/no-floating-promises/), [Oxlint JS plugins](https://oxc.rs/docs/guide/usage/linter/js-plugins.html), fast-check [race conditions](https://fast-check.dev/docs/advanced/race-conditions/) and [model-based testing](https://fast-check.dev/docs/advanced/model-based-testing/), Storybook [test runner](https://storybook.js.org/docs/writing-tests/integrations/test-runner) and [Vitest addon](https://storybook.js.org/docs/writing-tests/integrations/vitest-addon), [Chromatic pricing](https://www.chromatic.com/pricing), [release-please action](https://github.com/googleapis/release-please-action), [release-please customizing](https://github.com/googleapis/release-please/blob/main/docs/customizing.md), [release-please manifest options](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md), [Vercel remote cache](https://vercel.com/changelog/free-vercel-remote-cache); versions from the [npm registry](https://registry.npmjs.org/) on 2026-11-07.

**Nix and deploy:** [Magic Nix Cache](https://github.com/DeterminateSystems/magic-nix-cache-action), [Cachix pricing](https://www.cachix.org/pricing), [install-nix-action](https://github.com/cachix/install-nix-action), [statix](https://github.com/oppiliappan/statix), [deadnix](https://github.com/astro/deadnix), [nixfmt](https://github.com/NixOS/nixfmt), [treefmt](https://github.com/numtide/treefmt), [nixpkgs JavaScript docs](https://github.com/NixOS/nixpkgs/blob/master/doc/languages-frameworks/javascript.section.md), [deploy-rs](https://github.com/serokell/deploy-rs), [comin](https://github.com/nlewo/comin), [Tailscale action](https://github.com/tailscale/github-action), Renovate [nix manager](https://docs.renovatebot.com/modules/manager/nix/) and [postUpgradeTasks](https://docs.renovatebot.com/configuration-options/#postupgradetasks), [squawk](https://github.com/sbdchd/squawk), [gitleaks](https://github.com/gitleaks/gitleaks), [zizmor audits](https://docs.zizmor.sh/audits/), [actionlint](https://github.com/rhysd/actionlint); measured: `nix eval` of nixpkgs-unstable and nixos-26.05 attributes on 2026-11-07, rechecked against [nixos-26.05's all-packages.nix](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/pkgs/top-level/all-packages.nix) and the pnpm and Maestro package files of both branches.

**Agents:** Claude Code [hooks](https://code.claude.com/docs/en/hooks) and [skills](https://code.claude.com/docs/en/skills), Codex [hooks](https://learn.chatgpt.com/docs/hooks) and [skills](https://learn.chatgpt.com/docs/build-skills) (redirected from developers.openai.com), [MobileBuildMCP](https://github.com/getsentry/XcodeBuildMCP), [agent-device](https://www.npmjs.com/package/agent-device), [Playwright MCP](https://github.com/microsoft/playwright-mcp), [Storybook MCP addon](https://www.npmjs.com/package/@storybook/addon-mcp); the installed skills' own SKILL.md files (wayfinder, to-spec, to-tickets, triage, implement, tdd, code-review, ship).
