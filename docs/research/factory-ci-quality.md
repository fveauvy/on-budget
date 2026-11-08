# Software factory: CI/CD, guard rails and agentic tooling

> **Status (2026-11-08).** Background research. The ADRs, `SPEC.md`, [`docs/factory.md`](../factory.md) and [`docs/architecture.md`](../architecture.md#references) win where this note disagrees. It uses on-budget, the working name, for what is now Shoestring ([ADR 0020](../adr/0020-shoestring-as-the-product-name.md)). Superseded or amended here: Swift-only rails, Kamal, macOS archives, license.

> **Status (2026-11-07).** The Swift-centric guard rails and CI layout assume SwiftUI iOS and macOS apps. The owner has since chosen one UI codebase shared by iOS, macOS and web, with a native SwiftUI watch app ([ADR 0002](../adr/0002-one-ui-codebase-native-watch.md)), so the pipeline will be re-planned once the UI stack is chosen; until then nothing here lints, tests or deploys the shared UI or the web app. The GitHub-plan, signing, TestFlight and agent-hook findings stay relevant. No always-on Mac is assumed, which weakens the self-hosted runner recommendation. The account is on GitHub Free: branch protection returned 403 "Upgrade to GitHub Pro" while the repo was private. The repo is now public ([ADR 0001](../adr/0001-self-hosted-single-owner-instances.md)), so the same call returns 404 "Branch not protected" and Free covers rulesets, secret scanning and hosted macOS runners; TL;DR 1, 2 and 6 follow from that. The Kamal deploy (TL;DR 5, §2) predates [ADR 0003](../adr/0003-nixos-just-tailscale-infrastructure.md) (NixOS flake, `just`, Tailscale) and is superseded.

> Researched 2026-11-07, revised the same day after independent fact-checks. Prices, versions and runner images are as of that date. Confidence: **H** high, **M** medium, **L** low. "Secondary" marks a source that isn't primary.

## Question

How should `on-budget` set up CI/CD and quality guard rails so that agent-written code stays correct and maintainable, at solo-developer cost? The setup, as first researched: SwiftUI iOS and macOS apps over shared local Swift packages, a Swift or TypeScript API on Postgres running on one small server, a private repo `fveauvy/on-budget` owned by a personal account, one human, and code mostly written by Claude Code and Codex through T3 Code. Since then the UI became one shared codebase and the repo became public (see Status). The four sub-questions are:

1. Where should Apple CI run?
2. How should signing, TestFlight, notarization, versioning and backend deploys work?
3. Which guard-rail tools, exactly?
4. How do we make Swift development agent-friendly?

## TL;DR

1. **Split CI by cost.** (M-H)
   - While the repo is public, GitHub's standard hosted runners are free and unlimited, macOS included.
   - Linux hosted runners: every check that doesn't need a Mac (secrets, format, lint, PR title, backend).
   - Hosted macOS runners: every `swift test` and `xcodebuild` job, on `xcode-27` (a preview) if the project needs Xcode 27, otherwise on `macos-26`.
   - **Xcode Cloud**, which includes 25 compute hours a month: release archives to TestFlight for iOS and macOS, signed by Apple, so no signing secret is stored in GitHub.
   - **No self-hosted runner.** GitHub says they "should almost never be used for public repositories", and no always-on Mac is assumed.
   - If the repo goes private, hosted macOS costs $0.062/min, and Xcode Cloud's 25 hours can't absorb the PR and main runs (see question 1).
2. **Stay on GitHub Free while the repo is public.** Public repos on Free get rulesets and required checks, secret scanning, push protection, code scanning and free hosted runners. CodeQL's license also requires an OSI-approved license on the code, which is undecided. Still missing: the merge queue, which needs an organization, and Code Quality's coverage rules, which need Team or Enterprise Cloud, so the coverage gate stays one we write ourselves. Keep gitleaks in the pre-commit hook, which stops secrets before they're committed. The ruleset only binds agents whose token can't administer the repo: the owner's `gh` token can delete it. GitHub Pro ($4/month) matters only if the repo goes private; even then a user-owned private repo gets no secret scanning, push protection or CodeQL, and gitleaks in CI would replace the first two. (H)
3. **Guard rails, all pinned with `mise` and run the same way locally, in agent hooks and in CI.** These are the Swift-side rails; the shared UI's rails wait for the UI-stack choice. (M)
   - SwiftFormat 0.64.6 and SwiftLint 0.65.1 in strict mode.
   - Swift 6 language mode, approachable concurrency, and warnings treated as errors.
   - Swift Testing for unit tests, XCTest only for UI and performance tests.
   - swift-snapshot-testing 1.19.7 on one pinned simulator.
   - A coverage floor on the domain packages.
   - Periphery (dead code), advisory only.
   - Dependabot, Conventional Commit PR titles, and SHA-pinned Actions.
4. **Agent-friendly codebase.** (M)
   - Domain logic lives in local SPM packages, so `swift test` runs in seconds with no simulator.
   - One entry point, `mise run check`, used by hooks, agents and CI.
   - Hooks in both agents: format on edit, fast tests on stop, with a `stop_hook_active` guard against loops.
   - Xcode's MCP server (`xcrun mcpbridge`) for interactive work; `xcodebuild`, MobileBuildMCP and `xcsift` for headless work.
   - Xcode 27.2's JSON `.xcproj` project format makes merge conflicts less likely, not impossible. It opens only in Xcode 27.0 and later, and CocoaPods' Xcodeproj gem (used by CocoaPods and fastlane) can't read it yet, so adopt it once the chosen stack's native tooling can.
5. **Backend (deploy superseded by ADR 0003).** This plan built the container on Linux, deployed it with Kamal 2.12 over Tailscale (no public SSH), let the `/up` health check gate traffic, and used `kamal rollback` to undo. ADR 0003 has since chosen a NixOS flake deployed with `just` over Tailscale, so the deploy needs re-planning. Expand/contract migrations linted by squawk don't depend on the deploy tool. If deploys run from GitHub Actions, production secrets sit in a GitHub environment restricted to tags, which on the public repo can also require a reviewer. (M)
6. **Cost.** **$0/month extra** while the repo is public: standard hosted runners are free there, macOS included, and release archives use about 4 of Xcode Cloud's 25 hours. If the repo goes private: $4 for Pro, plus the macOS jobs at about **$170/month** on hosted runners ($205 with nightly runs), or US$49.99/month for Xcode Cloud's 100-hour tier, or a self-hosted runner on a Mac that sleeps (see the pipeline section). (M)

## 1. CI for Apple apps

### GitHub-hosted macOS runners

Image `20261026`, from the [runner-images README](https://github.com/actions/runner-images), the [macos-26 readme](https://github.com/actions/runner-images/blob/main/images/macos/macos-26-arm64-Readme.md), the [xcode-27 readme](https://github.com/actions/runner-images/blob/main/images/macos/xcode-27-arm64-Readme.md) and the [Xcode 27 preview notice](https://github.com/actions/runner-images/issues/14404):

| Label | OS, arch | Xcode | Status |
|---|---|---|---|
| `macos-latest` = `macos-26` | macOS 26, arm64 | 26.0.1 to 26.6 (default 26.6) | GA |
| `xcode-27` | macOS 27 base since 2026-09-16, arm64 | 27.0, **27.1 (default)**, 27.2 RC | **Preview**: "software can be unstable", possible queueing |
| `macos-15` | macOS 15, arm64 | 16.0 to 16.4 (default 16.4), plus 26.0.1 to 26.3 | GA |
| `macos-14` | none | none | Retired 2026-11-02 ([#15226](https://github.com/actions/runner-images/issues/15226)) |

- **Image policy.** The stated rule is one Xcode major version per macOS image, though `macos-15` carries two. Betas only appear on the newest image ([README](https://github.com/actions/runner-images)). Xcode 27 lives only on `xcode-27`.
- **Hardware.** Standard arm64 macOS runners are a 3-core M1 with 7 GB RAM and a 14 GB SSD, on public and private repos alike; the public-repo list includes `xcode-27` ([hosted runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)). Larger macOS runners are only for Team and Enterprise organizations ([pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing)).
- **Price.** On private repos, `actions_macos` costs **$0.062/min**, against $0.006/min for 2-core Linux x64 and $0.005/min for arm64. Each job's minutes are rounded up ([runner pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing)). On public repos the macOS minutes cost nothing: "Use of the standard GitHub-hosted runners is free and unlimited on public repositories" ([hosted runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)). (H)
- **Included compute is now a dollar amount**: Free $12/month, Pro $18/month. That is about 193 or 290 macOS minutes. Self-hosted runners, and standard runners on public repos, are free and don't draw on it ([Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions)). (H) Note: [github.com/pricing](https://github.com/pricing) still advertises "2,000 CI/CD minutes/month" for Free and "3,000" for Team; the billing docs' dollar amounts are the authoritative figures.
- **Self-hosted fee.** GitHub announced a $0.002/min "Actions cloud platform charge" for self-hosted runners in December 2025, then wrote: "We're postponing the announced billing change for self-hosted GitHub Actions to take time to re-evaluate our approach" ([GitHub, 2025-12-15](https://github.com/resources/insights/2026-pricing-changes-for-github-actions); [Techzine, secondary](https://www.techzine.eu/news/devops/137396/github-bends-to-criticism-and-delays-paid-self-hosting-of-runners/)). The same page says self-hosted usage on public repos "will remain free", and today's billing docs still list self-hosted runners as free. Whether a new date has been set: unverified. (H)
- **Concurrency.** At most 5 concurrent macOS jobs on Free and Pro ([limits](https://docs.github.com/en/actions/reference/limits)).
- **App Store floor.** Since 2026-04-28, uploads must be built with Xcode 26 or later ([upcoming requirements](https://developer.apple.com/news/upcoming-requirements/)). `macos-26` still produces valid uploads. It can't open `.xcproj` projects (Xcode 27 only) and doesn't ship the iOS 27 SDK.
- **Preinstalled on `xcode-27`**: SwiftFormat 0.64.3, xcbeautify 3.2.1, fastlane 2.241.2, xcodes 2.1.0, CodeQL bundle 2.27.3.

### Self-hosted runner on a Mac

- **Status: not recommended now.** The repo is public, and no always-on Mac is assumed, so none can be the always-on runner this section assumed. Kept as reference in case the repo goes private. (H)
- **GitHub's warning.** Self-hosted runners aren't ephemeral and "can be persistently compromised by untrusted code in a workflow". They "should almost never be used for public repositories on GitHub, because any user can open pull requests against the repository and compromise the environment". GitHub advises caution even for private repos ([secure use](https://docs.github.com/en/actions/reference/security/secure-use)).
- **Threats here.** On the public repo, anyone can open a PR from a fork. By default only first-time contributors need approval before their workflows run; "Require approval for all external contributors" tightens that, but approved code still runs on the machine ([Actions settings](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository)). On a private repo, what remains is the supply chain (SPM packages, build-tool plugins, macros, third-party actions) and prompt-injected agent code. Agents already run that code on this Mac.
- **Hardening.**
  - Run the runner as a dedicated standard macOS user, `ci`, with no access to the owner's home, keychain or signing identities.
  - Register the runner to this repo only.
  - Default workflows to `permissions: contents: read`.
  - Turn on the repo setting "Require actions to be pinned to a full-length commit SHA", and add `.github/workflows/actions.lock` (generated by `gh actions-lock`) ([Actions settings](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository), [lockfiles](https://docs.github.com/en/actions/concepts/security/workflow-dependency-lockfiles)).
  - Clean the workspace on every job.
  - For stronger isolation, use JIT runners or Tart VMs. Tart 2.41.1 is Fair Source and royalty-free on personal computers ([Tart licensing](https://tart.run/licensing/)).
- **Operations.**
  - A job waits **at most 24 h** for a self-hosted runner, then it is cancelled ([limits](https://docs.github.com/en/actions/reference/limits)).
  - The Mac must be awake, and local agents compete with CI for CPU. On a laptop that sleeps or travels, jobs sit queued until it wakes or the 24 h limit cancels them.
  - We own Xcode and simulator upkeep. The runner itself is `actions/runner` 2.338.2.
- **Xcode 27 requirements.** macOS Tahoe 26.6 or later, Apple silicon only ([Xcode 27 notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27-release-notes)).
- **Local Mac.** An Apple silicon Mac that meets Xcode 27's requirements. Local work needs Xcode 27.2, installed for example with `xcodes`.
- **Xcode packaging change.** Xcode 27.3 is the last release shipped as a `.xip`. Later releases come as `.aea` archives, extracted with `archiveutil extractXcodeAEA` ([27.3 beta notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_3-release-notes)).

### Xcode Cloud

- **Pricing.** 25 compute hours a month come with the Developer Program. Add-ons: 100 h for $49.99, 250 h for $99.99, 1,000 h for $399.99 a month. Every action counts, so 5 test actions of 12 minutes each use 1 hour ([Xcode Cloud](https://developer.apple.com/xcode-cloud/)). (H)
- **What it does.**
  - Build, test, analyze and archive actions.
  - Custom scripts: `ci_scripts/ci_post_clone.sh`, `ci_pre_xcodebuild.sh` and `ci_post_xcodebuild.sh` ([custom scripts](https://developer.apple.com/documentation/xcode/writing-custom-build-scripts)).
  - Secret environment variables, plus built-in ones: `CI_BUILD_NUMBER`, `CI_TAG`, `CI_PULL_REQUEST_*` ([env vars](https://developer.apple.com/documentation/xcode/environment-variable-reference)).
  - TestFlight "What to Test" notes, read from `TestFlight/WhatToTest.<locale>.txt` ([tester notes](https://developer.apple.com/documentation/xcode/including-notes-for-testers-with-a-beta-release-of-your-app)).
  - Its PR build status can be a required GitHub check ([merge requirements](https://developer.apple.com/documentation/xcode/configuring-requirements-for-merging-a-pull-request)).
- **Signing options.** An archive's "Deployment Preparation" is None, TestFlight (Internal Testing Only), or TestFlight and App Store ([actions](https://developer.apple.com/documentation/xcode/configuring-your-xcode-cloud-workflow-s-actions)). For direct distribution, Apple documents a **Notarize post-action**: "Add a post-action to notarize a macOS app if you intend to distribute it through your own channels", and the notarized app downloads from the build's artifacts ([distribution workflow](https://developer.apple.com/documentation/xcode/creating-a-workflow-that-builds-your-app-for-distribution)). These pages don't name the certificate type Xcode Cloud signs with. (H) Two 2026 forum threads report Xcode Cloud Developer ID exports failing with an empty certificate query and an HTTP 409 ([853905](https://developer.apple.com/forums/thread/853905), [855013](https://developer.apple.com/forums/thread/855013); user reports, unverified).
- **Limits.**
  - Apple platforms only, so no Linux and no backend. Logs live in Xcode and App Store Connect, which agents read less easily than GitHub logs: those are one `gh run view --log-failed` away.
  - Xcode projects and workspaces only: "You need to use Xcode to initially configure your project or workspace" ([actions](https://developer.apple.com/documentation/xcode/configuring-your-xcode-cloud-workflow-s-actions)). It "requires a consistent Xcode project or workspace that's continuously present"; with a tool that generates or edits it, "the initial configuration of Xcode Cloud and subsequent builds may fail" ([project setup](https://developer.apple.com/documentation/xcode/setting-up-your-project-to-use-xcode-cloud)). (H) So a Mac shell that isn't an Xcode project (Electron, Tauri) can't use it, a generated `ios/` folder (Expo prebuild) has to be committed, and the watch target must sit in the project or workspace that builds the iOS app it ships inside (inference).
  - The build image "doesn't include third-party tools or dependencies" apart from Homebrew ([dependencies](https://developer.apple.com/documentation/xcode/making-dependencies-available-to-xcode-cloud)), and custom scripts run "for every action" ([custom scripts](https://developer.apple.com/documentation/xcode/writing-custom-build-scripts)). A TypeScript UI therefore installs Node and its packages in `ci_post_clone.sh` once per action, on compute hours. (H)

### Codemagic and Bitrise

- **Codemagic.** The personal plan includes 500 free macOS M2 minutes a month. Pay-as-you-go costs $0.095/min on M2 and $0.114/min on M4 ([pricing](https://codemagic.io/pricing/)).
- **Bitrise.** Hobby is free ("a few times per month"). Starter costs $89/month billed annually or $99 billed monthly ([pricing](https://bitrise.io/pricing)).
- **Verdict.** Neither beats free hosted runners plus Xcode Cloud while the repo is public. Codemagic's 500 free minutes are a sensible plan B if it goes private. (M)

### Recommendation for question 1

- **macOS jobs for PRs and main** run on GitHub-hosted runners: free while the repo is public, with no machine to keep awake. Use `xcode-27` if the project needs Xcode 27 (`.xcproj`, iOS 27 SDK), otherwise the GA `macos-26`. Watch for the preview's queueing and the 7 GB of RAM.
- **Release archives** run on Xcode Cloud.
- **Everything else** runs on Linux hosted runners.
- **No self-hosted runner** on the public repo (see above).
- **If the repo goes private again**, every option costs something, because no always-on Mac is assumed (H):
  - Xcode Cloud for PR checks: 25 hours cover only 75 runs at 20 compute minutes, or 100 at 15, with nothing left for release archives. The cost model's 150 PR and 60 main runs need about 45 to 70 hours, plus 4 for archives, so the 100-hour tier at US$49.99/month ([Xcode Cloud](https://developer.apple.com/xcode-cloud/)).
  - Hosted macOS runners: about $167/month for PR and main at the model's volumes.
  - A self-hosted runner on a Mac that sleeps: free, but jobs wait until it wakes, up to the 24 h limit.
  - Or less macOS work: path filters so that only native (Swift, Xcode) changes run it, which matters once the UI is shared code. (M)

## 2. CD

### Code signing

| Option | Mechanism | Secret stored in GitHub | Use when |
|---|---|---|---|
| **Xcode Cloud managed signing** | Xcode Cloud signs archives per Deployment Preparation; its docs don't name the certificate type (user reports show cloud-managed "(Xcode Cloud)" certificates) | none | **Default, for TestFlight** |
| `xcodebuild` + App Store Connect API key | `-allowProvisioningUpdates -authenticationKeyPath/-authenticationKeyID/-authenticationKeyIssuerID` gives automatic signing in headless CI | `.p8` Team key; Admin role in practice (forum reports) | Releasing from GitHub Actions; not Developer ID cloud signing (known bug) |
| fastlane `match` 2.241.2 | Encrypted certificates and profiles in git, GCS or S3; `--readonly` in CI ([match](https://docs.fastlane.tools/actions/match/)) | Storage credentials and passphrase | Manual signing, or certificates shared across machines |

- **Since Xcode 13** (per its [release notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-13-release-notes)):
  - `xcodebuild` accepts API keys for automatic signing "in headless environments"; a key gets its role when it's created.
  - Cloud signing covers App Store Connect, Ad Hoc, Enterprise and Developer ID distribution.
  - Cloud certificates can't be exported: "you can't transmit or store the private key on your Mac".
  - By default they are "accessible only to members of your development team with the Admin role (or Account Holder for Developer ID)", and App Store Connect can grant them to other roles. Apple's roles table gives cloud-managed Developer ID certificates to the Account Holder, and to Admins as a separate permission; other cloud-managed types go to Account Holders and Admins, and to App Managers and Developers as a separate permission ([roles](https://developer.apple.com/help/account/access/roles/)). (H)
- **API keys.** "Individual keys aren't able to use Provisioning endpoints, access Sales and Finance, or `notaryTool`", so headless signing and notarization need a Team key, which takes an Admin account to generate ([API keys](https://developer.apple.com/documentation/appstoreconnectapi/creating-api-keys-for-app-store-connect-api)). (H) That cloud signing needs an Admin-role key comes from developer reports, not Apple's docs ([forum 698117](https://developer.apple.com/forums/thread/698117), secondary). (M)
- **Developer ID with an API key.** An Apple DTS engineer answered on the forums: "Your issue is a known bug. You can't cloud sign with a Developer ID cert when authenticating with an App Store Connect API key" (FB16835802) ([forum 776036](https://developer.apple.com/forums/thread/776036), read through search results because the forums block automated reads). No fix found. (M)
- **Renewal.** "A new cloud-managed certificate is automatically created 90 days before expiration when new signing requests are received", so renewal depends on signing activity; Account Holders and Admins can also rotate certificates by hand ([cloud-managed certs](https://developer.apple.com/help/account/certificates/cloud-managed-certificates)). (H)
- **Implication.** For direct distribution, plan on a local Developer ID certificate (or `match`) in CI, or on Xcode Cloud's Notarize post-action, not on headless cloud signing with an API key (spike 4). fastlane only earns its Ruby toolchain if API-key automatic signing fails. (M)

### Uploading to TestFlight

- **Upload paths.**
  - Xcode, including `xcodebuild -exportArchive` with an upload destination.
  - `xcrun altool --upload-app`.
  - Transporter.
  - The App Store Connect API ([upload builds](https://developer.apple.com/help/app-store-connect/manage-builds/upload-builds)). Its Build Uploads endpoints arrived in App Store Connect API 4.1 ([4.1 notes](https://developer.apple.com/documentation/appstoreconnectapi/app-store-connect-api-4-1-release-notes), [POST /v1/buildUploads](https://developer.apple.com/documentation/appstoreconnectapi/post-v1-builduploads)).
  - fastlane `upload_to_testflight`, which takes `changelog`, `groups` and `distribute_external` ([docs](https://docs.fastlane.tools/actions/upload_to_testflight/)).
- **TestFlight limits.** Up to 100 internal testers, who must be team members with the Account Holder, Admin, App Manager, Developer or Marketing role. Up to 10,000 external testers, invited by email or public link. The first build shared with external testers needs TestFlight App Review ([TestFlight](https://developer.apple.com/testflight/)). Builds expire: "Your build becomes unavailable for testers after 90 days" ([TestFlight overview](https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview)), so the household needs a new build at least every 90 days. Guideline 2.2 expects TestFlight apps to be "intended for public distribution", and significant updates to go to TestFlight App Review first ([guidelines](https://developer.apple.com/app-store/review/guidelines/)). (H)
- **Who tests.** The friends-and-family group this note first planned is gone: ADR 0001 dropped the hosted service. The household uses internal testing where each member holds an App Store Connect role, otherwise an external group. How other self-hosters get the native apps is undecided (`PRODUCT.md`).
- **Recommended flow.** An Xcode Cloud workflow starts on tag `v*`, archives iOS and macOS, and sends both to TestFlight: internal testers automatically, an external group (if any) after review. Tag a release at least every 90 days. The GitHub-only alternative is `xcodebuild archive` plus `-exportArchive` with an upload destination and an API key. (M)

### macOS outside the App Store (only if needed)

- **Requirements.** A Developer ID certificate, the Hardened Runtime, a secure timestamp, and no `get-task-allow` entitlement ([notarizing](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution)).
- **Commands** ([workflow](https://developer.apple.com/documentation/security/customizing-the-notarization-workflow), [TN3147](https://developer.apple.com/documentation/technotes/tn3147-migrating-to-the-latest-notarization-tool)):
  - `ditto -c -k --keepParent App.app App.zip`
  - `xcrun notarytool submit App.zip --key AuthKey.p8 --key-id KEY --issuer UUID --wait`
  - `xcrun notarytool log <id>`. Always read the log.
  - `xcrun stapler staple App.app`. A ZIP can't be stapled, so staple the app or DMG.
- **altool is gone for notarization**: the notary service stopped accepting it on 2023-11-01. (H)
- **Recommendation.** Ship the Mac app through TestFlight first. Mac App Store apps "must be appropriately sandboxed" (Guideline 2.4.5(i), [guidelines](https://developer.apple.com/app-store/review/guidelines/)), and Mac TestFlight builds go through the same App Store Connect upload, so plan on the sandbox (inference). Add Developer ID and notarization only if the app can't be sandboxed or needs to be downloadable directly, for self-hosters for example. In CI, use Xcode Cloud's Notarize post-action or a local Developer ID certificate, not API-key cloud signing (see Code signing). (M)

### Versioning

- **Apple's rules.** (H)
  - `CFBundleShortVersionString` is three integers. `CFBundleVersion` is one to three integers ([CFBundleVersion](https://developer.apple.com/documentation/bundleresources/information-property-list/cfbundleversion)).
  - iOS needs a unique (version, build) pair.
  - **Mac build numbers must keep increasing, even across versions.**
  - Xcode Cloud assigns integer build numbers to every build, PR builds included. Admins and App Managers can set the next number ([build numbers](https://developer.apple.com/documentation/xcode/setting-the-next-build-number-for-xcode-cloud-builds)).
- **Our strategy.**
  - The marketing version comes from tag `vX.Y.Z` (SemVer) and lives in one xcconfig.
  - **One monotonic integer** serves as the build number for both iOS and macOS: `CI_BUILD_NUMBER`.
  - If GitHub Actions ever uploads, beware: `github.run_number` doesn't change on a re-run ([contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)). Combine it with `run_attempt` or read the latest build from App Store Connect. Then set Xcode Cloud's next number above whatever was used.
- **Release notes.** Conventional Commits feed git-cliff 2.14.2, which feeds two places:
  - `TestFlight/WhatToTest.en-US.txt`, written in `ci_post_xcodebuild.sh`. Apple's sample script uses `git log -3`.
  - The GitHub Release.

  The API alternative is the `whatsNew` attribute on `betaBuildLocalizations` ([attributes](https://developer.apple.com/documentation/appstoreconnectapi/betabuildlocalizationcreaterequest/data-data.dictionary/attributes-data.dictionary)).

### Backend deploy pipeline

Superseded for deploys by [ADR 0003](../adr/0003-nixos-just-tailscale-infrastructure.md) (NixOS flake, `just`, Tailscale, no containers). The PR checks, without `docker build`, and the migration rules still apply.

1. **PR.**
   - Lint, typecheck, and test against a Postgres service container.
   - squawk 2.71.0 lints new SQL migrations ([squawk](https://github.com/sbdchd/squawk)).
   - `docker build`, without pushing.
2. **main.**
   - Build the image tagged with the git SHA. Use an arm64 Linux runner ($0.005/min) if the server is arm64.
   - Push to GHCR. GHCR storage shares Pro's artifact allowance, which the docs give as 1 GB in one place and 2 GB in another ([billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions), [plans](https://docs.github.com/en/get-started/learning-about-github/githubs-plans)).
   - Or skip GHCR: Kamal can run a local registry (`registry.server: localhost:5555`) ([Kamal registry](https://kamal-deploy.org/docs/configuration/docker-registry/)).
3. **Deploy** (tag `api-v*` or manual dispatch).
   - `tailscale/github-action@v4` joins the tailnet as an ephemeral `tag:ci` node, using an OAuth client or workload identity federation ([action](https://github.com/tailscale/github-action)).
   - Then run `kamal deploy` (Kamal 2.12.0).
   - Migrations are expand/contract only, run once from a `pre-deploy` hook or a one-off container ([hooks](https://kamal-deploy.org/docs/hooks/overview/)), after a `pg_dump`.
4. **Health check.** kamal-proxy polls `/up` every second until the deploy timeout, with 5 seconds per request ([proxy](https://kamal-deploy.org/docs/configuration/proxy/)).
5. **Rollback.** `kamal rollback <version>`. Old containers are pruned after 3 days by default ([rollback](https://kamal-deploy.org/docs/commands/rollback/)). The database is fixed forward, never auto-downgraded.

- **Secrets.** Use a GitHub environment named `production`, limited to the right tags or branches. That restriction works on all public repos and on Pro private repos. Required reviewers and wait timers are public-repo only on Free, Pro and Team, so on the public repo the owner can be the required reviewer (self-review stays allowed unless "prevent self-reviews" is on). If the repo goes private, the manual `workflow_dispatch` is the approval step ([environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)). (H)

## 3. Quality guard rails

### Formatter: SwiftFormat or swift-format

- **swift-format 604.0.0.** It ships in the Swift 6 toolchain, Xcode 16 and later, as `swift format`, and `lint --strict` fails on warnings ([README](https://github.com/swiftlang/swift-format)). Nothing to install, but its version follows the toolchain, so local and CI Xcode must match.
- **SwiftFormat 0.64.6 (2026-11-06).** Many more rules. `--lint` exits non-zero and `--reporter github-actions-log` annotates PRs. It reads `.swift-version` and supports `--lintonly` rules ([README](https://github.com/nicklockwood/SwiftFormat)). It has Linux binaries and is preinstalled on GitHub images.
- **Pick SwiftFormat**, pinned in `mise.toml`. Run it from the agent hook, the pre-commit hook and CI. Never run two formatters. (M)

### SwiftLint

- **Version.** 0.65.1 (2026-08-21). `--strict` fails on warnings. A build-tool plugin is available through `SimplyDanny/SwiftLintPlugins` ([README](https://github.com/realm/SwiftLint)), and a Linux amd64 binary ships with each release ([0.65.1 assets](https://github.com/realm/SwiftLint/releases/tag/0.65.1)).
- **Usage.** Run the CLI, not the plugin, to keep builds fast and give agents one lint step. Disable the style rules SwiftFormat already owns. (M)

### Swift 6 language mode and concurrency

- **Packages.**
  - `swiftLanguageModes: [.v6]` ([SE-0441](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0441-formalize-language-mode-terminology.md)).
  - `.defaultIsolation(MainActor.self)` on UI and app targets only ([SE-0466](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)).
  - `.enableUpcomingFeature("NonisolatedNonsendingByDefault")` ([SE-0461](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)).
  - `.treatAllWarnings(as: .error)`, available since Swift 6.2 ([Swift 6.2](https://www.swift.org/blog/swift-6.2-released/)).
  - Swift 6.4 adds the `@diagnose` attribute for controlling warnings in source (SE-0522) ([Swift 6.4](https://www.swift.org/blog/swift-6.4-released/)).
- **Xcode targets.** Swift language version 6, "Approachable Concurrency" on, MainActor as the default isolation for app targets, and warnings as errors at least in CI.
- **Domain packages** keep the `nonisolated` default and `Sendable` value types, so they stay portable to Linux and the server. (M)

```swift
// swift-tools-version: 6.2
// (6.2 minimum for defaultIsolation/treatAllWarnings; Xcode 27 ships Swift 6.4)
let strict: [SwiftSetting] = [.enableUpcomingFeature("NonisolatedNonsendingByDefault"),
                              .treatAllWarnings(as: .error)]
// .target(name: "BudgetCore", swiftSettings: strict)
// .target(name: "BudgetUI",   swiftSettings: strict + [.defaultIsolation(MainActor.self)])
// Package(..., swiftLanguageModes: [.v6])
```

### Swift Testing or XCTest

- **Apple's guidance.** Use Swift Testing for new unit tests. Keep XCTest for UI and performance tests. Don't mix the two APIs within one test ([XCTest](https://developer.apple.com/documentation/xctest)). (H)
- **Swift 6.4** ([Swift 6.4](https://www.swift.org/blog/swift-6.4-released/), [Xcode 27 notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27-release-notes)):
  - `XCTAssert` works inside Swift Testing tests, and `#expect` works inside XCTests (ST-0021).
  - `swift test --maximum-repetitions N --repeat-until fail` repeats tests to hunt flakes.
- **Xcode 27.3 beta** adds UI tests written with Swift Testing and `XCUIAutomation` ([27.3](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_3-release-notes)). It's still beta.
- **Policy.** Swift Testing everywhere except UI tests. Parameterized tests for money and budget math. Inject `Clock`, UUIDs, `Locale` and `TimeZone`.

### Snapshot tests

- **Library.** swift-snapshot-testing 1.19.7 (2026-10-21). Swift Testing suites opt in with `@Suite(.snapshots(record: .failed))`. Record modes are `all`, `failed`, `missing` and `never`, also settable with the `SNAPSHOT_TESTING_RECORD` environment variable ([source](https://github.com/pointfreeco/swift-snapshot-testing/blob/main/Sources/SnapshotTesting/AssertSnapshot.swift)).
- **Caveat.** "Snapshots must be compared using the exact same simulator that originally took the reference" ([README](https://github.com/pointfreeco/swift-snapshot-testing)).
- **In CI.**
  - Set `TEST_RUNNER_SNAPSHOT_TESTING_RECORD=never`; xcodebuild only passes `TEST_RUNNER_`-prefixed variables through to tests ([Apple env vars](https://developer.apple.com/documentation/xcode/environment-variable-reference)).
  - Pin one device and OS in the test plan.
  - Record references on the same pinned simulator and OS that CI uses, for example with a `workflow_dispatch` job on the hosted runner that uploads them as an artifact.

### UI tests

- **Scope.** A handful of smoke flows with XCTest and XCUIAutomation, run on main and nightly rather than on every PR.
- **New in Xcode 27.** Test plans can set how severe a target-app crash is ([notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27-release-notes)).
- **Agents** check UI ad hoc through the Xcode MCP (see section 4) instead of growing brittle UI suites.

### Coverage gates

- **No native gate.** GitHub's coverage ruleset rule needs GitHub Code Quality, which is Team or Enterprise only ([rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets), [Code Quality](https://docs.github.com/en/code-security/concepts/code-quality/code-quality)). (H)
- **DIY gate.**
  - Collect coverage with `swift test --enable-code-coverage` or `xcodebuild -enableCodeCoverage YES -resultBundlePath R.xcresult`.
  - Export it as JSON with `xcrun xccov view --report --json`, or with xcsift 1.7.4, which converts `.profraw` and `.xcresult` files ([xcsift](https://github.com/ldomaradzki/xcsift)).
  - Check the floor with `jq`: for example at least 85 % line coverage on the domain packages (blocking), report-only for UI code.
- **Optional.** Codecov's Developer plan is free for 1 user, with 250 private uploads, PR comments and status checks ([pricing](https://about.codecov.io/pricing/)).

### Dead code: Periphery

- **Status.** Periphery has become a commercial product. The OSS repo is archived under MIT, with 3.8.0 (2026-07-25) as its last release. Small indie, hobby and open-source projects stay free, and the open-beta license is free ([README](https://github.com/peripheryapp/periphery), [announcement](https://periphery.pro/a-new-chapter)).
- **Usage.** Run it weekly as advisory: `periphery scan --strict --retain-codable-properties`. After `swift test`, add `--skip-build --index-store-path` ([README](https://github.com/peripheryapp/periphery); `--strict` and `--baseline` are in the [3.8 source](https://github.com/peripheryapp/periphery/blob/master/Sources/Frontend/Commands/ScanCommand.swift)). (M)

### Dependency updates for SPM

- **Dependabot** ([ecosystems](https://docs.github.com/en/code-security/dependabot/ecosystems-supported-by-dependabot/supported-ecosystems-and-repositories)):
  - Its `swift` ecosystem covers Swift 5 and 6, with version and security updates. Private registries must be git; non-declarative manifests aren't supported.
  - It also covers `github-actions`, `docker` and `npm`.
  - It's free and involves no third party, but it has no `mise` support.
  - TypeScript caveats: `bun` gets version updates but no security updates, like `docker`. When pnpm 12 writes `pnpm-lock.yaml` as two YAML documents (its default when a project pins a pnpm version), "Dependabot alerts and Dependabot security updates are not created for those dependencies". (H)
- **Renovate 44.222.2:**
  - Managers for `swift` (`Package.swift`), `xcodegen`, `mise`, `github-actions`, `dockerfile`, `docker-compose` and `npm` ([managers](https://docs.renovatebot.com/modules/manager/)).
  - Mend Community Cloud is free, private repos included ([hosted](https://docs.renovatebot.com/mend-hosted/overview/)), but it gives a third party access to the repo.
- **Neither documents** updating packages declared inside an `.xcodeproj`. So declare every remote dependency in a local `Package.swift`, and have the app project reference only local packages.
- **Pick Dependabot.** Add Renovate, self-hosted in Actions, only if `mise.toml` tool pins need a bot. (M)

### Secrets: GitHub native scanning and gitleaks

- **GitHub.** On public repos, secret scanning "runs automatically for free", and push protection is free too ([secret scanning](https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning), [features by repo type](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security)). Both are on for this repo (checked 2026-11-07). "Push protection for users", on by default, also blocks the owner's pushes of secrets to public repos ([push protection](https://docs.github.com/en/code-security/secret-scanning/introduction/about-push-protection)). If the repo goes private, **user-owned private repos get no secret scanning**: organization private repos need Secret Protection, and user-owned repos get it only with Enterprise Managed Users. (H)
- **gitleaks 8.30.1.**
  - Pre-commit hook: `gitleaks git --pre-commit --redact --staged --verbose` ([hooks file](https://github.com/gitleaks/gitleaks/blob/master/.pre-commit-hooks.yaml)).
  - CI: a full-history `gitleaks git` scan.
  - gitleaks-action needs no license on personal accounts; repos owned by an organization need a free license key ([action](https://github.com/gitleaks/gitleaks-action)).

### CodeQL: free on the public repo, once the code has an OSI license

- **Support.** It handles Swift 5.4 to 6.3, on macOS only, so Swift 6.4 (Xcode 27) isn't listed yet. TypeScript 2.6 to 7.0 is covered ([languages](https://codeql.github.com/docs/codeql-overview/supported-languages-and-frameworks/)).
- **Public repo.** Code scanning, the CodeQL CLI and Copilot Autofix come free with public repos ([features by repo type](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security)). But the CLI license allows automated analysis in CI only on an "Open Source Codebase", one "released under an OSI-approved License" ([license](https://github.com/github/codeql-cli-binaries/blob/main/LICENSE.md)), and the license is undecided (`PRODUCT.md`). (H)
- **If the repo goes private.**
  - Code scanning on private repos needs a GitHub Code Security license, which only Team and Enterprise plans can buy ([code scanning](https://docs.github.com/en/code-security/code-scanning/introduction-to-code-scanning/about-code-scanning), [features by repo type](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security)).
  - The CodeQL CLI license forbids use on non-open-source code without GitHub Advanced Security ([license](https://github.com/github/codeql-cli-binaries/blob/main/LICENSE.md)). (H)

### Pre-commit hooks: lefthook or pre-commit

- **The options.** lefthook 2.2.2 is a single Go binary that runs commands in parallel and supports `{staged_files}` ([README](https://github.com/evilmartians/lefthook)). pre-commit 4.7.0 needs Python but has a large hook ecosystem.
- **Pick lefthook**, installed through `mise`:
  - pre-commit: SwiftFormat on staged files, SwiftLint, gitleaks.
  - commit-msg: the Conventional Commit check.
- **Agents can skip hooks with `--no-verify`.** So CI re-runs everything, and Claude Code denies `Bash(git commit --no-verify *)` ([permissions](https://code.claude.com/docs/en/permissions)).

### Commits, branches and rulesets for a solo developer

- **Commits.** Squash-merge only. The PR title must be a Conventional Commit (`feat(budget): …`), checked in CI. Ruleset commit-metadata rules are Enterprise-only ([about rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)).
- **Plan gate.** "Rulesets are available in public repositories with GitHub Free", and on private repos with Pro, Team and Enterprise Cloud; protected branches follow the same rule ([about rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets), [protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)). So the public repo needs no upgrade: `GET branches/main/protection` returns 404 "Branch not protected" (checked 2026-11-07), where the private repo got 403 "Upgrade to GitHub Pro" (reported, no longer checkable). (H)
- **Ruleset on `main`.**
  - Require a PR, with 0 approvals.
  - Require the `gate` status check, in strict mode.
  - Require linear history.
  - Block force pushes and restrict deletions.
  - Require conversations to be resolved.
  - **No bypass actors.** Agents use the owner's `gh` token, so any owner bypass is also an agent bypass.
- **Keep agents out of repo administration.** That token (classic `repo` scope, checked locally) carries repo admin, and admins "can create, edit, and delete rulesets"; the REST update and delete endpoints need only "Administration" write ([rules API](https://docs.github.com/en/rest/repos/rules)). So an agent can switch the gate off with `gh api`. Give agents a fine-grained token on this repo without Administration. Denying `gh api` ruleset calls in agent permissions is a weaker second layer, since Claude Code's Bash rules aren't "a security boundary around the program" ([permissions](https://code.claude.com/docs/en/permissions)). (H)
- **No merge queue.** It needs a public repo owned by an organization, or a private one on Enterprise Cloud ([merge queue](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue)). Moving this public repo to a free organization would unlock it; gitleaks-action would then need its free license key.
- **Same price as Pro, if the repo goes private.** A free organization on GitHub Team costs $4 per user a month ([pricing](https://github.com/pricing)) and gets the same private-repo rulesets. It can also buy Secret Protection ($19) or Code Security ($30) per active committer a month ([security plans](https://github.com/security/plans)), and Code Quality, which has its own per-committer license ([Code Quality](https://docs.github.com/en/code-security/concepts/code-quality/code-quality)). A merge queue on a private repo still needs Enterprise Cloud. (H)
- **Path filters can block merges.** A workflow skipped by path filters leaves its required check *Pending*, which blocks the merge. So the only required check is a `gate` job that always runs, using `if: always()` with `needs` ([troubleshooting](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks)). (H)

### Backend equivalents

- **TypeScript.**
  - Biome 2.6.6 for format and lint.
  - `tsc --noEmit` with `strict` on.
  - Vitest.
  - knip 6.44.2 for dead code and unused dependencies.
  - Dependabot for npm.
- **Swift.**
  - The same SwiftFormat, SwiftLint and Swift 6 settings, running on Linux.
  - A server framework: Hummingbird 2.28.1 or Vapor 4.122.3 (Vapor 5 is in beta).
  - Periphery, which supports SPM projects on Linux ([README](https://github.com/peripheryapp/periphery)).
- **Both.** A Postgres service container in CI, squawk for migrations, a `/up` health endpoint, and gitleaks.

## 4. Agentic development for Swift

### Xcode's built-in agents

- **Xcode 26.3** added agentic coding with Anthropic's Claude Agent and OpenAI's Codex. It also exposed Xcode's capabilities over the **Model Context Protocol**, with fine-grained permissions ([26.3 notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-26_3-release-notes)).
- **Xcode 27** ships Swift 6.4 and needs macOS 26.6 or later on Apple silicon ([27 notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27-release-notes)). New for agents:
  - Gemini and Antigravity join the provider list, and ACP agents can be added.
  - Plan mode, with plans as editable Markdown.
  - Plugins that bundle skills, MCP servers and ACP agents.
  - Agents can boot simulators, tap and take screenshots.
  - MCP tools for the debugger, schemes, build settings and entitlements, plus preview-snapshot variants.
  - A filesystem security layer for agents.
  - `lldb-mcp`, an MCP server for the debugger.
  - A preview, `sudo xcrun mcp-server enable`, that runs the MCP server without an open workspace; `--unsafe-always-allow-all-agents` is meant for unattended machines.
- **Versions.** 27.2 is current. 27.3 beta adds GitHub Copilot ([27.3](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_3-release-notes)). (H)
- **Setup.** Xcode › Settings › Intelligence › Agents › Get, then sign in ([setup](https://developer.apple.com/documentation/xcode/setting-up-coding-intelligence)).
- **Separate config.** Agents launched inside Xcode read their config from `~/Library/Developer/Xcode/CodingAssistant/{ClaudeAgentConfig,codex,gemini}`, not from `~/.claude` ([customizing](https://developer.apple.com/documentation/xcode/extending-and-customizing-agents)).

### Xcode MCP for external agents (Claude Code and Codex in T3 Code)

- **Setup** ([external agents](https://developer.apple.com/documentation/xcode/giving-external-agents-access-to-xcode)). (H)
  - In Intelligence settings, set "Allow External Agents to Use Xcode Tools" to Always or While Xcode is Open.
  - Register the bridge: `claude mcp add --transport stdio xcode -- xcrun mcpbridge`, or `codex mcp add xcode -- xcrun mcpbridge`.
  - **Open the project in Xcode** before prompting.
- **Use it** for interactive checks: previews, simulator, debugger. Not for CI, which has no GUI session and can't answer permission prompts.

### MobileBuildMCP (formerly XcodeBuildMCP)

- **What it is.** v2.7.1 (2026-09-23), maintained by Sentry: an MCP server plus a CLI. Install it with `brew install mobilebuildmcp` (from the `getsentry/xcodebuildmcp` tap) or with npm ([README](https://github.com/getsentry/XcodeBuildMCP)).
- **Setup.** `claude mcp add MobileBuildMCP -- npx -y mobilebuildmcp@latest mcp` ([clients](https://github.com/getsentry/xcodebuildmcp.com/blob/main/app/docs/_content/clients.mdx)). Config lives in `.mobilebuildmcp/config.yaml`, where `enabledWorkflows` picks the tool groups.
- **Behavior.** It skips macro validation.
- **Telemetry.** Sentry error telemetry is on by default. Turn it off with `MOBILEBUILDMCP_SENTRY_DISABLED=true` ([privacy](https://github.com/getsentry/xcodebuildmcp.com/blob/main/app/docs/_content/privacy.mdx)).
- **When to use it.** When Xcode isn't open. Use one MCP per task to save context. (M)

### CLI feedback loop shared by agents and CI

- **Packages.** `swift build` and `swift test --filter …`. Swift Testing is on by default in SwiftPM since Xcode 16 ([16 notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-16-release-notes)).
- **Apps.** `xcodebuild test -scheme App -destination 'platform=iOS Simulator,name=…,OS=27.2' -derivedDataPath .build/xcode -resultBundlePath …`, piped into `xcsift` (JSON or TOON output, GitHub annotations, coverage) or into xcbeautify 3.2.1.
- **Simulators.** `xcrun simctl create|boot|install|launch|io booted screenshot`. `reboot` is new in 27.
- **CI failures.** `gh run view --log-failed`.

### Making the codebase agent-friendly

- **Package layout.** Put the code in `Packages/` (now only the watch app's Swift code; see Status):
  - `BudgetCore`: the domain, Foundation only.
  - `Persistence`.
  - `APIClient`.
  - The UI and feature modules.

  App targets stay thin, and `swift test` runs on macOS in seconds with no simulator. Keeping `BudgetCore` Linux-clean is optional: it gives cheap Linux CI and lets a Swift backend reuse it.
- **One entry point.** `mise run fmt|lint|test:fast|test:app|check`, used by hooks, agents and CI. The `mise` registry already has swiftformat, swiftlint, gitleaks, lefthook, xcbeautify, xcsift, periphery, git-cliff and xcodes ([mise](https://github.com/jdx/mise); checked with `mise registry`, 2026.11.1).
- **Determinism.** Inject clocks and IDs. No network in unit tests. One fixed snapshot simulator. Repeat runs to find flakes.
- **Parallel worktrees.** T3 Code runs agents in parallel worktrees, so give each worktree its own DerivedData (`-derivedDataPath`) and its own simulator.
- **Project format.**
  - Xcode 27.2's **JSON `.xcproj`** is smaller, "merge-friendly", easier for agents to edit, and an open format (Apache-2.0). Apple says merge conflicts become "less likely", not impossible. It opens in Xcode 27.0 and later ("Projects using .xcproj also open in earlier versions of Xcode 27"), not in Xcode 26 ([doc](https://developer.apple.com/documentation/xcode/updating-your-xcode-project-configuration-file-format), [27.2 notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_2-release-notes), [format repo](https://github.com/apple/xcode-project-format)). (H)
  - Tooling lags. CocoaPods' Xcodeproj gem, which CocoaPods and fastlane use, has an open request to support it ([#1052](https://github.com/CocoaPods/Xcodeproj/issues/1052)), as do fastlane's versioning actions ([#30323](https://github.com/fastlane/fastlane/issues/30323)) and XcodeGen ([#1653](https://github.com/yonaskolb/XcodeGen/issues/1653)). React Native 0.88 still builds iOS with CocoaPods by default; Swift Package Manager is opt-in ([0.88](https://reactnative.dev/blog/2026/10/19/react-native-0.88)). So adopt `.xcproj` only once the chosen stack's native tooling reads it, and keep the fastlane fallback in mind.
  - Use buildable folders as well; they keep file lists out of the project file (Xcode 16) ([16 notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-16-release-notes)).
  - Skip XcodeGen and Tuist unless `.xcproj` disappoints.
- **AGENTS.md.**
  - Claude Code (v2.1.277 or later) reads AGENTS.md when there is no CLAUDE.md. Otherwise it reads it through `@AGENTS.md`, which is the pattern this repo already uses ([memory](https://code.claude.com/docs/en/memory)).
  - Codex reads AGENTS.md files from the repo root down to the working directory, up to a combined **32 KiB** (`project_doc_max_bytes`) ([Codex](https://developers.openai.com/codex/guides/agents-md)).
  - Keep the root file short: commands, layout, and invariants (money as `Decimal` or minor units, tests required with every change). Put a nested AGENTS.md in each package.

### Agent hooks

- **Claude Code** ([hooks](https://code.claude.com/docs/en/hooks)). (H)
  - A `PostToolUse` hook on `Edit|Write` formats the edited file. Exit code 2 shows its stderr to Claude, but it can't block, because the tool has already run.
  - A `Stop` hook runs the fast checks. Exit code 2 makes Claude keep working. The input's `stop_hook_active` is true when Claude is already continuing because of a stop hook; check it "to avoid blocking on a condition that will never resolve". Claude Code also caps stop hooks at 8 consecutive continuations, but the count "resets each time Claude calls a tool", so a test that keeps failing can loop for a long time. `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` raises the cap.
  - A `PostToolUse` hook matching `Edit|Write` doesn't run when a Bash command rewrites the same file. `FileChanged` doesn't fill that gap: it watches only literal filenames (or `watchPaths`), has no decision control, and on exit 2 it shows stderr to the user only, not to Claude. Instead, add a matcher-less `PostToolUse` hook that runs `git status --porcelain`, as the docs suggest, or read `tool_response.bashEditDiff` on Bash calls (v2.1.269 or later, public beta, best effort, meant "to find what to review, not to enforce a policy"). The Stop hook and CI stay the backstop.
- **Codex** ([Codex hooks](https://developers.openai.com/codex/hooks)). (H)
  - Hooks are on by default (`[features].hooks`). They live in `hooks.json` or in `[hooks]` tables of `config.toml`, under `~/.codex` or the repo's `.codex/`.
  - Project hooks load only once the project is trusted, and new or changed hooks need review in `/hooks`.
  - The `PostToolUse` matcher is `apply_patch|Edit|Write`.
  - `Stop` keeps Codex going on exit code 2 or on `{"decision":"block"}`.

```json
{ "hooks": {
  "PostToolUse": [{ "matcher": "Edit|Write", "hooks": [{ "type": "command",
      "command": "${CLAUDE_PROJECT_DIR}/scripts/agent-format.sh", "onFailure": "block" }] }],
  "Stop": [{ "hooks": [{ "type": "command",
      "command": "${CLAUDE_PROJECT_DIR}/scripts/agent-stop.sh", "timeout": 300 }] }] } }
```

```sh
#!/bin/sh
# scripts/agent-stop.sh: one retry, then let Claude stop; CI is the real gate.
if jq -e '.stop_hook_active' >/dev/null; then exit 0; fi
mise run test:fast >&2 || exit 2
```

Two ways feed a failure back to Claude: exit code 2 with stderr, or exit 0 with a JSON `{"decision": "block", "reason": "…"}`; on `PostToolUse` the reason is added next to the tool result. Since v2.1.295, `"onFailure": "block"` makes any other failure (a crash, exit 1, a timeout) do what exit 2 does, except on `Stop`, `SubagentStop`, `TaskCompleted` and `TeammateIdle` hooks, so the Stop wrapper must still exit 2 itself. (H)

### Known pitfalls

- **Xcode 27 lock-in.** `.xcproj` opens only in Xcode 27.0 and later, so it rules out the `macos-26` image and Xcode 26. Tools that parse project files do lag: CocoaPods' Xcodeproj gem, fastlane and XcodeGen have open issues (see Project format).
- **Simulators.**
  - In Xcode 27.2, cloning a simulator on runtime 27.2 can fail, and parallel testing clones simulators. A workaround is in the [27.2 notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_2-release-notes).
  - In 27.1, the first simulator launch can take minutes ([27.1](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_1-release-notes)).
- **Duplicate module names.** Xcode 27's dependency scanner fails when two Clang modules share a name, as with vendored module maps.
- **Build system change.** SwiftPM 6.4 now uses Swift Build by default ([6.4](https://www.swift.org/blog/swift-6.4-released/)). Compare its results with Xcode builds.
- **Snapshots.** References depend on the exact simulator and OS. Updates to Xcode or the runner image can change rendering, so re-record deliberately in a dedicated PR.
- **Hosted runner memory.** Hosted macOS runners have 7 GB of RAM; Xcode plus a simulator can swap.
- **Xcode 26.3 known issue.** If you deny an agent access to a project under Desktop, Documents or Downloads, you can't undo it ([26.3](https://developer.apple.com/documentation/xcode-release-notes/xcode-26_3-release-notes)). A repository outside those folders is unaffected.
- **Bypasses.** Agents skip local hooks and act with the owner's token, which can also delete rulesets. The real gate is CI plus rulesets with no bypass, and only while the agents' token lacks Administration.
- **Truncation.** Codex silently truncates AGENTS.md content beyond 32 KiB.

## Recommended pipeline

| Stage | Trigger | Runner | Jobs | Blocking |
|---|---|---|---|---|
| **PR** | `pull_request`, with concurrency `cancel-in-progress` | Linux (`ubuntu-24.04`) | `hygiene`: gitleaks diff scan, `swiftformat --lint`, `swiftlint --strict`, Conventional-Commit PR title, `Package.resolved` drift check | yes |
| | | Linux | `backend` (if `api/**` changed): lint, typecheck, tests with a Postgres service, squawk, `docker build` | yes |
| | | Hosted macOS (`xcode-27`, or `macos-26`) | `apple` (if Swift or Xcode files changed): `swift test` on packages with coverage floor; `xcodebuild test` for iOS (unit + snapshot, pinned simulator) and macOS; Debug builds of both apps | yes |
| | | Linux, plus macOS for native shells | `ui` (shared UI and web app): to be defined once the UI stack is chosen, then added to `gate`'s `needs` | yes |
| | | Linux | `gate`: `if: always()`, `needs` all of the above, fails on any `failure` or `cancelled` | **the only required check** |
| **main** | `push` | same | Everything above, plus UI smoke tests, coverage baseline as an artifact, backend image push (superseded by ADR 0003) | UI tests advisory at first |
| **Release** | tag `vX.Y.Z`, at least every 90 days | Xcode Cloud | Archive iOS and macOS, TestFlight: internal automatically, an external group only if needed | yes |
| | tag `vX.Y.Z` | Linux | git-cliff release notes, GitHub Release | no |
| **API deploy** (superseded by ADR 0003) | tag `api-v*` or dispatch | Linux + Tailscale | `kamal deploy`, `/up` check, smoke test; on failure `kamal rollback` | yes |
| **Scheduled** | nightly or weekly | Hosted macOS and Linux | Full UI suite, test repetition for flakes, Periphery, full-history gitleaks; Dependabot weekly | advisory |

```yaml
gate:
  if: always()
  needs: [hygiene, backend, apple]
  runs-on: ubuntu-24.04
  steps:
    - if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')
      run: exit 1
```

**Monthly cost estimate.** Assumptions: 150 PR runs, 60 main runs, 8 releases, 20 API deploys, 30 scheduled runs. Minutes per run are guesses, to measure in phase 2. The public-repo column is the current case.

| Item | Minutes | Public repo | If private |
|---|---|---|---|
| Linux: (150 + 60) × 6, plus 20 × 8, plus 30 × 3 | ≈ 1,510 | **$0** (standard runners free) | ≈ $9 at $0.006, inside Pro's $18 → $0 |
| macOS on hosted runners: 150 × 12, plus 60 × 15, plus 30 × 20 | ≈ 3,300 | **$0** (standard runners free) | ≈ $205 at $0.062; ≈ $167 for PR + main only |
| Or PR + main on Xcode Cloud (no always-on Mac, private repo) | ≈ 2,700 to 4,200 compute minutes (45 to 70 h) | not needed | 100-hour tier, **US$49.99** |
| Xcode Cloud: 8 × 2 archives × 15 | ≈ 240 (4 h of 25) | **$0** (included) | **$0** (included) |
| GitHub Pro | none | not needed | **$4** ([FAQ](https://docs.github.com/en/get-started/learning-about-github/faq-about-changes-to-githubs-plans)) |
| Apple Developer Program | none | $99/year, already needed for TestFlight | same |

- **Budget cap.** If the repo goes private, set a GitHub Actions budget of about $10. That's about 160 hosted macOS minutes.

## Guard-rail list

| Guard rail | Tool (version 2026-11-07) | Where it runs | Blocking? |
|---|---|---|---|
| Swift formatting | SwiftFormat 0.64.6 (`--lint` in CI) | agent PostToolUse hook, lefthook pre-commit, CI Linux | blocking |
| Swift lint | SwiftLint 0.65.1 `--strict` | lefthook, CI Linux | blocking |
| Compiler strictness | Xcode 27.2 / Swift 6.4; Swift 6 mode; `treatAllWarnings(as: .error)` | everywhere | blocking |
| Unit tests (packages) | Swift Testing via `swift test` | agent Stop hook, CI macOS | blocking |
| App tests | `xcodebuild test` (Swift Testing + XCTest) | CI macOS | blocking |
| Snapshot tests | swift-snapshot-testing 1.19.7, record mode `never` | CI macOS | blocking once stable |
| UI smoke tests | XCTest / XCUIAutomation | main, nightly | advisory, then blocking before release |
| Coverage | xccov / llvm-cov JSON or xcsift 1.7.4 + `jq` floor | CI macOS | blocking for `BudgetCore`, advisory elsewhere |
| Dead code | Periphery 3.8.0 OSS (or Pro beta) | weekly | advisory |
| Secrets | gitleaks 8.30.1; GitHub secret scanning and push protection (free on the public repo) | lefthook pre-commit, CI (diff + nightly full history), GitHub | blocking |
| Code scanning | CodeQL default setup (TypeScript; Swift up to 6.3) | GitHub, free on the public repo | once the code has an OSI license |
| Dependencies | Dependabot (`swift`, `github-actions`, `docker`, `npm`; no security updates for `docker`, `bun` or pnpm 12 two-document lockfiles) | GitHub | PRs; auto-merge patches when green |
| Actions supply chain | SHA-pinning setting + `actions.lock` | GitHub | blocking |
| Commit convention | Conventional Commit PR-title check; lefthook commit-msg | CI Linux, local | blocking |
| Branch protection | Ruleset on `main` (PR required, `gate`, linear history, no force push or delete, no bypass) | GitHub (Free while public; Pro if private) | blocking, while agents can't administer the repo |
| Agent permissions | Fine-grained agent token without Administration; Claude Code deny rules (`--no-verify`, `push … main`, `gh pr merge --admin`, `gh api` ruleset calls); Codex project hooks | GitHub, local | token blocking; deny rules best effort |
| Shared UI and web app | Not planned yet: lint, typecheck, unit, offline and E2E tests, web build and deploy | CI | pending the UI-stack choice |
| Backend (TypeScript) | Biome 2.6.6, `tsc --noEmit` strict, Vitest, knip 6.44.2 | CI Linux | blocking (knip advisory) |
| Backend (Swift) | Same Swift tools on Linux; Hummingbird 2.28.1 or Vapor 4.122.3 | CI Linux | blocking |
| Migrations | squawk 2.71.0; expand/contract only | CI Linux | blocking |
| Deploy safety | Kamal 2.12.0 health check + `kamal rollback` (superseded by ADR 0003); tag-restricted environment | deploy job | blocking |

## Setup order

1. **Phase 0: owner-only prerequisites (about 1 h).**
   - Stay on GitHub Free while the repo is public; upgrade to Pro only if it goes private.
   - Set fork PR workflows to "Require approval for all external contributors". Keep secret scanning and push protection on (both on today).
   - Create a fine-grained token for agents, limited to this repo, without Administration.
   - Join the Apple Developer Program and create the app records.
   - Install Xcode 27.2 and the iOS 27.2 simulator runtime.
   - Create the Tailscale OAuth client and `tag:ci`, if CI still deploys after the ADR 0003 re-plan.
2. **Phase 1: local loop, so agents can start.**
   - `mise.toml` pins.
   - `.swiftformat` and `.swiftlint.yml`.
   - The `Packages/` layout with strict `swiftSettings`.
   - The Xcode project, in `.xcproj` once the stack's native tooling reads it.
   - The `mise run` tasks.
   - lefthook.
   - AGENTS.md "Commands" and "Invariants" sections.
   - Claude Code and Codex hooks and deny rules.
   - Xcode MCP registration.
3. **Phase 2: PR CI.**
   - Linux `hygiene` job and `gate`.
   - The `apple` job on hosted macOS.
   - The ruleset requiring `gate`.
   - SHA pinning and the actions lockfile.
   - `dependabot.yml`, and turn on Dependabot security updates (off today).
   - Measure job durations to replace the cost guesses.
4. **Phase 3: release.**
   - Xcode Cloud workflow on tag `v*`: archive iOS and macOS, send to TestFlight.
   - TestFlight groups: internal, plus an external group only for household members without an App Store Connect role. Release at least every 90 days.
   - Generate `WhatToTest`.
   - git-cliff and the GitHub Release job.
5. **Phase 4: backend (deploy superseded by ADR 0003; re-plan).**
   - Dockerfile and `/up` endpoint.
   - Kamal config and the Tailscale action.
   - The `production` environment.
   - squawk, the migration step, and `pg_dump` backups.
6. **Phase 5: hardening.**
   - Snapshot suite and coverage floors.
   - UI smoke tests on main.
   - Weekly Periphery.
   - Nightly flake-repetition runs, and the budget alert if the repo goes private.

## Open questions and spikes

**Decisions for the owner**

1. **GitHub plan.** Answered: Free. While the repo is public, Free covers the ruleset; Pro ($4) or a Team organization ($4 per user) matters only if it goes private. Moving the public repo to a free organization would add the merge queue.
2. **Mac.** Answered: no always-on Mac. With the public repo, PR checks run on hosted macOS runners; if the repo goes private, see the options in question 1's recommendation.
3. **Minimum Xcode.** Do we require Xcode 27 or iOS 27? That decides `.xcproj` and whether the hosted runner can be the GA `macos-26`.
4. **Backend language.** Swift (shared domain package, one toolchain) or TypeScript (Biome and knip ecosystem)?
5. **Apple's GitHub app.** Read access is moot now that the repo is public; the app's permission set wasn't re-checked.
6. **License.** CodeQL in CI needs an OSI-approved license on the code; the license is undecided (`PRODUCT.md`).

**Spikes, 1 to 2 hours each**

1. **Runner user** (only if the repo goes private). A self-hosted runner under its own user: launchd setup, simulator and UI tests outside the console session, disk hygiene for DerivedData and simulators.
2. **Universal purchase.** One app record for iOS and macOS, with Xcode Cloud build numbers. Check the Mac rule that build numbers always increase.
3. **Sandbox.** Mac App Store apps must be sandboxed (Guideline 2.4.5(i)), and Mac TestFlight builds most likely too (inference). Does the app, in the chosen UI stack, fit inside it? If not, plan the Developer ID path.
4. **Developer ID.** Headless Developer ID cloud signing with an API key is a known Apple bug (FB16835802), so plan on a local Developer ID certificate, or `match`, for direct distribution. Test whether Xcode Cloud's Notarize post-action works for this team, given the 2026 failure reports.
5. **`.xcproj` support.** Do Periphery's Xcode mode and the chosen UI stack's native tooling understand it? CocoaPods' Xcodeproj gem and fastlane don't yet (open issues). Does `xcodes` install `.aea` archives?
6. **Package.resolved.** Do Dependabot's `Package.swift` updates stay in sync with the app's resolved file? Add a CI check that `xcodebuild -resolvePackageDependencies` leaves no diff.
7. **Periphery.** Does OSS 3.8.0 work with Swift 6.4, or should we take a Pro (free tier) license?

**Could not verify**

- Whether GitHub has set a new date for the self-hosted platform charge. GitHub's own page (2025-12-15) says it's postponed, and the billing docs list self-hosted runners as free.
- GitHub Pro's 2026 price. $4 comes from a plan-change FAQ.
- Xcode's build-setting identifiers for Approachable Concurrency and Default Actor Isolation. We give only the UI names.
- That Mac TestFlight builds must be sandboxed like Mac App Store apps. Inferred from the shared App Store Connect upload.
- That cloud signing needs an Admin-role API key. Only developer reports say so.
- Whether Apple has fixed Developer ID cloud signing with an API key (FB16835802), and whether Xcode Cloud's Developer ID exports fail as the 2026 forum reports say.
- When XcodeBuildMCP was renamed to MobileBuildMCP. The README already uses the new name.
- Whether the `xcrun mcp-server` preview (27 beta 5) has changed in 27.2.
- All per-job minute estimates.

## Sources

**GitHub**
- [Actions runner pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing)
- [Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions)
- [2026 Actions pricing changes (self-hosted charge postponed)](https://github.com/resources/insights/2026-pricing-changes-for-github-actions)
- [Pricing](https://github.com/pricing), [Security plans](https://github.com/security/plans)
- [Plans](https://docs.github.com/en/get-started/learning-about-github/githubs-plans)
- [Plan-change FAQ ($4 Pro)](https://docs.github.com/en/get-started/learning-about-github/faq-about-changes-to-githubs-plans)
- [Limits](https://docs.github.com/en/actions/reference/limits)
- [Hosted runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)
- [Secure use](https://docs.github.com/en/actions/reference/security/secure-use)
- [Actions settings (SHA pinning, fork PR approvals)](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository)
- [Workflow lockfiles](https://docs.github.com/en/actions/concepts/security/workflow-dependency-lockfiles)
- [Contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)
- [Environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
- [About rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
- [Available rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets)
- [Rulesets REST API](https://docs.github.com/en/rest/repos/rules)
- [Protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [Merge queue](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue)
- [Required checks troubleshooting](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks)
- [Secret scanning](https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning)
- [Push protection](https://docs.github.com/en/code-security/secret-scanning/introduction/about-push-protection)
- [Code scanning](https://docs.github.com/en/code-security/code-scanning/introduction-to-code-scanning/about-code-scanning)
- [Security features by repo type](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security)
- [Code Quality](https://docs.github.com/en/code-security/concepts/code-quality/code-quality)
- [Dependabot ecosystems](https://docs.github.com/en/code-security/dependabot/ecosystems-supported-by-dependabot/supported-ecosystems-and-repositories)
- [runner-images](https://github.com/actions/runner-images)
- [macos-26 image](https://github.com/actions/runner-images/blob/main/images/macos/macos-26-arm64-Readme.md)
- [xcode-27 image](https://github.com/actions/runner-images/blob/main/images/macos/xcode-27-arm64-Readme.md)
- [#14404](https://github.com/actions/runner-images/issues/14404)
- [#15226](https://github.com/actions/runner-images/issues/15226)
- [actions/runner](https://github.com/actions/runner)
- Secondary: [Techzine on the self-hosted fee](https://www.techzine.eu/news/devops/137396/github-bends-to-criticism-and-delays-paid-self-hosting-of-runners/)

**Apple**
- Release notes: [Xcode 27](https://developer.apple.com/documentation/xcode-release-notes/xcode-27-release-notes), [27.1](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_1-release-notes), [27.2](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_2-release-notes), [27.3 beta](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_3-release-notes), [26.3](https://developer.apple.com/documentation/xcode-release-notes/xcode-26_3-release-notes), [16](https://developer.apple.com/documentation/xcode-release-notes/xcode-16-release-notes), [13](https://developer.apple.com/documentation/xcode-release-notes/xcode-13-release-notes)
- Xcode Cloud: [overview and pricing](https://developer.apple.com/xcode-cloud/), [actions](https://developer.apple.com/documentation/xcode/configuring-your-xcode-cloud-workflow-s-actions), [distribution workflow](https://developer.apple.com/documentation/xcode/creating-a-workflow-that-builds-your-app-for-distribution), [project setup](https://developer.apple.com/documentation/xcode/setting-up-your-project-to-use-xcode-cloud), [dependencies](https://developer.apple.com/documentation/xcode/making-dependencies-available-to-xcode-cloud), [scripts](https://developer.apple.com/documentation/xcode/writing-custom-build-scripts), [env vars](https://developer.apple.com/documentation/xcode/environment-variable-reference), [tester notes](https://developer.apple.com/documentation/xcode/including-notes-for-testers-with-a-beta-release-of-your-app), [build numbers](https://developer.apple.com/documentation/xcode/setting-the-next-build-number-for-xcode-cloud-builds), [merge requirements](https://developer.apple.com/documentation/xcode/configuring-requirements-for-merging-a-pull-request)
- Coding intelligence: [setup](https://developer.apple.com/documentation/xcode/setting-up-coding-intelligence), [external agents](https://developer.apple.com/documentation/xcode/giving-external-agents-access-to-xcode), [customizing agents](https://developer.apple.com/documentation/xcode/extending-and-customizing-agents)
- Project format: [`.xcproj`](https://developer.apple.com/documentation/xcode/updating-your-xcode-project-configuration-file-format), [xcode-project-format](https://github.com/apple/xcode-project-format)
- App Store Connect: [upcoming requirements](https://developer.apple.com/news/upcoming-requirements/), [upload builds](https://developer.apple.com/help/app-store-connect/manage-builds/upload-builds), [Build Uploads API](https://developer.apple.com/documentation/appstoreconnectapi/post-v1-builduploads), [API 4.1 notes](https://developer.apple.com/documentation/appstoreconnectapi/app-store-connect-api-4-1-release-notes), [betaBuildLocalizations](https://developer.apple.com/documentation/appstoreconnectapi/betabuildlocalizationcreaterequest/data-data.dictionary/attributes-data.dictionary), [TestFlight](https://developer.apple.com/testflight/), [TestFlight overview](https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview), [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/), [App Sandbox](https://developer.apple.com/documentation/xcode/configuring-the-macos-app-sandbox)
- Signing and notarization: [cloud-managed certs](https://developer.apple.com/help/account/certificates/cloud-managed-certificates), [roles](https://developer.apple.com/help/account/access/roles/), [API keys](https://developer.apple.com/documentation/appstoreconnectapi/creating-api-keys-for-app-store-connect-api), [notarizing](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution), [notarization workflow](https://developer.apple.com/documentation/security/customizing-the-notarization-workflow), [TN3147](https://developer.apple.com/documentation/technotes/tn3147-migrating-to-the-latest-notarization-tool)
- Bundle keys and testing: [CFBundleVersion](https://developer.apple.com/documentation/bundleresources/information-property-list/cfbundleversion), [CFBundleShortVersionString](https://developer.apple.com/documentation/bundleresources/information-property-list/cfbundleshortversionstring), [XCTest](https://developer.apple.com/documentation/xctest)
- Secondary, developer forum reports: [698117 (Admin key)](https://developer.apple.com/forums/thread/698117), [776036 (DTS: known bug)](https://developer.apple.com/forums/thread/776036), [853905](https://developer.apple.com/forums/thread/853905) and [855013](https://developer.apple.com/forums/thread/855013) (Xcode Cloud Developer ID exports)

**Swift and tools**
- Swift: [Swift 6.4](https://www.swift.org/blog/swift-6.4-released/), [Swift 6.2](https://www.swift.org/blog/swift-6.2-released/), [SE-0441](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0441-formalize-language-mode-terminology.md), [SE-0461](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md), [SE-0466](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)
- Format, lint, dead code: [swift-format](https://github.com/swiftlang/swift-format), [SwiftFormat](https://github.com/nicklockwood/SwiftFormat), [SwiftLint](https://github.com/realm/SwiftLint), [SwiftLintPlugins](https://github.com/SimplyDanny/SwiftLintPlugins), [Periphery](https://github.com/peripheryapp/periphery), [Periphery Pro](https://periphery.pro/a-new-chapter)
- Testing and coverage: [swift-snapshot-testing](https://github.com/pointfreeco/swift-snapshot-testing), [Codecov pricing](https://about.codecov.io/pricing/)
- Secrets and code scanning: [gitleaks](https://github.com/gitleaks/gitleaks), [gitleaks-action](https://github.com/gitleaks/gitleaks-action), [CodeQL languages](https://codeql.github.com/docs/codeql-overview/supported-languages-and-frameworks/), [CodeQL license](https://github.com/github/codeql-cli-binaries/blob/main/LICENSE.md)
- Hooks and tool management: [lefthook](https://github.com/evilmartians/lefthook), [pre-commit](https://github.com/pre-commit/pre-commit), [git-cliff](https://github.com/orhun/git-cliff), [mise](https://github.com/jdx/mise)
- Dependency bots: [Renovate managers](https://docs.renovatebot.com/modules/manager/), [Renovate hosted](https://docs.renovatebot.com/mend-hosted/overview/)
- Build tooling: [xcsift](https://github.com/ldomaradzki/xcsift), [xcbeautify](https://github.com/cpisciotta/xcbeautify), [xcodes](https://github.com/XcodesOrg/xcodes), [Tart](https://github.com/cirruslabs/tart), [Tart licensing](https://tart.run/licensing/)
- MobileBuildMCP: [repo](https://github.com/getsentry/XcodeBuildMCP), [clients](https://github.com/getsentry/xcodebuildmcp.com/blob/main/app/docs/_content/clients.mdx), [privacy](https://github.com/getsentry/xcodebuildmcp.com/blob/main/app/docs/_content/privacy.mdx)
- fastlane: [match](https://docs.fastlane.tools/actions/match/), [upload_to_testflight](https://docs.fastlane.tools/actions/upload_to_testflight/)
- `.xcproj` tool support: [Xcodeproj #1052](https://github.com/CocoaPods/Xcodeproj/issues/1052), [fastlane #30323](https://github.com/fastlane/fastlane/issues/30323), [XcodeGen #1653](https://github.com/yonaskolb/XcodeGen/issues/1653), [React Native 0.88](https://reactnative.dev/blog/2026/10/19/react-native-0.88)
- Other CI services: [Codemagic](https://codemagic.io/pricing/), [Bitrise](https://bitrise.io/pricing)

**Repo decisions**
- [ADR 0001](../adr/0001-self-hosted-single-owner-instances.md) (public repo, self-hosted), [ADR 0002](../adr/0002-one-ui-codebase-native-watch.md) (one UI codebase, native watch), [ADR 0003](../adr/0003-nixos-just-tailscale-infrastructure.md) (NixOS, `just`, Tailscale)

**Backend**
- Kamal: [proxy](https://kamal-deploy.org/docs/configuration/proxy/), [rollback](https://kamal-deploy.org/docs/commands/rollback/), [hooks](https://kamal-deploy.org/docs/hooks/overview/), [registry](https://kamal-deploy.org/docs/configuration/docker-registry/)
- Deploy and migrations: [Tailscale action](https://github.com/tailscale/github-action), [squawk](https://github.com/sbdchd/squawk)
- TypeScript tools: [Biome](https://github.com/biomejs/biome), [knip](https://github.com/webpro-nl/knip)
- Swift servers: [Vapor](https://github.com/vapor/vapor), [Hummingbird](https://github.com/hummingbird-project/hummingbird)

**Agents**
- Claude Code: [hooks](https://code.claude.com/docs/en/hooks), [memory and AGENTS.md](https://code.claude.com/docs/en/memory), [permissions](https://code.claude.com/docs/en/permissions)
- Codex: [hooks](https://developers.openai.com/codex/hooks), [AGENTS.md](https://developers.openai.com/codex/guides/agents-md)
