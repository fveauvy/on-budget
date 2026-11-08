# How T3 Code enforces code quality, and what Shoestring should take

> Research note, 2026-11-08, for [#8](https://github.com/fveauvy/shoestring/issues/8) on map [#6](https://github.com/fveauvy/shoestring/issues/6), feeding the quality-gates grilling [#16](https://github.com/fveauvy/shoestring/issues/16). It is background: `PRODUCT.md`, the ADRs, `SPEC.md`, [`docs/factory.md`](../factory.md) and [`docs/architecture.md`](../architecture.md) win until the map's final docs PR. The charting decisions ([#7](https://github.com/fveauvy/shoestring/issues/7)) already fix two points: Vite+ replaces Biome, and CodeQL stays. T3 Code was read at commit [`6417151`](https://github.com/pingdotgg/t3code/tree/6417151c093e54dbba0e6c824c69d03983793b9a) (2026-11-08) from a shallow clone and GitHub's API, never installed or run. Paths in backticks are T3 Code's at that commit, linked under [Sources](#sources). Versions come from npm dist-tags and GitHub releases read the same day. **H**, **M** and **L** mark confidence; "measured" marks numbers read from T3 Code's Actions API, "estimate" a computed figure, "judgment" my own assessment.

## Question

Which quality gates does pingdotgg/t3code run, how is each configured and enforced, what does each cost, and which should Shoestring adopt? The ticket asks for every gate: Vite+ (Oxlint with `vite.config.ts` and `oxlint-plugin-t3code`, Oxfmt, Vitest with browser mode), the `tsgo` typecheck and knip, pnpm settings, git hooks, every CI workflow, AI review, third-party licenses, the remote task cache and the agent setup. Then it asks to map them onto Shoestring: decide whether `just` and the Nix devShell stay next to Vite+, and add gitleaks with the IBAN rule, the personal-terms check, NixOS VM tests and Swift for the watch.

## Answer

1. **Adopt T3 Code's JavaScript core and keep Shoestring's safety rails around it.**
   - Vite+ 1.5.0, through its project-local CLI, formats, lints, tests and runs the staged-file checks.
   - `tsc -b` on TypeScript 7.0.2 stays the typecheck.
   - knip 6.44.2 becomes a blocking check.
   - CodeQL, gitleaks, the personal-terms check, zizmor, actionlint, `actions.lock` and the rulesets stay. T3 Code's repository has no gitleaks, zizmor, actionlint or SHA lockfile, and no visible CodeQL setup.

   **M-H**
2. **The Nix devShell and `just` stay; lefthook, treefmt and Biome go.**
   - Vite+ manages Node.js and pnpm only from its global CLI, which Shoestring doesn't install. The local package "requires an existing Node.js runtime and package manager", and the devShell supplies both, plus every tool npm doesn't carry.
   - `just` stays the one entry point across languages, and the CI parity of `just ci-<job>`. It calls `vp` for JavaScript work.
   - `.vite-hooks/pre-commit` running `vp staged` replaces lefthook. `vp fmt`, with nixfmt and SwiftFormat called directly, replaces treefmt.

   **M**
3. **Port T3 Code's rules, not its whole configuration.**
   - Take its lint discipline: unused suppressions as errors, a reason on every suppression, and `no-restricted-imports` for the dependency direction.
   - Take its test-hygiene rules: no wall clock, no real-time waits, no polling loops, no per-test timeouts or retries, no `.only` in CI.
   - Take its Hermes API ban, its React Compiler rules, and a test that compiles every client file and fails on any compiler bailout.
   - Take `@shadcn/lint` for the web kit.
   - Make every rule an error or turn it off, except the experimental React Compiler rules, which warn until they prove reliable. T3 Code leaves three whole categories at warning level, and its CI never fails on them.

   **M**
4. **CI keeps its shape, with T3 Code's refinements.**
   - `gate` accepts a skipped job only when `changes` said that job wasn't needed, because GitHub counts a job skipped by a condition as a pass.
   - Change detection fails open.
   - The local fast check reuses CI's path mapping.
   - GitHub-hosted runners stay, free on a public repository. T3 Code pays Blacksmith about $0.29–0.35 per full run (estimate).

   **H** on the facts, **M** on the fit
5. **Skip, adapt or defer the rest.**
   - Skip:
     - the PR size and vouch labels: zizmor flags their `pull_request_target` and `issue_comment` triggers, and `factory.md` bans the first;
     - Windows tests;
     - Macroscope: paid per kilobyte of diff, with automatic approval;
     - the remote task cache: T3 Code's own Cloudflare service, and Vite+ hasn't shipped remote caching;
     - `.pnpmfile.cjs`, `.cursor/` and `.repos/`.
   - Adapt the agent context budget into byte budgets for `AGENTS.md`, and the license notices with ADR 0013's allowlist. Take CodeRabbit as a free, advisory reviewer, and `t3.json` for worktree setup.
   - Defer the Expo fingerprint check, which only matters with over-the-air updates ([#18](https://github.com/fveauvy/shoestring/issues/18)), and Effect's diagnostics through `@effect/tsgo` ([#21](https://github.com/fveauvy/shoestring/issues/21)).

   **M**
6. **The version rule holds at the release level; #16 should accept its caveats explicitly.**
   - Vite+ 1.5.0 is a stable release, but not every part is stable upstream:
     - Oxfmt, bundled as 0.76.0, is still in beta;
     - Oxlint's JS plugins, which custom rules need, are in alpha;
     - Oxlint's React Compiler rules are experimental.
   - T3 Code's typecheck runs `tsrs` 0.23.0, a Rust port of the TypeScript checker built on TypeScript 7.1 beta, which the rule excludes.

   **H**

## 1. T3 Code's chain at a glance

| Stage | What runs |
|---|---|
| Editor and agents | No agent hooks. `AGENTS.md` asks agents to run `vp run check:changed` before pushing and forbids repo-wide checks. `.vscode/settings.json` formats with the Oxc extension |
| Commit | `.vite-hooks/pre-commit` runs `vp staged`: `vp fmt`, then `vp lint`, on staged files, and knip on the whole repository when code or a manifest is staged |
| Pull request | `ci.yml` runs lint and format, knip, the build and typecheck, sharded tests, Rust, a macOS native lint and a release smoke test, ending in `Check`. Side workflows label size, author trust and native-fingerprint changes, and check the agent context budget. CodeRabbit and Macroscope review |
| Merge | The ruleset "Passing CI on main" requires `Check` plus seven older job names, and a pull request with no required approval, merged by squash or rebase. Nothing else is required |

The ruleset was read through GitHub's API (`rulesets/13856153`); its bypass actors are visible only to admins. **H**

## 2. The gates

### 2.1 Vite+: format, lint, test and staged checks

Vite+ "brings together Vite, Vitest, Oxlint, Oxfmt, Rolldown, tsdown, and Vite Task in a single vite-plus package" ([guide](https://viteplus.dev/guide/)). Version 1.0, released 2026-09-28, is "stable, MIT licensed" ([announcement](https://voidzero.dev/posts/announcing-vite-plus-1-0)). T3 Code pins `vite-plus` 1.4.0 in its pnpm catalog and aliases `vite` to `npm:@voidzero-dev/vite-plus-core@1.4.0` (`pnpm-workspace.yaml`). One root `vite.config.ts` of 675 lines holds the `run`, `test`, `staged`, `fmt` and `lint` blocks.

| Gate | What it checks | Configuration | Where it blocks | Cost (measured) |
|---|---|---|---|---|
| Format: Oxfmt through `vp fmt` and `vp check` | JavaScript, TypeScript, JSON, CSS, Markdown, YAML and more ([Oxfmt](https://oxc.rs/docs/guide/usage/formatter.html)) | `fmt` block: ignores generated, vendored and native paths and the lockfile; `sortPackageJson`; skips `.macroscope/ignore.md`, because "Markdown formatting corrupts it" | Pre-commit, then CI's `Lint` job → `Check`; `vp check` formats every file, docs included | `Lint` takes 35–37 s on an 8-vCPU runner, knip and codegen checks included |
| Lint: Oxlint through `vp lint` and `vp check` | Built-in plugins `eslint`, `oxc`, `react`, `unicorn` and `typescript`, plus two JS plugins | `correctness`, `suspicious` and `perf` at warning level; 74 rules set at the root, 39 to error and 35 off; path `overrides` add more; `reportUnusedDisableDirectives: "error"`; `typeAware: false`, `typeCheck: false` | Same as format. `vp check` fails on errors only; warnings fail only with `--deny-warnings` ([check](https://viteplus.dev/guide/check)), which T3 Code doesn't pass | In `Lint` |
| Boundaries: `eslint/no-restricted-imports` | Import paths between packages | Explicit `client-runtime` subpaths. No React, Expo or server code in shared forge definitions. No client entry in server code. Shared UI exports instead of borrowed class recipes | Same as lint | — |
| Custom rules: `oxlint-plugin-t3code` | 33 rules, 30 with tests: test hygiene (9), UI and copy (11), Effect and platform (6), React (3), security boundaries (2), comments and suppressions (2) | `jsPlugins: ["./oxlint-plugin-t3code/index.ts", "@shadcn/lint"]`, scoped by `overrides` | Same as lint | — |
| Design-system rules: `@shadcn/lint` 0.1.5 | Tailwind classes in `apps/web/src`: `no-raw-colors`, `no-arbitrary-values` with an allowlist, `require-static-classes`, `no-restyle` (only layout classes on `components/ui` exports), `no-unknown-classes` | `settings.shadcn.ui: "~/components/ui"` | Same as lint | — |
| React Compiler | Oxlint's React Compiler rules (`react/purity`, `refs`, `immutability`, `set-state-in-effect` and others) as errors, plus `t3code/no-manual-memo` and `no-hookless-use-function`. A test (`apps/web/src/reactCompilerCoverage.test.ts`) compiles every web source file with `oxc-transform-react` and fails on any bailout, which lint misses ("try/finally, throw inside try"). Mobile has the same test | Lint rules in `overrides`; the test in the web and mobile suites | Lint; `Test Web` and `Test` → `Check` | — |
| Tests: Vitest through `vp test` | Unit and integration tests in every package | Root `test`: `retry: 0` ("Flaky tests are fixed, not retried"), `allowOnly: !process.env.CI`, 60 s hook and test timeouts, a full set of fake timers. Server: `fileParallelism: false`, 8 shards weighted by recorded durations. Mobile: a `react` project on jsdom, with React Native mocked and Expo's Babel preset compiling with the React Compiler | `Test`, `Test Web`, `Test Server 1–8` → `Check` | `Test` 116–125 s and `Test Web` 174–213 s on 8 vCPUs; each server shard 91–174 s on 4 vCPUs |
| Browser mode | Web component tests in headless Chromium | `apps/web/vite.config.ts`: a `browser` project for `*.browser.test.{ts,tsx}`, the Playwright provider (`@vitest/browser-playwright` 5.0.5, Playwright 1.60.0), Chromium only; renders go through a disposable helper that `t3code/require-disposable-render` enforces | `Test Web` → `Check` | In `Test Web` |
| Staged checks: `vp staged` | Formats then lints staged files; runs knip when code or a manifest is staged (§2.4) | `staged` block | Pre-commit | Not measured |

- **Browser mode is stable:** Vitest 4 removed its experimental tag ([Vitest 4](https://vitest.dev/blog/vitest-4)), and Vite+ 1.5.0 bundles Vitest 5.1.0 ([release](https://github.com/voidzero-dev/vite-plus/releases/tag/v1.5.0)). **H**
- **The React Compiler rules are experimental:** Oxlint "added all recommended rules to Oxlint's correctness category" on 2026-08-18 ([post](https://oxc.rs/blog/2026-08-18-react-compiler-support)), but its plugin page says they "are experimental and off by default" ([plugins](https://oxc.rs/docs/guide/usage/linter/plugins.html)). Vite+ 1.5.0's notes list a false positive of `react/immutability` in Oxlint 1.91.0 (oxc#28401). **H**
- **JS plugins are in alpha:** custom rules run as Oxlint JS plugins, which "are currently in alpha" ([JS plugins](https://oxc.rs/docs/guide/usage/linter/js-plugins.html), [alpha post](https://oxc.rs/blog/2026-03-11-oxlint-js-plugins-alpha)).
  - Vite+ re-exports the plugin API as `vite-plus/lint/plugins` and `RuleTester` as `vite-plus/lint/plugins-dev`.
  - Its docs advise against a separate `@oxlint/plugins` dependency, which "can drift from the linter that loads your plugin" ([lint](https://viteplus.dev/guide/lint)). T3 Code's plugin still declares one.

  **H**
- **Port, don't copy:** 15 of the 33 rule files, and the shared `utils.ts`, import Effect modules. The test rules also target Effect's `TestClock`, `Schedule` and `yieldNow` as well as plain timers. Outside an Effect codebase they are rewritten, not copied. **H**

### 2.2 Typecheck and unused code

| Gate | What it checks | Configuration | Where it blocks | Cost (measured) |
|---|---|---|---|---|
| Typecheck: `@effect/tsgo` 0.67.1 | TypeScript errors, and Effect Language Service diagnostics: `tsconfig.effect.json` sets 86 of them to error, such as global `Date`, `fetch`, `console` and `JSON` inside Effect code | `vp run -r --cache --concurrency-limit 4 typecheck`; every package runs `effect-tsgo tsrs --noEmit`. `tsconfig.base.json` is `strict` plus `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `erasableSyntaxOnly` and `verbatimModuleSyntax`. "There is no `typescript` package to run" (`AGENTS.md`) | CI's `Build and Typecheck`, in the background beside the desktop build → `Check`; not at commit | 53–62 s for the whole job, with the remote cache |
| Unused code: knip 6.34.0 | Unused files, dependencies and exports | `knip.jsonc`: entries per workspace; `types` and `nsTypes` off ("Exported types are part of our contracts"); `includeEntryExports` for private packages; a preprocessor that recognizes Effect schemas through TypeScript 6's compiler API (`typescript-legacy`, `npm:typescript@~6.0.3`); two passes run side by side as a Vite+ task | Pre-commit, on any code or manifest change, and CI's `Lint` → `Check` | In `Lint` |

- **What `tsrs` is:** the Effect-patched build of `tsrs`, a "Rust port of the TypeScript 7 type checker" started 2026-10-01. Its npm `latest` is `0.23.0-ts7.1.0-beta`, and its README sets "a stable version against TypeScript 7.1" as a goal ([tsrs](https://github.com/oxc-project/tsrs)). `@effect/tsgo` also wraps the stock checker: `effect-tsgo tsc` supports TypeScript 7.0.2 ([Effect-TS/tsgo](https://github.com/Effect-TS/tsgo)). **H**
- **Knip no longer needs TypeScript:** 6.44.2 depends on `oxc-parser` and `oxc-resolver` and on no `typescript` package (npm manifest). Only T3 Code's own preprocessor needs the TypeScript 6 API. **H**

### 2.3 pnpm settings

T3 Code pins pnpm 12.14.0 and Node 26.12.1 in `package.json`. These settings in `pnpm-workspace.yaml` act at every install, CI included, at no measurable cost. **H**

- **Catalog:** 63 shared versions. `vite` resolves to `vite-plus-core`, so every package shares one copy.
- **Overrides:** 83.
  - They collapse duplicate copies (Tailwind, `lightningcss`, `effect`, `ws`, `undici`) and cut unused edges with `"-"`.
  - They pin every patched package "so lockfile-free release installs cannot drift past the patch".
- **Patches:** 35 `patchedDependencies`, eight of them with a comment saying why.
- **`strictPeerDependencies: true`:** "Unmet peers fail resolution". pnpm's default is false ([peers](https://pnpm.io/settings/peer-dependencies)).
- **Build scripts:** 13 `allowBuilds` entries. pnpm's `strictDepBuilds`, on by default, fails an install that finds an unreviewed build script ([build](https://pnpm.io/settings/build)).
- **Agent skills:** `permissions` gives `@tanstack/router-plugin`, `drizzle-kit` and `expo-sqlite` `skills: false`. Since 12.11.0, pnpm can link the agent skills a dependency ships into the project's skill directories ([build](https://pnpm.io/settings/build)).
- **Release age:** T3 Code leaves `minimumReleaseAge` unset, so pnpm's built-in 1,440 minutes apply.
  - The built-in default is not strict: when no version in range is old enough, pnpm installs a younger one. Only an explicit setting turns strict mode on ([resolution](https://pnpm.io/settings/dependency-resolution)).
  - T3 Code exempts `@effect/tsgo` 0.67.1 and its seven platform packages.
  - `trustPolicy` stays at its default, off.
- **`.pnpmfile.cjs`:** on Windows only, it adds Linux x64 glibc to `supportedArchitectures` "because WSL shares node_modules".

### 2.4 Git hooks

- **The hook:** `.vite-hooks/pre-commit` holds one line, `vp staged`.
- **Installation:** the `prepare` script runs `vp config --no-agent` at every install. It installs Vite+'s hook dispatcher and leaves agent instruction files untouched ([commit hooks](https://viteplus.dev/guide/commit-hooks)).
- **The checks:** `vp staged` bundles lint-staged 17. The `staged` block holds two entries:
  - `"*"` runs `vp fmt`, then `vp lint`, "in order, so lint reads the formatted file. CI lint failures cost a full round trip";
  - code files, `package.json` and `knip.jsonc` run `vp run knip:check`, which takes no file list, since "any code or manifest change can leave a file, export, or dependency unused".
- **Bypass:** the hooks are a convenience. `VP_GIT_HOOKS=0`, `HUSKY=0` or an init script under `~/.config/vite-plus/` turns them off ([commit hooks](https://viteplus.dev/guide/commit-hooks)). CI is the boundary. **H**

### 2.5 CI workflows

**`ci.yml`** runs on `pull_request` and on pushes to `main`, cancels superseded PR runs, and uses Blacksmith runners. Its top-level permission is `contents: read`; jobs that use the task cache add `id-token: write`.

| Job | Does | Runs when |
|---|---|---|
| `Changes` | Sparse checkout of the manifests. `.github/scripts/ci-changes.cjs` maps changed files to packages and their dependents. "Detection fails open: if the diff cannot be resolved, everything runs". `main` runs every Linux job | Always |
| `Lint` | Rejects committed PR evidence (`.github/pr-assets`), then runs knip, the generated-GraphQL checks and `vp check` | Always |
| `Build and Typecheck` | Desktop build, preload-bundle check, a boot of the production web build in Chromium, and the typecheck in parallel | Build or typecheck inputs changed |
| `Test`, `Test Web`, `Test Server 1–8` | Vitest suites; the server shards gate themselves after checkout | Affected packages |
| `Rust` | `cargo fmt --check` and `cargo test --locked` for three crates | Rust paths |
| `Mobile Native Static Analysis` | SwiftLint `--strict`, ktlint and detekt, installed unpinned from Homebrew, on a macOS runner ("~6.7x a Linux minute") | Native mobile sources |
| `Release Smoke` | `scripts/release-smoke.ts` exercises the release-only workflow steps | Release inputs |
| `Check` | `if: always()`; needs every job. A job "may skip only when `changes` said every gate it represents was not needed", tested with jq over `toJSON(needs)` | Always; the required check |

GitHub reports a job skipped by a condition as a success, so a skipped required job never blocks a merge ([required checks](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/collaborating-on-repositories-with-code-quality-features/troubleshooting-required-status-checks), [conditions](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-jobs-with-conditions)). `Check`'s jq test closes that gap. **H**

**Side workflows:**

| Workflow | Trigger | What it checks | Where it blocks | Cost (measured) |
|---|---|---|---|---|
| `pr-size.yml` | `pull_request_target` | Labels `size:XS` to `size:XXL` by changed lines. Whitespace is ignored, and tests are left out of mixed PRs | Never: it only labels | 42–55 s, GitHub-hosted |
| `pr-vouch.yml` | `pull_request_target`, `issue_comment` (`/recheck-vouch`), pushes that change `.github/VOUCHED.td` | `mitchellh/vouch/action/check-user@v1` labels the author `vouch:trusted`, `unvouched` or `denounced`. `CONTRIBUTING.md`: "These are context, not eligibility rules" | Never | 17–21 s |
| `agent-context-budget.yml` | `pull_request`, label changes included | Fails when a value in `agentContextBudget.json` grows past `main`'s, unless a maintainer adds `agent-context-increase`. A server test keeps the file equal to the measured context: Claude system append 5,350 B, Codex additional context 7,080 B, MCP tool list 48,116 B | A failing check the ruleset doesn't require | 7–11 s |
| `windows-tests.yml` | `workflow_dispatch` only | Gives contributors a cloud Windows machine: "nothing in the suite passes on Windows yet" | Never | 86–673 s on 8-vCPU Windows |
| `mobile-fingerprint-check.yml` and `mobile-fingerprint-label.yml` | `pull_request`, then a trusted `workflow_run` from `main` | Compares `expo-updates fingerprint:generate` for iOS and Android at the merge result and at its base. The trusted job applies "📱 Native Change", so native changes can be batched before a store build and `main` stays updatable over the air | Never: "The check is advisory: it always passes" | 21–67 s, median 24 s |

- **Release reruns:** `release.yml` reruns `vp check`, the typecheck and the tests on the release commit before it builds ("Release quality checks"). Delivery is [#9](https://github.com/fveauvy/shoestring/issues/9)'s subject.
- **zizmor:** its `dangerous-triggers` audit flags `pull_request_target`, `workflow_run` and `issue_comment` ([zizmor](https://docs.zizmor.sh/audits/#dangerous-triggers)), so it would flag the size, vouch and fingerprint-label workflows, among others. **H**

### 2.6 AI review

**CodeRabbit:**
- **Configuration:** `.coderabbit.config.ts`, which CodeRabbit evaluates server-side; the `@coderabbitai/config` SDK only adds types ([TypeScript configuration](https://docs.coderabbit.ai/configuration/typescript-configuration)).
  - Automatic reviews, with no high-level summary or status comment, and `request_changes_workflow: false`.
  - Path instructions hold changed code to `docs/internals/effect-services.md`, `web-ui.md` and `testing.md`.
  - Authors who aren't an owner, member or collaborator can't approve their own PRs or override pre-merge checks.
- **Where it blocks:** it posts a `CodeRabbit` check that the ruleset doesn't require. Its pre-merge checks block a merge only "when paired with Automatic Approvals" ([pre-merge checks](https://docs.coderabbit.ai/pr-reviews/pre-merge-checks)).
- **Cost:** the Open source plan gives "Unlimited public repositories" and "Team features", with rate limits that vary with popularity. "For public repositories with less than 10 stars, CodeRabbit requires reviews to be triggered manually" ([plans](https://docs.coderabbit.ai/management/plans)).

**H**

**Macroscope:**
- **Correctness review:** on PRs. `.macroscope/ignore.md` replaces Macroscope's default ignore list.
- **Approvability:** it decides whether Macroscope may approve a PR without human review ([approvability](https://docs.macroscope.com/approvability)). On the one PR checked (PR 32926) it posted a passing `Approvability Check` run. `.macroscope/approvability.md` holds two kinds of PR back for a human: any PR that "changes product defaults", and any that "adds or broadens a directive that disables or suppresses a lint, type-checker, LSP, or other static-analysis diagnostic".
- **Check Run Agents:** three Markdown agents in `.macroscope/check-run-agents/`: Effect service conventions, test signal and UI consistency.
  - Each sets `conclusion: neutral` and `requires: [Check]`, with budgets per run and per PR of $5/$25, $1/$5 and $2/$10.
  - A `neutral` agent can't fail. `conclusion: failure` "is not supported with `input: incremental`" ([configuration](https://docs.macroscope.com/check-run-agents/configuration)).
- **Where it blocks:** nowhere the ruleset requires.
- **Cost:** usage only ([pricing](https://docs.macroscope.com/pricing)).
  - Reviews cost $0.025–0.20 per KB of diff: $0.05 in the default mode, with a $0.50 minimum per review.
  - Agents cost the model's price plus 5%, and approvability $0.30 per eligibility run.
  - A new workspace gets $100 of credit, then 1,000 free agent credits ($10) a month.

**H**

### 2.7 Third-party licenses and the remote task cache

**License notices** (`scripts/lib/third-party-licenses.ts`, `docs/internals/open-source-licenses.md`):
- **What it collects:** a Vite plugin in the web build collects the production and optional dependencies of the web, server and desktop packages, and cross-checks the module ids actually bundled. It emits `third-party-licenses.json`, which the Settings page reads. Metro generates the mobile equivalent.
- **When it fails:** "The build fails when a collected package has no distributable license identifier or contains no license or notice text".
- **No allowlist:** only `UNLICENSED` and `proprietary` are rejected outright.
- **Configuration:** the SPDX templates (license list v3.28.0) are committed, so builds never fetch them. `third-party-licenses.config.json` holds 19 custom notices and 83 package overrides.
- **Where it blocks:** the web build, so `Build and Typecheck` → `Check`.

**H**

**Remote task cache** (`.task-cache.json`, `infra/task-cache/README.md`):
- **The client:** Vite+'s native remote client, which ships undocumented: "Vite+ 1.3 includes an experimental remote client even though its TypeScript configuration and CLI help omit it". T3 Code switches it on through a Node preload hook.
- **The server:** T3 Code's own Cloudflare Worker with D1 and R2, deployed with Alchemy, plus Axiom telemetry. It keeps results for 7 days within 64 GB.
- **Access:** CI holds a read token. Same-repository PRs to `main` and pushes to `main` read and write. Forks get nothing.
- **Role:** it isn't a gate, but it is why T3 Code's typecheck and lint take under a minute.
- **Vite+'s own position:** remote caching is among the features "planned for future versions" ([announcement](https://voidzero.dev/posts/announcing-vite-plus-1-0)), and reusing Vite Task's cache through GitHub Actions is "experimental" ([guide](https://viteplus.dev/guide/github-actions-cache)).

**H**

### 2.8 Agent setup

- **`AGENTS.md`:** 24,670 bytes; `CLAUDE.md` is `@AGENTS.md`. It holds:
  - the product's non-negotiables;
  - "The three ways to hurt yourself": killing processes by pattern, writing to the live install, baking origins into dev builds;
  - a "Hit every surface" checklist: entry points, clients, providers, agents, contracts, reverse states, connection modes, docs;
  - a Verifying section: `vp run check:changed` before pushing, and "Do not run repo-wide checks… CI owns the full suite";
  - what to test and what not to: a failing test first for logic bugs, before and after captures for visual ones;
  - test rules that lint enforces, mirrored in `docs/internals/testing.md` "for review bots";
  - PR rules: Conventional titles, before and after images.
- **`.agents/skills/`:** `test-t3-app` and `test-t3-mobile` drive T3 Code's own browser and device panels; `contribution-triage` handles outside PRs. Each skill has an `agents/openai.yaml` giving Codex a display name, a short description and a default prompt. `.claude/skills` is a committed symlink to `.agents/skills`.
- **`.cursor/rules/cursor-cloud.mdc`:** `alwaysApply` notes for Cursor's cloud VMs only, "kept out of the shared `AGENTS.md`".
- **`t3.json`:**
  - It defines a "Dev" script and a "Setup Worktree" script with `runOnWorktreeCreate: true` ([schema](https://t3.codes/schema/t3.json)).
  - The setup script writes `.t3/setup-running`, then `setup-done` or `setup-failed`, and `AGENTS.md` tells agents to wait for those markers.
- **`.repos/`:** read-only copies of the Effect and Alchemy sources, about 231 MB, for agents to read; `AGENTS.md` says never to import from them.
- **No agent hooks:** there is no `.claude/settings.json` and no Codex hooks file. Instructions, `vp staged` and CI do the enforcing.

**H**

### 2.9 What T3 Code doesn't run

- **Security tooling:** no secret scanner in the repository, no Dependabot or Renovate configuration, no zizmor and no actionlint. There is no CodeQL workflow either; whether default setup is on can't be seen without admin access.
- **Action pinning:** most actions are pinned by tag, such as `actions/checkout@v6`. Thirteen `uses:` lines in eleven workflows still reference `voidzero-dev/setup-vp@v1`, a tag Vite+'s docs say "no longer receives updates" ([CI](https://viteplus.dev/guide/ci)). Two more pin it by SHA.
- **Risky triggers:** five workflows run on `pull_request_target`: the size and vouch labelers, plus three preview workflows. Three run on `workflow_run`.

**H**

## 3. Costs

- **Wall time:** the 40 latest completed `ci.yml` runs took 1 min 37 s to 4 min 15 s, mostly 3–4 minutes (measured).
- **One full run on `main`** ([run 42478610884](https://github.com/pingdotgg/t3code/actions/runs/42478610884)):

  | Runner | Jobs | Runner time |
  |---|---|---|
  | 8 vCPU | `Lint` 37 s, `Build and Typecheck` 53 s, `Test` 125 s, `Test Web` 174 s, `Release Smoke` 29 s | 7.0 min |
  | 4 vCPU | eight server shards at 146–174 s each, `Rust` 38 s | 21.7 min |
  | 2 vCPU | `Changes` 5 s, `Check` 10 s | 0.25 min |
- **A PR run** ([run 42482286835](https://github.com/pingdotgg/t3code/actions/runs/42482286835)) shows the same spread.
- **Blacksmith's list prices** ([pricing](https://www.blacksmith.sh/pricing)):

  | Runner | Price per minute |
  |---|---|
  | Ubuntu x64, 2 vCPU | $0.004 |
  | Ubuntu x64, 4 vCPU | $0.008 |
  | Ubuntu x64, 8 vCPU | $0.016 |
  | Windows, 8 vCPU | $0.032 |
  | macOS M4 | $0.08 |

  The first 3,000 minutes of each month are free.
- **Per full run:** about $0.29, or $0.35 if each job is billed in whole minutes (estimate). The side workflows run on GitHub-hosted `ubuntu-24.04`, except the fingerprint check and the Windows tests, which run on Blacksmith.
- **AI review:** CodeRabbit is free for public repositories; Macroscope bills by diff size (§2.6).
- **For Shoestring:** the same gates on GitHub-hosted runners cost $0, since standard runners are free on public repositories ([factory.md Costs](../factory.md#costs)). At Shoestring's size, nothing on the TypeScript side needs T3 Code's sharding or remote cache (judgment).

## 4. Mapping onto Shoestring

### 4.1 Fixed by the charting decisions

**Vite+ replaces Biome.** Each job Biome had in `factory.md` gets a successor:

| Biome's job | Successor | Status |
|---|---|---|
| Formatting | Oxfmt | Beta upstream |
| Recommended rules | Oxlint's `correctness` category plus explicit rules | Stable |
| `useReactCompiler` (nursery) | Oxlint's React Compiler rules, plus T3 Code's compile-coverage test, which checks what the compiler actually does | Rules experimental |
| `noFloatingPromises` (deferred) | Oxlint's type-aware `typescript/no-floating-promises` | Engine (tsgolint 7) stable since 2026-07-22 ([post](https://oxc.rs/blog/2026-07-22-type-aware-linting-stable)) |
| GritQL plugins | `@shadcn/lint` for the Tailwind web kit; in-repo JS-plugin rules for the Expo kit and for literal JSX text | JS plugins alpha |

There is one gap. Biome's `reactNative` domain has no native Oxlint counterpart ([plugins](https://oxc.rs/docs/guide/usage/linter/plugins.html)), so the bans on raw React Native text and literal colors become JS-plugin rules, on the alpha API. **M**

**CodeQL stays:** default setup for JavaScript/TypeScript and Actions, blocking high and critical alerts. Swift stays out: CodeQL 2.27.4 supports Swift 5.4–6.3 ([supported versions](https://github.com/github/codeql/blob/main/docs/codeql/reusables/supported-versions-compilers.rst)), and the watch builds with Swift 6.4. **H**

### 4.2 `just` and the Nix devShell next to Vite+

Both stay: Vite+ joins the devShell and replaces neither. **M**

- **Node and pnpm still come from Nix.**
  - Vite+'s `vp env`, which manages them, "belongs to the global CLI and is not included in the project-local vite-plus package". The local package "requires an existing Node.js runtime and package manager" ([env](https://viteplus.dev/guide/env), [local CLI](https://viteplus.dev/guide/local-cli)).
  - Shoestring installs only the local CLI: `vite-plus` in the catalog, run as `pnpm exec vp` or from `node_modules/.bin`. No `vite.plus` installer, no `vp env`, no `setup-vp` in CI.
  - `pmOnFail: ignore`, already set for F11, also stops pnpm from switching to the version in `packageManager` on its own, a behavior Vite+'s docs describe ([env](https://viteplus.dev/guide/env)).
- **Nix pins what npm can't.** That covers PostgreSQL 19, gitleaks, squawk, nixfmt, SwiftFormat and SwiftLint, Maestro, typos, actionlint, zizmor, process-compose and `just`. It also covers Node and pnpm themselves, which must match the `fetchPnpmDeps` build of `packages.default` (ADR 0003).
- **`just` stays the entry point across languages:** Nix (`build`, `vm-test`), Xcode (`ios`, `apple-test`), PostgreSQL (`db`), process-compose (`dev`), and `ci-<job>`, which reproduces a CI failure locally with the same binaries.
  - `vp run` can't take those jobs over. It caches only the tasks run through it, and it runs them "in a clean environment" where "only a small set of common variables, such as PATH, HOME, and CI, are passed through" ([cache](https://viteplus.dev/guide/cache)). The devShell's `PG*` variables would each need an `untrackedEnv` entry. Its cache in CI is still experimental.
  - The JavaScript recipes become thin calls to `vp check`, `vp test` and cached `vp run` tasks.
- **lefthook goes.** `.vite-hooks/pre-commit` runs `vp staged`, installed by `prepare: vp config --no-agent`. `--no-agent` stops Vite+ from editing the size-budgeted `AGENTS.md`.
- **treefmt goes (judgment).** Without Biome, it would only dispatch nixfmt and SwiftFormat. The edit hook, `vp staged` and `hygiene` call `vp fmt`, `nixfmt` and `swiftformat` by file type instead.
- **Biome goes**, as decided.
- **The hook bypasses get deny rules.** Vite+'s hooks honor `VP_GIT_HOOKS=0`, `HUSKY=0` and an init script ([commit hooks](https://viteplus.dev/guide/commit-hooks)), so Claude Code's deny rules should add `VP_GIT_HOOKS=0`, `HUSKY=0` and `vp hooks disable` beside `git commit --no-verify`. As before, this is a convenience, not a boundary.

### 4.3 Gate by gate

| T3 Code gate | Verdict | Shoestring form |
|---|---|---|
| `vp fmt` (Oxfmt) | Adopt | Every file Oxfmt reads, plus nixfmt and SwiftFormat for the rest; at edit, at commit, and in `hygiene` |
| `vp lint` base configuration | Adapt | `correctness` and `suspicious` as errors, `perf` rule by rule. No rule sits at warning level except the React Compiler rules, so `factory.md`'s "every check that runs blocks" holds; T3 Code's 35 "off" entries are a tested list of noisy rules to start from. `reportUnusedDisableDirectives: "error"`. `no-restricted-imports` encodes architecture.md's dependency direction: `packages/domain` imports no platform, `apps/server` no UI package, and connector code that calls the data provider runs only on the server (judgment) |
| `oxlint-plugin-t3code` | Adapt: about a dozen of the 33 rules, rewritten without Effect, keeping T3 Code's MIT notice | `require-suppression-reason`, `no-low-value-comments`. The test rules: `no-wall-clock-in-tests`, `no-real-time-wait-in-tests`, `no-polling-loop-in-tests`, `no-test-timeout-or-unreasoned-skip`, `no-test-in-loop`, `no-dom-test-environment`, `no-frame-waits-in-browser-tests`. `no-hermes-unsupported-apis` for `packages/*` and the Expo app. `no-manual-memo` and `no-hookless-use-function` with the React Compiler. Shoestring's own (judgment): raw style values and literal JSX text in the Expo kit, floats on money in `packages/domain` (ADR 0017), and a `can()` call in every mutation handler (ADR 0010). The plugin imports `vite-plus/lint/plugins` and tests every rule with `RuleTester` in the `js` job |
| `@shadcn/lint` | Adopt with [#17](https://github.com/fveauvy/shoestring/issues/17) | 0.2.5 on the web kit, with T3 Code's five rules; it replaces the GritQL raw-value plugin there |
| React Compiler rules and coverage test | Adopt | Rules at warning level until their false positives are measured; the coverage test blocks in `js` and `web` |
| Vitest through `vp test` | Adopt | `retry: 0`, no `.only` in CI, project-level timeouts only, and fake timers through an injected clock, since the household time zone defines today (ADR 0017) |
| Vitest browser mode | Adopt for the web kit | `*.browser.test.tsx` in Chromium through Playwright 1.65.0; Playwright E2E with axe in three browsers stays |
| `effect-tsgo tsrs` typecheck | Replace | `tsc -b` on TypeScript 7.0.2, since `tsrs` is built on 7.1 beta. `effect-tsgo tsc`, which accepts 7.0.2, only if [#21](https://github.com/fveauvy/shoestring/issues/21) adopts Effect |
| Type-aware lint (off in T3 Code) | Spike | `typeAware: true` with `no-floating-promises` and a short list. Oxlint's docs say to build dependent packages so `.d.ts` files exist ([type-aware](https://oxc.rs/docs/guide/usage/linter/type-aware.html)), while Shoestring's packages export source |
| knip | Adopt, blocking | 6.44.2 in `hygiene` and the Stop hook, with `types` off as in T3 Code. Not at commit, so the commit hook stays a fast secrets and privacy check (judgment). It needs no TypeScript 6 alias, so it leaves F4's list |
| pnpm settings | Adapt | Keep `factory.md`'s stricter set: an explicit `minimumReleaseAge`, which makes the delay strict, and `trustPolicy: no-downgrade`. Add `strictPeerDependencies: true`; `permissions` with `skills: false` for `drizzle-kit` and `expo-sqlite`, both in Shoestring's stack; Vite+'s overrides (`vite` → `vite-plus-core`, `vitest` pinned to the bundled version, [local CLI](https://viteplus.dev/guide/local-cli)); and a comment on every patch and exclusion. Skip `.pnpmfile.cjs`, which serves Windows and WSL only |
| `.vite-hooks` and `vp staged` | Adopt | §4.4 |
| `ci.yml` structure | Adapt | Keep Shoestring's jobs and `gate`. Add the skip test, fail-open `changes` with a sparse checkout, per-job timeouts, the rejection of committed PR evidence, and one path mapping shared by `changes` and `just check-fast`, as T3 Code's `check:changed` shares its mapping. Keep `permissions: {}`, `persist-credentials: false`, SHA pins through `actions.lock` and free GitHub-hosted runners |
| `Release Smoke` | Defer to [#20](https://github.com/fveauvy/shoestring/issues/20) | — |
| macOS native lint (SwiftLint) | Adapt | §4.4, Swift |
| PR size | Skip | Runs on `pull_request_target`, and `to-tickets` already sizes the tickets |
| PR vouch | Skip | Runs on `pull_request_target` and `issue_comment`. There is one maintainer, outside contributors' workflows already wait for approval, and vouch calls itself "an experimental system" ([vouch](https://github.com/mitchellh/vouch)) |
| Agent context budget | Adapt | Shoestring ships no agent-facing product text. A `hygiene` step enforces `factory.md`'s budgets instead: a root `AGENTS.md` under 4 KiB, nested ones under 2 KiB, and under 32 KiB from the root to any directory |
| Windows tests | Skip | No Windows target; the map puts Windows out of scope |
| Expo fingerprint | Defer to [#18](https://github.com/fveauvy/shoestring/issues/18) | Useful only with over-the-air updates, which ADR 0004 rules out today. If they come: the untrusted check with a trusted labeler, with a decision on the `workflow_run` trigger that zizmor flags |
| CodeRabbit | Adopt, advisory | Free Open source plan; `.coderabbit.config.ts` path instructions pointing at the `AGENTS.md` and `factory.md` rules; no automatic approvals and not a required check. Below 10 stars a review starts with `@coderabbitai review`, and Shoestring has none yet |
| Macroscope | Skip | Paid per KB, and its automatic approval runs against `factory.md`'s rule that every approval stays human |
| Third-party licenses | Adapt | ADR 0013's allowlist (permissive or MPL) in `hygiene`, plus a notices manifest built into the web, desktop and iPhone apps that fails on a missing license or notice, as T3 Code's does. The MIT license itself asks that its notice be "included in all copies or substantial portions of the Software" (T3 Code's `LICENSE`) |
| Remote task cache | Skip | Vite Task's local cache only, until Vite+ ships remote caching |
| `AGENTS.md` | Adapt | Keep the 4 KiB budget and borrow three things: before pushing, run the changed-files check, and leave the full suite to CI; the test rules, naming the lint rules that enforce them; a short "every surface" checklist covering web, desktop, iPhone and the watch glance, online and offline, personal and shared spaces, en and fr |
| `.agents/` and `.claude/skills` | Adopt | As planned, with an `agents/openai.yaml` per skill for Codex |
| `.cursor/` | Skip | No Cursor |
| `t3.json` | Adopt | "Setup Worktree" runs `just setup` with `runOnWorktreeCreate`, which fits one PostgreSQL cluster per worktree; "Dev" runs `just dev` with `stopOnSettle` |
| `.repos/` | Skip | 231 MB of reading material; revisit only if Effect is adopted |

### 4.4 Beyond T3 Code

**gitleaks 8.30.1 with the IBAN rule.** T3 Code has nothing equivalent.
- At commit: `gitleaks git --pre-commit --staged --redact` as a function entry in `vp staged`, so it scans the whole index instead of receiving a file list, the way T3 Code calls knip.
- In CI: the PR diff, in `hygiene`.
- Nightly: the full history.
- The generator's reserved IBANs stay allowlisted exactly, and an IBAN with a valid checksum is refused at commit and in CI ([factory.md Build order](../factory.md#build-order), step 0).

**The personal-terms check:** an in-repo script that reads `.git/info/personal-terms`, run as a function entry in `vp staged`, at commit only. By design, CI never sees the denylist.

**NixOS VM tests:**
- Unchanged: `nix flake check -L` on `ubuntu-24.04` with KVM (fresh install, upgrade, rollback, backup and restore, public interface closed).
- New: if the React web app builds with Vite 8 and Rolldown, as T3 Code's does, rather than with `expo export`, F5 becomes "`vp build` inside the `fetchPnpmDeps` derivation".
- The CLI is a Node.js script over N-API bindings (the `vp` script, `vite-plus.linux-x64-gnu.node`), which makes that plausible, but it is unverified.

**Swift for the watch:**
- SwiftFormat 0.64.6 at edit and at commit (`*.swift`), and in `hygiene`.
- SwiftLint 0.65.1 with `--strict` once the watch target exists (build order step 6). Its configuration starts from T3 Code's opt-in list: `force_unwrapping`, `implicitly_unwrapped_optional`, `fatal_error_message` and others.
- SwiftLint runs in `hygiene` on Linux if nixpkgs' build runs every configured rule. Otherwise it runs in `apple`, as T3 Code runs it on macOS at "~6.7x a Linux minute".
- Swift 6.4 in Swift 6 mode with warnings as errors, `swift test`, and the watch scheme's `xcodebuild test` stay in `apple`.
- No Swift SAST (§4.1).

### 4.5 Proposed guard rails for #16

This table would replace `factory.md`'s. *Edit* and *Stop* are the agent hooks; *Commit* is `.vite-hooks/pre-commit`. Every row blocks, except the React Compiler rules, which warn until their false positives are measured.

| Tool | Version | Scope | Agent hook | Commit | CI |
|---|---|---|---|---|---|
| `vp fmt` (Oxfmt) | 0.76.0, in Vite+ 1.5.0 | JS, TS, JSON, CSS, Markdown, YAML, TOML, HTML | Edit | Yes | `hygiene` (`vp check`) |
| nixfmt, SwiftFormat | 1.5.0, 0.64.6 | `*.nix`, `*.swift` | Edit | Yes | `hygiene`; `nix flake check` |
| `vp lint` (Oxlint): every rule an error or off | 1.91.0 | TS, JS | Stop | Staged files | `hygiene` |
| In-repo Oxlint plugin, with `RuleTester` fixtures | Vite+ 1.5.0, JS plugins in alpha | Per `overrides` | Stop | Staged files | `hygiene`; fixtures in `js` |
| `@shadcn/lint` | 0.2.5 | Web kit | Stop | Staged files | `hygiene` |
| React Compiler: Oxlint rules (warn) and the coverage test | Oxlint 1.91.0, `oxc-transform-react` 0.157.0 | Web and Expo apps | Stop | — | `js`, `web` |
| `tsc -b` | TypeScript 7.0.2 | All TypeScript | Stop | — | `js` |
| knip | 6.44.2 | Workspace | Stop | — | `hygiene` |
| Vitest (`vp test`): `retry: 0`, no `.only` in CI | 5.1.0 | Packages, server, tools | Stop (related tests) | — | `js`, `server` |
| Vitest browser mode | 5.1.0, Playwright 1.65.0 | Web kit | — | — | `web` |
| gitleaks with the IBAN rule | 8.30.1 | Staged changes, PR diff, history | — | Yes | `hygiene`; nightly |
| Personal-terms check | In-repo | Staged files | — | Yes | — |
| typos | 1.52.0 | Code and English docs | — | Yes | `hygiene` |
| `AGENTS.md` byte budgets | In-repo | `AGENTS.md` files | — | — | `hygiene` |
| License allowlist and notices manifest | In-repo | App build closures, pods, Swift packages | — | — | `hygiene` (dependency changes); app builds |
| zizmor, actionlint, `actions.lock` | 1.32.0, 1.7.12 | `.github/` | — | — | `hygiene` |
| CodeQL default setup | CLI 2.27.4 | JS/TS, Actions | — | — | The ruleset |
| Swift: strict compiler, `swift test`, watch tests; SwiftLint `--strict` from step 6 | Xcode 27.2; SwiftLint 0.65.1 | `apple/`, Swift modules | — | — | `apple`, perhaps `hygiene` for SwiftLint |
| `nix flake check -L`: packages and VM tests | `nixos-unstable` | `nix/`, the package | — | — | `nix` |
| `gate`, with the skip test | In-repo | Every job | — | — | The only required check |
| CodeRabbit | Open source plan | PRs, on request | — | — | Advisory |

Rows of `factory.md`'s table that don't change, and aren't repeated:
- server integration tests, squawk and migration drift, and the connector contract kit;
- `l10n-check`, `ds-check`, and Storybook or whichever workshop #17 picks;
- Playwright E2E with axe, Maestro, and the Expo checks;
- the PR-title check and the pin mirrors;
- secret scanning and push protection, Dependabot, and the rulesets.

### 4.6 What the final docs PR would change in `factory.md`

- **Pins:**
  - drop Biome, treefmt and lefthook;
  - add `vite-plus` 1.5.0 with its bundled versions, knip 6.44.2, `@shadcn/lint` 0.2.5 and `oxc-transform-react` 0.157.0;
  - raise typos to 1.52.0 and SwiftFormat to 0.64.6;
  - add the overrides Vite+ requires.
- **Layout:** `vite.config.ts` replaces `biome.jsonc`, `treefmt.toml` and `lefthook.yml`; `.vite-hooks/pre-commit` and `t3.json` are added; `tools/lint/` replaces `tools/biome/`.
- **Commands:** `just fmt` runs `vp fmt`, `nixfmt` and `swiftformat`. `just check-fast` uses the path mapping of the `changes` job.
- **Guard rails:** the table in §4.5. The React Compiler rules take over the exception `useReactCompiler` had.
- **Deferred tools:** knip leaves the list; `noFloatingPromises` becomes the type-aware spike.
- **CI:** `gate` tests every skip.
- **Agent setup:**
  - deny rules for `VP_GIT_HOOKS=0`, `HUSKY=0` and `vp hooks disable`;
  - an `agents/openai.yaml` per skill;
  - `AGENTS.md`'s budget enforced by `hygiene`.
- **Spikes:** F5 becomes "`vp build` in Nix", F4 drops knip, and F9 is rewritten for Oxlint (§6).

## 5. Versions

The newest stable release or release candidate of each tool, checked 2026-11-08.

| Tool | Version | Released | T3 Code uses | For Shoestring |
|---|---|---|---|---|
| `vite-plus` (Vite+) | 1.5.0 | 2026-11-04 | 1.4.0 | Adopt; 1.0 stable since 2026-09-28 |
| Bundled in Vite+ 1.5.0 | Vite 8.3.5, Rolldown 1.2.16, tsdown 0.23.4, Vitest 5.1.0, Oxlint 1.91.0, oxlint-tsgolint 7.0.2005, Oxfmt 0.76.0 | — | Vitest 5.0.5, Oxlint 1.89.0, Oxfmt 0.74.0 (from 1.5.0's changelog) | The bundled copies; standalone `oxlint` is 1.92.0 and `oxfmt` 0.77.0, and Oxfmt is beta upstream |
| `@vitest/browser-playwright` | 5.1.0 | 2026-11-02 | 5.0.5 | Adopt |
| Playwright | 1.65.0 | 2026-11-05 | 1.60.0 | Already planned |
| `@shadcn/lint` | 0.2.5 | 2026-11-02 | 0.1.5 | Adopt with #17; first published 2026-09-14 |
| `oxc-transform-react` | 0.157.0 | 2026-11-03 | ~0.152.0 | For the coverage test |
| knip | 6.44.2 | 2026-11-07 | 6.34.0 | Adopt |
| TypeScript | 7.0.2 | 2026-07-08 | None (`tsrs`) | Keep; 7.1.0-beta excluded |
| `tsrs` | 0.23.0-ts7.1.0-beta | 2026-11-07 | Through `@effect/tsgo` | Excluded: built on TypeScript 7.1 beta |
| `@effect/tsgo` | 0.67.5 | 2026-11-07 | 0.67.1 | Only with Effect (#21) |
| pnpm | 12.18.0 | 2026-11-05 | 12.14.0 | Keep |
| Node.js | 26.13.0 LTS | 2026-10-28 | 26.12.1 | Keep |
| `@coderabbitai/config` | 1.2.0 | 2026-11-07 | 1.1.0 | Optional types only |
| `voidzero-dev/setup-vp` | v1.22.0 | 2026-10-23 | `@v1` | Not used |
| `mitchellh/vouch` | v1.5.0 | 2026-07-12 | `@v1` | Skip |
| `@expo/fingerprint`, `expo-updates` | 0.21.9, 58.0.21 | — | 0.21.5 (patched), ~58.0.17 | Defer (#18) |
| gitleaks | 8.30.1 | 2026-03-21 | — | Keep |
| typos | 1.52.0 | 2026-11-02 | — | Keep; `factory.md` has 1.51 |
| zizmor | 1.32.0 | 2026-11-02 | — | Keep |
| actionlint | 1.7.12 | 2026-03-30 | — | Keep |
| nixfmt | 1.5.0 | 2026-09-11 | — | Keep |
| `just` | 1.59.0 | 2026-10-15 | — | Keep |
| SwiftFormat | 0.64.6 | 2026-11-06 | — | Keep; `factory.md` has 0.64.3 |
| SwiftLint | 0.65.1 | 2026-08-21 | Homebrew, unpinned | Adopt at step 6 |
| CodeQL CLI and action | 2.27.4 and v4.39.3 | 2026-11-02 and 2026-11-04 | — | Keep |
| lefthook, treefmt, Biome | 2.2.2, 2.6.0, 2.6.6 | — | — | Drop |

## 6. Risks and spikes

- **Oxfmt is beta, and each Vite+ upgrade can reformat.** Vite+ 1.5.0's notes warn that "the new oxlint and oxfmt versions can flag code that passed before. If your CI runs `vp check`, run `vp fmt` after upgrading." Mitigation: an exact catalog pin, with Dependabot's monthly npm group and its 7-day cooldown as the only upgrade path. **H**
- **JS plugins are alpha, so a custom rule can break in a patch release.** **M**
  - Keep the rule set small and import through `vite-plus/lint/plugins`.
  - Give every rule, and every known miss, a `RuleTester` fixture in `js`; this is F9 rewritten for Oxlint.
  - A rule that stops working then fails its fixtures instead of passing silently.
- **The React Compiler rules are experimental.** Mitigation: warning level, with the coverage test as the blocking gate. **M**
- **Type-aware lint over packages that export source is unverified.** Spike: on the step-1 workspace, run `vp check` with `typeAware` and `typeCheck` beside `tsc -b`, and compare diagnostics, time and memory. **L**
- **`vp build` inside a Nix derivation is unverified.** This is F5 for the new web stack. **L**
- **`vp staged` is lint-staged 17, so partially staged files go through its stash and restore.** Spike: a staged IBAN with a valid checksum and a denylisted term are refused, including when only part of a file is staged, and formatting never stages unstaged hunks. **M**
- **`@shadcn/lint` is young:** 0.x, first published 2026-09-14. Pin it exactly. **M**
- **Unverified:**
  - Blacksmith's billing granularity, behind the $0.29–0.35 range;
  - whether CodeRabbit's manual trigger below 10 stars applies to a personal account's repository as written on its plans page;
  - whether T3 Code runs CodeQL default setup, which only admins can see.

## Sources

**T3 Code at commit `6417151`** (`https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/<path>`):
- **Toolchain and configuration:** [`vite.config.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/vite.config.ts), [`package.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/package.json), [`pnpm-workspace.yaml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/pnpm-workspace.yaml), [`.pnpmfile.cjs`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.pnpmfile.cjs), [`.vite-hooks/pre-commit`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.vite-hooks/pre-commit), [`knip.jsonc`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/knip.jsonc), [`tsconfig.base.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/tsconfig.base.json), [`tsconfig.effect.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/tsconfig.effect.json), [`oxlint-plugin-t3code/`](https://github.com/pingdotgg/t3code/tree/6417151c093e54dbba0e6c824c69d03983793b9a/oxlint-plugin-t3code), [`apps/web/vite.config.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/web/vite.config.ts), [`apps/web/src/reactCompilerCoverage.test.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/web/src/reactCompilerCoverage.test.ts), [`apps/mobile/vite.config.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/vite.config.ts), [`apps/mobile/.swiftlint.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/.swiftlint.yml), [`apps/mobile/Brewfile`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/Brewfile), [`scripts/mobile-native-static-check.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/mobile-native-static-check.ts), [`scripts/check-changed.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/check-changed.ts), [`scripts/knip-schemas.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/knip-schemas.ts), [`.vscode/settings.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.vscode/settings.json).
- **CI:** [`.github/workflows/ci.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/ci.yml), [`pr-size.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/pr-size.yml), [`pr-vouch.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/pr-vouch.yml), [`agent-context-budget.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/agent-context-budget.yml), [`windows-tests.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/windows-tests.yml), [`mobile-fingerprint-check.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/mobile-fingerprint-check.yml), [`mobile-fingerprint-label.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/mobile-fingerprint-label.yml), [`release.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release.yml), [`.github/actions/setup-ci-deps/action.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/actions/setup-ci-deps/action.yml), [`.github/actions/ci-changes/action.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/actions/ci-changes/action.yml), [`.github/scripts/ci-changes.cjs`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/scripts/ci-changes.cjs), [`apps/server/src/provider/agentContextBudget.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/server/src/provider/agentContextBudget.json) and [its test](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/server/src/provider/AgentContextBudget.test.ts).
- **Review, licenses, cache:** [`.coderabbit.config.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.coderabbit.config.ts), [`.macroscope/`](https://github.com/pingdotgg/t3code/tree/6417151c093e54dbba0e6c824c69d03983793b9a/.macroscope), [`scripts/lib/third-party-licenses.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/lib/third-party-licenses.ts), [`docs/internals/open-source-licenses.md`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/internals/open-source-licenses.md), [`third-party-licenses.config.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/third-party-licenses.config.json), [`infra/task-cache/README.md`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/infra/task-cache/README.md), [`LICENSE`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/LICENSE).
- **Agents and policy:** [`AGENTS.md`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/AGENTS.md), [`CONTRIBUTING.md`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/CONTRIBUTING.md), [`docs/internals/testing.md`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/internals/testing.md), [`.agents/skills/`](https://github.com/pingdotgg/t3code/tree/6417151c093e54dbba0e6c824c69d03983793b9a/.agents/skills), [`.claude/skills`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.claude/skills), [`.cursor/rules/cursor-cloud.mdc`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.cursor/rules/cursor-cloud.mdc), [`t3.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/t3.json), [`scripts/setup-worktree.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/setup-worktree.ts), [`t3.json` schema](https://t3.codes/schema/t3.json).
- **GitHub API, read 2026-11-08:**
  - the ruleset [`rulesets/13856153`](https://api.github.com/repos/pingdotgg/t3code/rulesets/13856153) and [the rules on `main`](https://api.github.com/repos/pingdotgg/t3code/rules/branches/main);
  - durations of the 40 latest completed `ci.yml` runs, and of the latest successful side-workflow runs;
  - job timings of [run 42478610884](https://github.com/pingdotgg/t3code/actions/runs/42478610884) and [run 42482286835](https://github.com/pingdotgg/t3code/actions/runs/42482286835);
  - the check runs on PR 32926's head commit.

**Vite+:** [getting started](https://viteplus.dev/guide/), [check](https://viteplus.dev/guide/check), [lint](https://viteplus.dev/guide/lint), [format](https://viteplus.dev/guide/fmt), [test](https://viteplus.dev/guide/test), [commit hooks](https://viteplus.dev/guide/commit-hooks), [CI](https://viteplus.dev/guide/ci), [task caching](https://viteplus.dev/guide/cache), [GitHub Actions cache](https://viteplus.dev/guide/github-actions-cache), [project-local CLI](https://viteplus.dev/guide/local-cli), [env](https://viteplus.dev/guide/env), [1.0 announcement](https://voidzero.dev/posts/announcing-vite-plus-1-0), releases [v1.0.0](https://github.com/voidzero-dev/vite-plus/releases/tag/v1.0.0) and [v1.5.0](https://github.com/voidzero-dev/vite-plus/releases/tag/v1.5.0), [npm manifest of 1.5.0](https://registry.npmjs.org/vite-plus/1.5.0) and its published tarball (`bin/vp`, export map), [setup-vp releases](https://github.com/voidzero-dev/setup-vp/releases).

**Oxc:**
- **Docs:** [Oxfmt](https://oxc.rs/docs/guide/usage/formatter.html), [built-in plugins](https://oxc.rs/docs/guide/usage/linter/plugins.html), [JS plugins](https://oxc.rs/docs/guide/usage/linter/js-plugins.html), [type-aware linting](https://oxc.rs/docs/guide/usage/linter/type-aware.html).
- **Blog** ([feed](https://oxc.rs/feeds/blog-en.atom)): [Oxlint 1.0](https://oxc.rs/blog/2025-06-10-oxlint-stable), [Oxfmt beta](https://oxc.rs/blog/2026-02-24-oxfmt-beta), [JS plugins alpha](https://oxc.rs/blog/2026-03-11-oxlint-js-plugins-alpha), [type-aware linting stable](https://oxc.rs/blog/2026-07-22-type-aware-linting-stable), [React Compiler support](https://oxc.rs/blog/2026-08-18-react-compiler-support).
- **`tsrs`:** [repository](https://github.com/oxc-project/tsrs), [npm dist-tags](https://registry.npmjs.org/-/package/tsrs/dist-tags).

**Other tools:**
- **Effect:** [Effect-TS/tsgo](https://github.com/Effect-TS/tsgo), [npm](https://registry.npmjs.org/@effect/tsgo).
- **Linters and test runners:** [knip 6.44.2 manifest](https://registry.npmjs.org/knip/6.44.2), [shadcn-ui/lint](https://github.com/shadcn-ui/lint), [Vitest 4](https://vitest.dev/blog/vitest-4), [Vitest releases](https://github.com/vitest-dev/vitest/releases).
- **pnpm:** [supply-chain security](https://pnpm.io/supply-chain-security), [dependency resolution](https://pnpm.io/settings/dependency-resolution), [build](https://pnpm.io/settings/build), [peer dependencies](https://pnpm.io/settings/peer-dependencies).
- **Review services:** CodeRabbit [plans](https://docs.coderabbit.ai/management/plans), [pre-merge checks](https://docs.coderabbit.ai/pr-reviews/pre-merge-checks), [TypeScript configuration](https://docs.coderabbit.ai/configuration/typescript-configuration), [configuration reference](https://docs.coderabbit.ai/reference/configuration), [pricing](https://www.coderabbit.ai/pricing); Macroscope [pricing](https://docs.macroscope.com/pricing), [Check Run Agents configuration](https://docs.macroscope.com/check-run-agents/configuration), [approvability](https://docs.macroscope.com/approvability).
- **CI and security:** [Blacksmith pricing](https://www.blacksmith.sh/pricing), [vouch](https://github.com/mitchellh/vouch), [zizmor audits](https://docs.zizmor.sh/audits/#dangerous-triggers), GitHub [required status checks](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/collaborating-on-repositories-with-code-quality-features/troubleshooting-required-status-checks) and [job conditions](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-jobs-with-conditions), CodeQL [2.27.4](https://github.com/github/codeql-cli-binaries/releases/tag/v2.27.4) and [supported versions](https://github.com/github/codeql/blob/main/docs/codeql/reusables/supported-versions-compilers.rst).
- **Versions:** the [npm registry](https://registry.npmjs.org/) dist-tags and publish times; [Node.js releases](https://nodejs.org/dist/index.json); GitHub releases of [gitleaks](https://github.com/gitleaks/gitleaks/releases), [just](https://github.com/casey/just/releases), [typos](https://github.com/crate-ci/typos/releases), [zizmor](https://github.com/zizmorcore/zizmor/releases), [actionlint](https://github.com/rhysd/actionlint/releases), [nixfmt](https://github.com/NixOS/nixfmt/releases), [SwiftFormat](https://github.com/nicklockwood/SwiftFormat/releases), [SwiftLint](https://github.com/realm/SwiftLint/releases), [CodeQL action](https://github.com/github/codeql-action/releases).

**This repository:** [`docs/factory.md`](../factory.md), [`docs/architecture.md`](../architecture.md), [factory-v2.md](factory-v2.md), ADRs [0003](../adr/0003-nixos-just-tailscale-infrastructure.md), [0004](../adr/0004-expo-react-native-for-iphone-web-and-mac.md), [0010](../adr/0010-spaces-for-access-and-sync.md), [0013](../adr/0013-mit-license.md) and [0017](../adr/0017-money-in-minor-units-time-in-the-household-zone.md), and issues [#6](https://github.com/fveauvy/shoestring/issues/6), [#7](https://github.com/fveauvy/shoestring/issues/7), [#16](https://github.com/fveauvy/shoestring/issues/16), [#18](https://github.com/fveauvy/shoestring/issues/18) and [#21](https://github.com/fveauvy/shoestring/issues/21).
