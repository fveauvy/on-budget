# Server deploys from a release tag without GitHub Pro

> Research note, 2026-11-08, for ticket #12. Plans, limits and versions as of that day, read from GitHub Docs, Tailscale's docs, the tools' tagged sources and their release pages. Confidence: **H** high, **M** medium, **L** low. "Judgment" marks my own assessment, "sketch" code written for this note, "measured" a check run for this note. Builds on [ADR 0003](../adr/0003-nixos-just-tailscale-infrastructure.md), [ADR 0018](../adr/0018-forward-only-upgrades-with-a-client-support-window.md), [`docs/architecture.md`](../architecture.md#the-reference-deployment) (The reference deployment), [`docs/factory.md`](../factory.md) (CI/CD) and [nixos-vps-infra.md](nixos-vps-infra.md), without editing them.

## Question

How can a release tag deploy the server to the tailnet-only NixOS host automatically, without GitHub Pro on the private ops repository? The ticket compares a protected environment on the public repository, a pull-based host and GitHub Pro, on credential exposure and supply-chain risk, rollback, ADR 0018's schema floor, observability and effort. ADR 0003 and `factory.md` assume CI deploys need Pro "for environment secrets and a `main`-only rule".

## Answer

**Deploy from the ops repository's own CI on GitHub Free, with no stored deploy secret** (option D below; judgment, on **H** facts).

- **Why Pro looked necessary.** On Free, every writer can read a private repository's secrets, and environments exist only on public repositories (F1, F2). Pro was there to keep a stored, root-equivalent key away from workflows on other branches. With no stored key, there is nothing to keep away.
- **Keyless access.** The deploy job's GitHub OIDC token is exchanged, through Tailscale's workload identity federation, for an ephemeral `tag:deploy` node. The federated identity matches the token's `sub` (`repo:<owner>/<ops>:ref:refs/heads/main`) and its `workflow_ref` (the deploy workflow on `main`). Tailscale then refuses a run from another branch, a pull request or another workflow (F9, F10). That claim match does the work of an environment's `main`-only rule.
- **SSH.** Tailscale SSH (`tag:deploy` to `tag:prod` as `deploy`, accept) needs no key, but it takes port 22 on the tailnet address (F11). The owner's touch-gated OpenSSH would then move to a second tailnet-only port. The fallback is a deploy key stored as an ops repository secret, which every ops writer can read.
- **Trigger.** An hourly scheduled job reads the public repository's latest release and checks it: newer than the pin, a release-please commit on `main`. It bumps only the Shoestring input, runs the host VM test and commits with `GITHUB_TOKEN`. Then deploy-rs deploys with magic rollback. The checks cost about $1.50 of the $12 of compute Free includes each month (F4).
- **Floor.** When the new release's floor is above the running version, the job deploys with `--magic-rollback false --auto-rollback false`, as ADR 0018 requires. Required reviewers could hold that deploy instead, but private repositories don't get them, even on Pro (F1).
- **Rejected.** The public-repository environment would show every deploy log to anyone signed in to GitHub (F6), and it puts a key to the private ops repository into the public repository. comin has no automatic rollback and follows branches, not tags (F15). Pro adds $4 a month and nothing this design needs.
- **Found on the way.** factory.md binds the `release` environment to `v*` tags as well as `main`. A tag push runs the workflow files of the tagged commit (F5), so any writer can reach that environment's secrets, the agents' machine account included. Those secrets include the Cachix token, which publishes what the host substitutes (F17). Binding `release` to `main` only and building in release-please's run on `main` closes that path (F8).

## Facts

### GitHub plans, secrets and events

- **F1 (H).** On GitHub Free, environments can be configured only on public repositories. In private repositories, environment secrets, deployment branch and tag rules, and environment variables need Pro or Team. On Free, Pro and Team, required reviewers, wait timers, custom protection rules and the admin-bypass switch work only on public repositories. A deployment branch rule is matched against the run's `GITHUB_REF`. A workflow that names a missing environment creates it, unprotected, but only admins configure environments.
- **F2 (H).** "Any user with write access to your repository has read access to all secrets configured in your repository." Redaction of secrets in logs "is not guaranteed".
- **F3 (M on price, H on features).** GitHub Pro costs $4 a month according to GitHub's plan FAQ. github.com/pricing now lists only Free, Team ($4 per user per month) and Enterprise ($21). In private repositories, Pro adds protected branches, required pull-request reviewers and code owners, and its included Actions compute is $18 a month instead of Free's $12.
- **F4 (H).** Private repositories pay for runners out of the included compute: Linux 1-core costs $0.002 a minute and 2-core $0.006, and each job is rounded up to a whole minute. Standard runners are free on public repositories. `ubuntu-slim` is a 1-CPU unprivileged container, so it has no KVM, and its jobs stop after 15 minutes.
- **F5 (H).** A tag push runs the workflows at the pushed commit, "including workflows that are not merged into the default branch". `schedule`, `workflow_run` and `repository_dispatch` runs use the default branch's workflow file, and their `GITHUB_REF` is the default branch. Schedules run at most every 5 minutes, can be delayed or dropped at load peaks such as the top of the hour, and notify whoever last edited the cron line. A push made with `GITHUB_TOKEN` starts no workflow run; only `workflow_dispatch` and `repository_dispatch` events from that token do.
- **F6 (H).** Anyone signed in to GitHub can read a public repository's workflow runs and logs.
- **F7 (H).** A deploy key reaches one repository, is read-only by default and never expires.
- **F8 (H).** An immutable release locks its tag to the commit. It comes with a release attestation from GitHub, covering the tag, commit SHA and assets, which `gh release verify` checks. release-please makes lightweight tags with no signature of their own (measured: its own `v17.11.2` tag points straight at a commit). release-please-action v5.0.0 outputs `release_created`, `tag_name` and `version` in its run on `main`.

### OIDC and Tailscale

- **F9 (H; M for private repositories on Free).** GitHub OIDC's `sub` claim depends on the job:
  - `repo:OWNER/REPO:environment:NAME` when the job references an environment;
  - `repo:OWNER/REPO:pull_request` for pull-request events;
  - otherwise `repo:OWNER/REPO:ref:refs/heads/BRANCH`, or `…:ref:refs/tags/TAG` for a tag.

  Other claims include `workflow_ref`, `job_workflow_ref`, `job_workflow_sha`, `event_name`, `repository_visibility` and `runner_environment`. A repository can change its `sub` template through the REST API. Repositories created after 2026-07-15, or opted in, get an immutable `sub` that carries owner and repository IDs. The docs set no plan limit, and their sample token comes from a private repository.
- **F10 (H).** In Tailscale's workload identity federation, a tailnet admin sets an issuer, a subject, optional custom claim rules, and the scopes and tags to grant. Values match exactly, except for `*`. The client ID and audience aren't secrets. tailscale/github-action v4.4.0 uses federation through `oauth-client-id`, `audience` and `tags`, with `id-token: write` and Tailscale 1.90.1 or later. Its nodes are ephemeral and preapproved, and its `authkey` input is deprecated.
- **F11 (H).** Tailscale SSH takes port 22 on the Tailscale address, and no other port can be set. Rules from a tagged source can only accept: check mode isn't allowed. Restarting tailscaled ends its sessions. Tailscale SSH is available on every plan.
- **F12 (H; M on federation).** Tailscale's free Personal plan allows up to 6 users, 50 tagged resources and 1,000 minutes of ephemeral nodes a month. An ephemeral node leaves the tailnet shortly after going offline, or at once on `tailscale logout`. Neither the pricing page nor the federation guide limits federation to a paid plan.

### Deploy tools

- **F13 (H).** deploy-rs v1.0.0 (2026-10-19) is the version nixpkgs ships on `nixos-unstable` and 26.05.
  - Magic rollback reconnects after activation, and the node reverts unless that confirmation arrives within `confirmTimeout`, 30 s by default.
  - `autoRollback`, on by default, reverts a failed activation.
  - `--magic-rollback`, `--auto-rollback`, `--confirm-timeout`, `--remote-build` and `--log-dir` override these settings for one run.
  - The activator ignores SIGHUP, to "ensure that this process stays alive after the SSH connection dies".
  - A rollback runs the previous generation's `deploy-rs-activate`, so it fails when that generation wasn't deployed by deploy-rs (#86, still open).
  - `--remote-build` copies only the derivation over `ssh-ng` and builds on the host. A closure built on the runner must be signed (`LOCAL_KEY`) before a non-root user can copy it.
- **F14 (H).** nixpkgs restarts tailscaled once, at the end of activation (`stopIfChanged = false`), because "activation over Tailscale can hang" otherwise.
- **F15 (H).** comin v0.14.0 (2026-07-19) comes from its own flake, not nixpkgs.
  - It polls every 60 s and builds on the host.
  - It deploys the `main` branch or a `testing-<hostname>` branch, never a tag, and refuses force-pushed `main` commits.
  - It checks the GPG or SSH signature of the last commit only.
  - It can `switch`, `test` or `boot`.
  - A post-deployment script receives `COMIN_STATUS`, and a Prometheus exporter is built in.
  - Neither its features nor its options include an automatic rollback.
  - Deleting its state file lets an attacker roll `main` back to an older signed commit.
- **F16 (H).** NixOS `system.autoUpgrade` has neither a health check nor a rollback.
- **F17 (H).** Unless a cache was created with a self-generated key, Cachix signs pushed paths with a key it manages, and per-cache tokens grant write access.

## The options

### A. A protected environment on the public repository

How it works: a `deploy` job in the public repository references an environment limited to `main`, and Tailscale's federated identity matches `repo:<owner>/shoestring:environment:deploy`. The job clones the ops repository with a deploy key (or a GitHub App token) stored in that environment, then runs deploy-rs.

- **Credentials and supply chain.** Environments are free on a public repository (F1). The job must not run on tags: with a `v*` rule, any writer could tag a branch commit and run their own workflow inside the environment (F1, F5), the machine account included. So the job runs in release-please's run on `main` (F8) or in a `workflow_run` workflow (F5). Only code merged to `main`, which the maintainer reviews, then reaches the environment. Stored: the ops deploy key (F7), plus an SSH key unless Tailscale SSH is used. The public repository, which takes outside pull requests, becomes part of the host's trust root.
- **Privacy.** Anyone signed in can read the logs (F6). They would show node and host names, store paths named after the host, deploy-rs output, the smoke test's URL, and evaluation errors that quote the private configuration. AGENTS.md keeps home infrastructure and the instance address out of the repository. A silenced job (`--log-dir` to a file never uploaded, `log-mode: quiet`, pass or fail only) avoids the leak, but then every useful log is on the host.
- **Rollback.** Magic rollback works from the runner (F13).
- **Floor.** This option handles it best: required reviewers are free here (F1), so a second environment can hold floor-raising deploys for one click.
- **Observability.** GitHub keeps the environment's deployment history, but the logs must say almost nothing.
- **Effort.** Medium: two repositories, a cross-repository key and log hygiene.

### B. A pull-based host

How it works: comin on the host follows the ops repository's `main`, and the host holds a read-only deploy key (F7). For a release to deploy, an ops workflow, or the maintainer, bumps the Shoestring input and pushes to `main` with `GITHUB_TOKEN`. In the systemd timer variant, the host would build and switch on its own.

- **Credentials and supply chain.** This is the option's strongest point: CI holds no credential to the host. But the host can't follow signed release tags: release-please's tags carry no signature, and comin follows branches (F8, F15). comin can require the maintainer's signature on `main`, which keeps a touch but makes every bump a manual step. Or it can accept a bot's key, stored as an ops repository secret that any ops writer can read (F2): the stored root credential again.
- **Rollback.** There is none. comin has no automatic rollback (F15), and neither has NixOS's autoUpgrade (F16). A switch that breaks Tailscale or outbound traffic leaves a host that can't fetch its own fix, so recovery goes through the provider's console. comin's `test` operation leaves the bootloader alone, so a reboot reverts, but again only from the console. The timer variant could probe and switch back, but that would be new code holding root.
- **Floor.** Deploys only move forward, so nothing needs switching off. A failed floor-raising release stays failed, as ADR 0018 expects.
- **Observability.** `comin status`, the Prometheus exporter (the reference deployment runs no Prometheus), and a post-deployment script that pings Healthchecks with `COMIN_STATUS` (F15); logs go to journald.
- **Effort.** Medium for comin: its flake, and the host builds every commit. High for the timer.

### C. GitHub Pro on the ops repository

How it works: nixos-vps-infra.md's design, with a `production` environment limited to `main` that holds the secrets, and deploy-rs over Tailscale.

- **Credentials.** Environment secrets reach only jobs on `main`, and protected branches stop direct pushes by writers who aren't admins (F1, F3). But the ops repository has no such writer, and the maintainer, an admin, can bypass the protection.
- **Rollback.** Magic rollback.
- **Floor.** Even on Pro, required reviewers work only on public repositories (F1). A floor-raising release either deploys with rollbacks off or waits for `just deploy`, as in D.
- **Observability.** Full logs, kept private, and a deployment history.
- **Effort.** Low, for $4 a month (F3).

### D. The ops repository's own CI on Free, keyless (recommended)

How it works: as C, without environments. Tailscale's claim match replaces the `main`-only rule, and no secret is stored.

- **Credentials.**
  - Nothing can be stolen ahead of time. GitHub mints an OIDC token for each job, and Tailscale admits only the deploy workflow on `main` (F9, F10).
  - A run from another branch or a pull request carries a different `sub`, and another workflow carries a different `workflow_ref`, so Tailscale refuses both.
  - Only the deploy job gets `id-token: write`. With Tailscale SSH, no SSH key exists either (F11).
  - Exposure: whoever can push to the ops repository's `main`, which Free can't protect (F3). That means the maintainer alone, as long as the machine account isn't a collaborator there.
- **Privacy.** Logs stay in the private repository.
- **Rollback.**
  - Magic rollback runs from the runner, as `just deploy` does today. If the runner disappears or the network breaks, the host reverts after the 60–120 s confirm timeout (F13).
  - Activation survives the end of the SSH session (F13), and tailscaled restarts only at the end of activation (F14).
  - Before deploying, the job checks that the running generation carries `deploy-rs-activate`, since a rollback onto a break-glass generation fails (F13, judgment).
- **Floor.**
  - The job compares the new release's floor with the version `/api/v1/meta` reports. When the floor is higher, it deploys with both rollbacks off (F13, ADR 0018).
  - Doing that unattended is acceptable, because the release path changes only the Shoestring input, and the host VM test has just run on it (judgment). nixpkgs moves in the weekly update PR, with rollbacks on.
  - The cautious alternative: stop, notify the maintainer and leave that release to `just deploy`.
- **Observability.**
  - Full logs stay private, and the maintainer gets notified when a scheduled run fails (F5).
  - The job checks `/api/v1/meta` through `app.<domain>` and pings a Healthchecks check.
  - `just deploy` stays as the manual and break-glass path.
- **Effort.** Low to medium: one federated identity, one grant, one workflow, the floor flag, and a VM test of the SSH path.
- **Cost.** Hourly checks run about 730 one-minute `ubuntu-slim` jobs a month: $1.46. Checks every 15 minutes would cost $5.84. Each release adds one host VM test and one deploy on 2-core runners (F4). On Tailscale's side, this stays within the Personal plan's 1,000 ephemeral minutes (F12).

## Comparison

| | A. Public environment | B. Pull (comin) | C. Pro | D. Free, keyless |
|---|---|---|---|---|
| Plan cost | $0 | $0 | $4 a month | $0 |
| Stored credential reaching the host | Ops deploy key, and an SSH key, in the public repository | Ops read key on the host | Environment secrets | None, with Tailscale SSH |
| Who can deploy their own code | Whoever merges to public `main` | Whoever pushes to ops `main`, or holds an allowed signing key | Whoever pushes to ops `main` | Whoever pushes to ops `main` |
| Deploy logs | Public | Host journal | Private | Private |
| Automatic rollback | Magic rollback | None | Magic rollback | Magic rollback |
| Floor-raising release | Held for one click | Forward only | Rollbacks off, or `just deploy` | Rollbacks off, or `just deploy` |
| Effort | Medium | Medium; timer: high | Low | Low to medium |

## Common to every option

- **No manual step means no touch.** Today, a deploy needs the maintainer's touch-gated SSH key. Every unattended option gives that power to whoever can change the deployed configuration. An agent running under the maintainer's macOS user can reach the maintainer's GitHub login (factory.md, Identity) and push to the ops repository. So run agents that hold only the machine token, and keep the machine account out of the ops repository.

  Optional hardening (judgment, unverified):
  - move the deploy steps into a reusable workflow, and match Tailscale on its `job_workflow_sha` (F9), so that changing those steps takes an edit in the Tailscale console;
  - have those steps deploy only commits the maintainer signed, or bumps that change nothing but the lock entry, to a verified release.
- **The release is what gets deployed.** Whichever option deploys, the host substitutes Shoestring from Cachix, and Cachix's write token can publish paths the host trusts (F17). That token sits in the `release` environment, which factory.md binds to `v*` tags. Any writer can therefore reach the token by tagging a branch commit (F1, F5). The check that "the tag points at a release-please commit on `main`" runs inside the very workflow it is meant to guard. Building and pushing in release-please's run on `main` (F8), with `release` bound to `main` only, removes that path; the tag still names the release.
- **Which release to deploy.** The machine account can create a release too. So the ops job deploys a release only if:
  - it is newer than the pinned one;
  - its commit is on the public `main`;
  - that commit's `.release-please-manifest.json` names the same version.

  With immutable releases on, `gh release verify` adds GitHub's attestation (F8).
- **TestFlight.** A `just testflight` step after the deploy would need the App Store Connect key as an ops repository secret, which every ops writer can read (F2). That is out of scope here.

## Recommended design (sketch)

- **Tailscale.**
  - A federated identity with GitHub as issuer and `repo:<owner>/<ops>:ref:refs/heads/main` as subject, or its immutable form (F9).
  - Custom claims: `workflow_ref` = `<owner>/<ops>/.github/workflows/deploy.yml@refs/heads/main` and `runner_environment` = `github-hosted`.
  - Scope `auth_keys`, tag `tag:deploy`.
  - Grants: `tag:deploy` to `tag:prod` on 22 and 443, plus an SSH rule letting `tag:deploy` reach `tag:prod` as `deploy`, accept. These replace the `tag:ci` rule that architecture.md plans for later.
- **Host.**
  - `services.tailscale.extraSetFlags = [ "--ssh" ]`, and a `deploy` user with sudo for activation.
  - The owner keeps OpenSSH on a second port opened only on `tailscale0`, or moves to Tailscale SSH check mode; decide in #20.
  - A VM test that logs in over both paths.
- **deploy-rs node.** `sshUser = "deploy"`, `user = "root"`, `remoteBuild = true` (so no signing key is needed), and `confirmTimeout` between 60 and 120 s.
- **Workflow**, below. A merge of the weekly update PR deploys through the same workflow, minus the bump.

```yaml
# Sketch: .github/workflows/deploy.yml in the private ops repository
on:
  schedule: [{ cron: "23 * * * *" }] # hourly, away from the top of the hour (F5)
  push: { branches: [main] }         # the maintainer's merges; GITHUB_TOKEN pushes start no run
  workflow_dispatch: {}
permissions: {}
concurrency: { group: deploy, cancel-in-progress: false }
jobs:
  plan:   # ubuntu-slim, contents: read
    # latest Shoestring release: newer than the pin? commit on public main? manifest matches?
    # outputs: the tag, and whether its schema floor is above the running version
  bump:   # ubuntu-24.04 (KVM), contents: write; only for a new release
    # point the shoestring input at the tag, lock only that input, run the host VM test,
    # then commit and push to main with GITHUB_TOKEN
  deploy: # ubuntu-24.04, contents: read and id-token: write
    steps:
      - uses: tailscale/github-action@<full SHA> # v4.4.0
        with:
          oauth-client-id: <client ID>  # not a secret (F10)
          audience: <audience>
          tags: tag:deploy
          log-mode: quiet
      - run: just deploy-ci # checks deploy-rs-activate, then deploy-rs; rollbacks off when the floor rises
      - run: just smoke     # /api/v1/meta reports the release; Healthchecks ping
```

## Doc changes this implies

- **ADR 0003, Consequences.** Unattended deploys no longer need Pro: they run from the ops repository's CI through workload identity federation.
- **factory.md.**
  - Deploys: the scheduled deploy workflow.
  - Costs: drop the Pro row.
  - Repository settings: `release` bound to `main`, with the build in release-please's run.
  - The flow diagram.
- **architecture.md, The reference deployment.**
  - Members' access: `tag:deploy` instead of `tag:ci`.
  - Builds and deploys: CI deploys with magic rollback, the floor rule and the `deploy-rs-activate` check.
  - Exposure: the SSH port split, if chosen.
- **ADR 0018** stays as it is: its rule becomes the job's flag.

## Spikes and unverified

- **S1, federation (half a day).**
  - A deploy job in a private repository on Free joins a Personal tailnet through a federated identity.
  - Record the exact `sub` the repository emits, default or immutable.
  - Check that Tailscale refuses another branch, a `pull_request` run and another `workflow_ref`.
- **S2, the deploy path (half a day, with nixos-vps-infra.md's S1).**
  - deploy-rs v1.0.0 with `--remote-build`, as the untrusted `deploy` user over Tailscale SSH.
  - Magic rollback after a deliberate network break, and after a tailscaled restart, within 60–120 s.
  - The owner's second SSH port.
- **S3, release checks.** Immutable releases on the releases release-please creates, and whether their tags lock as soon as they are published; `gh release verify` on `ubuntu-slim`; and the floor flag in the package.
- **Unverified.**
  - Pro's price beyond the plan FAQ.
  - Federation on the Personal plan.
  - KVM on 2-core private runners, which factory.md assumes.
  - Deploy downtime.
  - Whether the `ubuntu-slim` image ships `gh`.

## Versions checked 2026-11-08

Each version is the newest release on its project's release page; none has a newer release candidate.

- tailscale/github-action v4.4.0 (2026-10-29), whose default Tailscale is 1.104.4;
- Tailscale 1.104.5 (2026-11-03);
- deploy-rs 1.0.0 (2026-10-19);
- comin v0.14.0 (2026-07-19);
- Nix 2.35.4;
- NixOS 26.05, with 26.11 due at the end of November (nixpkgs #567567);
- gh 2.103.3 (2026-10-26);
- cachix/install-nix-action v31.11.3 (2026-10-27);
- release-please 17.11.2 (2026-08-24) and release-please-action v5.0.0 (2026-04-22);
- actions/create-github-app-token v3.3.0 (2026-10-12), only for option A's app-token variant.

## Sources

- **GitHub environments and secrets:**
  - [Deployments and environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
  - [Managing environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments)
  - [Secrets](https://docs.github.com/en/actions/concepts/security/secrets)
  - [Secure use](https://docs.github.com/en/actions/reference/security/secure-use)
  - [Managing deploy keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)
- **GitHub plans and billing:**
  - [GitHub's plans](https://docs.github.com/en/get-started/learning-about-github/githubs-plans)
  - [Plan changes FAQ](https://docs.github.com/en/get-started/learning-about-github/faq-about-changes-to-githubs-plans)
  - [Pricing](https://github.com/pricing)
  - [Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions)
  - [Runner pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing)
  - [Hosted runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)
- **GitHub events, logs and OIDC:**
  - [Events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)
  - [Workflow run logs](https://docs.github.com/en/actions/how-tos/monitor-workflows/use-workflow-run-logs)
  - [OIDC reference](https://docs.github.com/en/actions/reference/security/oidc)
  - [OIDC concepts](https://docs.github.com/en/actions/concepts/security/openid-connect)
  - [OIDC REST API](https://docs.github.com/en/rest/actions/oidc)
- **Releases and signatures:**
  - [Immutable releases](https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases)
  - [Enabling them](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/establish-provenance-and-integrity/prevent-release-changes)
  - [Verifying a release](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/verify-release-integrity)
  - [`gh release verify` at v2.103.3](https://github.com/cli/cli/blob/v2.103.3/pkg/cmd/release/verify/verify.go)
  - [Commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification)
  - [release-please-action v5.0.0 README](https://github.com/googleapis/release-please-action/blob/v5.0.0/README.md)
  - [release-please v17.11.2 tag ref](https://api.github.com/repos/googleapis/release-please/git/ref/tags/v17.11.2)
- **Tailscale:**
  - [github-action v4.4.0](https://github.com/tailscale/github-action/releases/tag/v4.4.0)
  - [its action.yml](https://github.com/tailscale/github-action/blob/v4.4.0/action.yml)
  - [its README](https://github.com/tailscale/github-action/blob/v4.4.0/README.md)
  - [Workload identity federation](https://tailscale.com/docs/features/workload-identity-federation)
  - [Trust credentials](https://tailscale.com/docs/reference/trust-credentials)
  - [Tailscale SSH](https://tailscale.com/docs/features/tailscale-ssh)
  - [Ephemeral nodes](https://tailscale.com/docs/features/ephemeral-nodes)
  - [GitHub CI/CD guide](https://tailscale.com/docs/solutions/connect-github-cicd-workflows-to-private-infrastructure-without-public-exposure)
  - [Pricing](https://tailscale.com/pricing)
  - [Client releases](https://github.com/tailscale/tailscale/releases)
- **deploy-rs:**
  - [v1.0.0](https://github.com/serokell/deploy-rs/releases/tag/v1.0.0)
  - [README](https://github.com/serokell/deploy-rs/blob/v1.0.0/README.md)
  - [cli.rs](https://github.com/serokell/deploy-rs/blob/v1.0.0/src/cli.rs)
  - [activate.rs](https://github.com/serokell/deploy-rs/blob/v1.0.0/src/bin/activate.rs)
  - [push.rs](https://github.com/serokell/deploy-rs/blob/v1.0.0/src/push.rs)
  - [#86](https://github.com/serokell/deploy-rs/issues/86)
- **nixpkgs:**
  - [deploy-rs package](https://github.com/NixOS/nixpkgs/blob/nixos-unstable/pkgs/by-name/de/deploy-rs/package.nix)
  - [tailscale module](https://github.com/NixOS/nixpkgs/blob/nixos-unstable/nixos/modules/services/networking/tailscale.nix)
  - [auto-upgrade module](https://github.com/NixOS/nixpkgs/blob/nixos-unstable/nixos/modules/tasks/auto-upgrade.nix)
  - [26.11 schedule, #567567](https://github.com/NixOS/nixpkgs/issues/567567)
  - [Nix tags](https://github.com/NixOS/nix/tags)
- **comin:**
  - [v0.14.0](https://github.com/nlewo/comin/releases/tag/v0.14.0)
  - [features](https://github.com/nlewo/comin/blob/v0.14.0/docs/features.md)
  - [design](https://github.com/nlewo/comin/blob/v0.14.0/docs/design.md)
  - [howtos](https://github.com/nlewo/comin/blob/v0.14.0/docs/howtos.md)
  - [module options](https://github.com/nlewo/comin/blob/v0.14.0/docs/generated-module-options.md)
  - [release notes](https://github.com/nlewo/comin/blob/v0.14.0/docs/release-notes.md)
- **Cachix:** [Getting started, auth tokens and signing keys](https://docs.cachix.org/getting-started).
- **This repository:**
  - [ADR 0003](../adr/0003-nixos-just-tailscale-infrastructure.md)
  - [ADR 0018](../adr/0018-forward-only-upgrades-with-a-client-support-window.md)
  - [architecture.md](../architecture.md#the-reference-deployment)
  - [factory.md](../factory.md)
  - [nixos-vps-infra.md](nixos-vps-infra.md)
