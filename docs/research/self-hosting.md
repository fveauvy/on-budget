# Self-hosting on-budget: packaging, upgrades and operations

> **Status (2026-11-08).** Background research. The ADRs, `SPEC.md`, [`docs/factory.md`](../factory.md) and [`docs/architecture.md`](../architecture.md#references) win where this note disagrees. It uses on-budget, the working name, for what is now Shoestring ([ADR 0020](../adr/0020-shoestring-as-the-product-name.md)). Superseded or amended here: Household roles; compose in v1; channels; refusing newer schemas; age-encrypted dumps; open license and relay; two PostgreSQL majors.

Research note, 2026-11-07. Versions, prices and availability are as of November 2026. It builds on [sync-engine.md](sync-engine.md), [web-client-sync.md](web-client-sync.md), [apple-client-stack.md](apple-client-stack.md), [bank-aggregation.md](bank-aggregation.md), [bank-alternatives.md](bank-alternatives.md), [connectors-and-license.md](connectors-and-license.md), [factory-ci-quality.md](factory-ci-quality.md), [nixos-vps-infra.md](nixos-vps-infra.md) and [native-apps-auth.md](native-apps-auth.md), and on ADRs [0001](../adr/0001-self-hosted-single-owner-instances.md) and [0003](../adr/0003-nixos-just-tailscale-infrastructure.md), and edits none of them. The older notes assumed one hosted instance for about 20 friends and family. Here, each instance belongs to one owner who runs it for their own household, and ADR 0003 makes the reference deployment "reachable only over Tailscale". "Secondary" marks third-party sources, and "judgment" marks my own assessment. Confidence levels are high, medium or low. The Paperless-ngx and Firefly III doc sites refused automated fetching, so I read those pages from the projects' own repositories. Enable Banking's terms page renders in JavaScript, so I read its text in the site's app bundle.

## Question

How should on-budget package and distribute its server: the API, the bundled web app, background jobs and the PostgreSQL schema? And what does a self-hoster need to install, expose, administer, back up and upgrade it safely, while offline iOS, macOS and web clients on other versions keep syncing?

## TL;DR

**Recommendation: one flake, two install paths, two containers, and no "first visitor becomes admin".**

1. **Artifacts** (medium-high).
   - One `flake.nix` builds the server package (API, web app assets, CLI and SQL migrations) and exports `nixosModules.default` (`services.on-budget`).
   - It also builds the OCI image with nixpkgs `dockerTools.buildLayeredImage`, natively on amd64 and arm64 runners. CI pushes the image to `ghcr.io/fveauvy/on-budget` by digest, joins the two into a multi-arch index and attests it. The image serves compose installs: the reference deployment needs no registry (ADR 0003). GHCR packages start private, so the package is made public once.
   - Every GitHub Release attaches a `compose.yaml` and `.env.example` pinned to that release's digest.
   - The runtime is on-budget plus vanilla PostgreSQL 18, and 19 once it ships (planned for 2026-11-19) and passes CI: no Redis, no custom Postgres image, and jobs run on Postgres. The image bundles the `pg_dump` of each supported major. PowerSync, if adopted, adds a service, bucket storage and a second hostname (P7).
2. **Upgrades** (medium-high).
   - Versions follow SemVer. Migrations run forward-only at start, under an advisory lock, after an automatic pre-migration dump.
   - The server refuses to start on a schema newer than itself, as Gitea does. Rolling back means the previous binary plus that dump.
   - The server accepts clients from the current and previous minor release, for at least 90 days. Apps carry a minimum server version.
3. **Backups** (medium).
   - Built in: an hourly `pg_dump` plus a dump of the roles, encrypted to the owner's `age` key, pre-upgrade dumps, and a versioned per-household export.
   - Retention is by age: 48 h of hourly dumps, then 14 days of daily ones. Pre-upgrade dumps are kept apart, so a burst of upgrades can't push scheduled ones out. Immich instead keeps the last 14 dumps by count, and its manual dumps count toward that limit.
   - The recovery point objective (RPO) is up to one hour; daily dumps would mean up to 24 h. PostgreSQL warns that "except in simple cases, pg_dump is generally not the right choice" for regular production backups; a few-MB household database restored whole is such a simple case (judgment). WAL archiving stays the upgrade path.
   - Off-site copies and restore drills stay with the self-hoster, and the docs give the commands.
   - A restore rotates an instance epoch: clients re-bootstrap and re-push their pending mutations, so `/push` must accept one ID gap per client. Writes the server acknowledged after the dump are lost, unless clients keep them until a dump covers them. Both amend sync-engine.md (§4).
4. **Administration** (medium-high).
   - There is no sign-up endpoint. `on-budget admin bootstrap`, run on the host, prints a one-time owner enrollment link valid for 1 hour.
   - The owner creates households (for example "test" and "production") and single-use invites. Routine tasks go in the web admin UI; break-glass recovery goes in the CLI.
5. **Security defaults** (medium-high).
   - The server refuses to start without generated secrets or an `https` public URL, and reads secrets from files.
   - Enable Banking keys are generated on the server, so people paste only a certificate into the provider's console. Members other than the owner wait for EB's written OK, because its terms forbid giving third parties access to the API (§7). Keys and tokens are envelope-encrypted with a key kept outside the database.
   - The app listens on loopback behind a TLS proxy. By default nothing is open to the internet (ADR 0003): Tailscale, with a `*.ts.net` certificate or a DNS-01 certificate for the owner's own name. A public instance opens only 443, plus 80 for ACME.
   - Rate limits and an append-only audit log are on by default.
6. **What one instance can't do alone** (medium).
   - Native passkeys need the instance's domain in the app's associated domains. APNs keys stay private to the app's publisher, including the per-bundle-ID keys Apple added in 2025.
   - So native apps sign in through the instance's web page in `ASWebAuthenticationSession`, as Nextcloud's iOS app does (merged in February 2026). native-apps-auth.md develops this sign-in design.
   - Push goes through Web Push or the self-hoster's own APNs key. Running a push relay, as Bitwarden does, is an owner decision.

## 1. Reference projects (November 2026)

**A. Packaging and releases**

| Project (licence) | Latest stable | Versioning, channels | Images and tags | Install artifact, runtime |
|---|---|---|---|---|
| Actual Budget (MIT) | v26.11.0, 2026-11-02 ([releases](https://github.com/actualbudget/actual/releases)) | CalVer, monthly; `nightly` from `master` | Docker Hub `actualbudget/actual-server`, GHCR `ghcr.io/actualbudget/actual`; `latest`, `latest-alpine`, `nightly` ([docs](https://actualbudget.org/docs/install/docker)) | Compose in the repo; one Node container with `tini`. The Dockerfile creates a non-root user but sets no `USER`. Healthcheck `node scripts/health-check.js` every 60 s ([compose](https://github.com/actualbudget/actual/blob/master/packages/sync-server/docker-compose.yml), [Dockerfile](https://github.com/actualbudget/actual/blob/master/sync-server.Dockerfile)) |
| Firefly III (AGPL-3.0) | v6.7.9, 2026-10-11 | SemVer; `develop-YYYYMMDD` pre-releases | `fireflyiii/core:latest` | App, MariaDB and an Alpine cron container calling `/api/v1/cron/<token>`; app published on `80` ([compose](https://github.com/firefly-iii/docker/blob/main/docker-compose.yml)) |
| Immich (AGPL-3.0) | v3.3.6, 2026-11-04; v3.4.0-rc.3 | SemVer, "breaking changes ... limited to major version releases" ([upgrading](https://docs.immich.app/install/upgrading)) | GHCR; `release`, the major metatag `v3` ("do not follow release candidates") and `vX.Y.Z` | `docker-compose.yml` and `example.env` attached to each release; "the compose file on main may not be compatible" ([v3.3.6 compose](https://github.com/immich-app/immich/releases/download/v3.3.6/docker-compose.yml)). The server image floats on `${IMMICH_VERSION:-release}`, which [example.env](https://github.com/immich-app/immich/releases/download/v3.3.6/example.env) sets to `v3`; Valkey and Postgres are pinned by digest. Server, ML, Valkey, and a custom Postgres **14** image with VectorChord |
| Paperless-ngx (GPL-3.0) | v3.4.2, 2026-11-03 | SemVer | `ghcr.io/paperless-ngx/paperless-ngx:latest` | Several compose variants plus an install script; Valkey and Postgres 18 ([compose](https://github.com/paperless-ngx/paperless-ngx/blob/main/docker/compose/docker-compose.postgres.yml)) |
| Vaultwarden (AGPL-3.0) | 1.37.5, 2026-11-01 | SemVer; `testing` from main | Docker Hub, GHCR, Quay; `latest`, `testing`, `-alpine`, `x.y.z` ([wiki](https://github.com/dani-garcia/vaultwarden/wiki/Which-container-image-to-use)) | One container; image `HEALTHCHECK` calls `/alive` every 60 s ([Dockerfile](https://github.com/dani-garcia/vaultwarden/blob/main/docker/Dockerfile.debian)) |
| Home Assistant (Apache-2.0) | 2026.11.0, 2026-11-04 | CalVer, monthly; public betas b1–b6 before release ([releases](https://github.com/home-assistant/core/releases)) | `ghcr.io/home-assistant/home-assistant:stable` | Compose with `network_mode: host` and `privileged: true` ([install](https://www.home-assistant.io/installation/linux)) |
| Ghostfolio (AGPL-3.0) | 3.98.0, 2026-11-04; five releases in eight days | SemVer | Docker Hub `ghostfolio/ghostfolio`, amd64, armv7 and arm64 ([README](https://github.com/ghostfolio/ghostfolio/blob/main/README.md)) | Compose with `cap_drop: ALL`, `no-new-privileges`, healthchecks and `depends_on: service_healthy`; Postgres 15 and Redis ([compose](https://github.com/ghostfolio/ghostfolio/blob/main/docker/docker-compose.yml)) |
| Maybe Finance (AGPL-3.0) | v0.6.0, 2025-07-24, **final** | Archived. "This repository is no longer actively maintained"; the company pivoted to B2B ([final release](https://github.com/maybe-finance/maybe/releases/tag/v0.6.0)). Community fork: [Sure](https://github.com/we-promise/sure), v0.7.6 on 2026-10-31 | GHCR; `latest` was the latest commit and `stable` the latest release ([docs](https://github.com/maybe-finance/maybe/blob/main/docs/hosting/docker.md)) | Web, a Sidekiq worker, Postgres 16 and Redis. The example compose had a hard-coded fallback `SECRET_KEY_BASE` ([compose](https://github.com/maybe-finance/maybe/blob/main/compose.example.yml)) |

**B. Operations**

| Project | First admin | Members | Config, secrets | Backup, export | Upgrade, downgrade, client versions |
|---|---|---|---|---|---|
| Actual | Unauthenticated `GET /account/needs-bootstrap` and `POST /account/bootstrap`: the first caller sets the server password, later calls get `already-bootstrapped` (sync-server source on `master`, 2026-11-07: [app-account.js](https://github.com/actualbudget/actual/blob/master/packages/sync-server/src/app-account.js), [account-db.js](https://github.com/actualbudget/actual/blob/master/packages/sync-server/src/account-db.js)) | OpenID mode: Admin and Basic roles, per-budget-file access ([multi-user](https://actualbudget.org/docs/config/multi-user)) | `config.json` or `ACTUAL_*` env; `trustedProxies` ([config](https://actualbudget.org/docs/config/)) | Export from Settings ([backup](https://actualbudget.org/docs/backup-restore/backup)) | The server serves the web client |
| Firefly III | "Firefly III opens up the registration form only for the first user" ([FAQ](https://docs.firefly-iii.org/references/faq/firefly-iii/general/)) | Users are "strictly separated"; to share, users "commit the capital sin of sharing your password" ([docs](https://docs.firefly-iii.org/explanation/more-information/what-its-not/)) | `.env`, with `*_FILE` for secrets; audit log "disabled by default" ([.env.example](https://github.com/firefly-iii/firefly-iii/blob/main/.env.example)) | No built-in backup; "Do not use the export function as a backup mechanism" ([backup](https://docs.firefly-iii.org/how-to/firefly-iii/advanced/backup/)) | Upgrades itself "even back from 4.7.x"; "In some cases the upgrade process is destructive" ([upgrade](https://docs.firefly-iii.org/how-to/firefly-iii/upgrade/docker/)) |
| Immich | "The first user to register will be the admin user" ([post-install](https://docs.immich.app/install/post-install)) | The admin creates users | `.env`; example `DB_PASSWORD=postgres` | Daily DB dump at 02:00, "keep last 14 backups"; manual dumps count "toward your retention limit"; restore from Administration > Maintenance; "Immich does not handle filesystem backups for you" ([backup](https://docs.immich.app/administration/backup-and-restore)) | "Downgrading ... even within the same minor version, is not supported". The app supports the current and prior major, the server only its own major, so "upgrading all mobile clients before upgrading the server" ([upgrading](https://docs.immich.app/install/upgrading)) |
| Paperless-ngx | `PAPERLESS_ADMIN_USER` and `_PASSWORD` create a superuser at start if absent ([config](https://docs.paperless-ngx.com/configuration/)) | Django users and permissions | `PAPERLESS_SECRET_KEY` "has no default value"; Paperless "will refuse to start if this is not set"; `_FILE` secrets ([setup](https://docs.paperless-ngx.com/setup/)) | `document_exporter`: zip, `--passphrase`, incremental. The importer needs "a completely empty installation"; "You cannot import the export generated with one version of paperless in a different version" ([admin](https://docs.paperless-ngx.com/administration/)) | "Running `docker compose up` will also apply any new database migrations"; v3 needed manual steps (same page) |
| Vaultwarden | Admin page only if `ADMIN_TOKEN` is set, preferably an Argon2 PHC string from `vaultwarden hash` ([admin page](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-admin-page)) | `SIGNUPS_ALLOWED` defaults to true; admins can invite with sign-ups off ([.env.template](https://github.com/dani-garcia/vaultwarden/blob/main/.env.template)) | `.env`; built-in login and admin rate limits | `/vaultwarden backup` (SQLite online backup), plus attachments and RSA keys ([backups](https://github.com/dani-garcia/vaultwarden/wiki/Backing-up-your-vault)) | Upstream clients auto-update, so "incompatible client and server versions can lead to sudden breakage" ([wiki](https://github.com/dani-garcia/vaultwarden/wiki/Which-container-image-to-use)) |
| Home Assistant | Owner created at onboarding; a lost owner password means "a new onboarding process" ([auth](https://www.home-assistant.io/docs/authentication/)) | The owner adds users | Not checked | Built-in backups, "encrypted by default"; the restore needs "the encryption key stored in the backup emergency kit"; restore during onboarding ([backups](https://www.home-assistant.io/common-tasks/general/backups/)) | Monthly releases with betas |
| Ghostfolio | "this first user will get the role `ADMIN`" ([README](https://github.com/ghostfolio/ghostfolio/blob/main/README.md)) | Sign-up | `.env`: `ACCESS_TOKEN_SALT`, `JWT_SECRET_KEY` | Not checked | `prisma migrate deploy` on every start ([entrypoint](https://github.com/ghostfolio/ghostfolio/blob/main/docker/entrypoint.sh)); readiness `/api/v1/health`, liveness `/api/v1/health/liveness` |

**TLS and proxy guidance.**
- Vaultwarden: HTTPS is "pretty much required" because the web vault's crypto APIs need a secure context. Prefer a reverse proxy to Rocket's built-in TLS, and Let's Encrypt or Caddy for certificates ([HTTPS](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS)).
- Actual: offers self-signed certificates, certificates from Tailscale or Caddy without internet exposure, or a reverse proxy ([HTTPS](https://actualbudget.org/docs/config/https)).
- Immich: the proxy must set `Host`, `X-Real-IP`, `X-Forwarded-Proto` and `X-Forwarded-For`, pass WebSockets, and serve "the root path of a (sub)domain". Examples cover nginx, Caddy, Apache and Traefik ([proxy](https://docs.immich.app/administration/reverse-proxy)).
- Actual, Paperless-ngx, Firefly III and Vaultwarden all have an explicit trusted-proxies setting.

**Lessons** (high, from the tables):
- Every project ships compose. Immich attaches its file to each release, but that file still follows the floating major tag `v3`, not the release's own image; only its Postgres and Valkey images are pinned by digest. A compose file pinned to its release's digest, as planned here, goes further than any of these precedents.
- Weak example defaults are common: `postgres`, `paperless`, a fixed `SECRET_KEY_BASE`, open sign-ups. Most examples publish the app port on every interface.
- "First visitor becomes admin" is the norm, and it is a race on any instance reachable before setup.
- None of these apps runs several households, each with internal sharing, in one instance. Actual's per-file access comes closest. Firefly III states that "any database administrator can see everybody's administrations" ([FAQ](https://docs.firefly-iii.org/references/faq/firefly-iii/general/)).
- Nix: nixpkgs has community modules for Actual, Firefly III, Immich, Paperless-ngx, Vaultwarden and Home Assistant, but none for Ghostfolio. None of the eight upstream repos ships its own flake ([nixpkgs modules](https://github.com/NixOS/nixpkgs/tree/master/nixos/modules/services)).

## 2. Packaging and distribution

### 2.1 What Nix-friendly projects do

- **headscale.** Its flake exports `nixosModules.default`, a `headscale-docker` image built with `dockerTools.buildLayeredImage` ("only works on Linux"), and VM tests through `pkgs.testers.runNixOSTest` ([flake](https://github.com/juanfont/headscale/blob/main/flake.nix)). Its published GHCR images, however, come from ko through goreleaser ([.goreleaser.yml](https://github.com/juanfont/headscale/blob/main/.goreleaser.yml)).
- **attic.** It publishes Nix-built images ([packages.nix](https://github.com/zhaofengli/attic/blob/main/flake/packages.nix), [push script](https://github.com/zhaofengli/attic/blob/main/.ci/build-and-push-images.sh)):
  - an x86_64 image from `dockerTools.buildImage` and a cross-compiled aarch64 image;
  - each pushed by digest with `skopeo copy docker-archive:`, then joined into one tag with `manifest-tool push from-spec`;
  - both images add `dockerTools.fakeNss` and a CA bundle.
- **nixpkgs modules** (immich, actual, vaultwarden):
  - a freeform `settings` option, as [RFC 42](https://github.com/NixOS/rfcs/blob/master/rfcs/0042-config-option.md) proposes;
  - secrets from a `secretsFile` or `environmentFile`, kept out of "the world-readable Nix store" ([vaultwarden module](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/security/vaultwarden/default.nix)), plus `LoadCredential`;
  - a local Postgres through `ensureUsers`, with peer auth over the Unix socket;
  - `openFirewall` off by default;
  - systemd sandboxing: `ProtectSystem=strict`, `RestrictAddressFamilies`, `SystemCallFilter`, and `DynamicUser` in actual ([immich](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/web-apps/immich.nix), [actual](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/web-apps/actual.nix)).
  - Vaultwarden's module adds a `backupDir` with a backup timer. Nextcloud's module ships a preconfigured `nextcloud-occ` admin wrapper ([nextcloud](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/web-apps/nextcloud.nix)).

### 2.2 Image builders

| | `dockerTools.buildLayeredImage` / `streamLayeredImage` | nix2container | Dockerfile + buildx |
|---|---|---|---|
| Source of truth | The flake package | The flake package | A second build definition |
| Reproducible | Yes; a static creation date by default ([nixpkgs manual](https://github.com/NixOS/nixpkgs/blob/master/doc/build-helpers/images/dockertools.section.md)) | Yes; per-layer `reproducible` flag | Not by default |
| Layers | One per store path, the most "popular" first, rest grouped | Explicit `layers` groups | One per instruction |
| Push | Tarball, then `skopeo copy` (attic) | `copyToRegistry` through a patched skopeo "nix" transport; "skip already pushed layers" ([README](https://github.com/nlewo/nix2container)) | `buildx --push` |
| Rebuild + reload or push (nix2container README; "we could not compare the same distribution mechanisms") | No `buildLayeredImage` figure. `streamLayeredImage` into `docker load`, no registry: ~7.5 s. `buildImage` + skopeo to a local registry: ~10 s | ~1.8 s with `copyToRegistry` | n/a |
| Dependencies | nixpkgs only | Extra flake input; latest release v1.1.0 (2026-09-24) | Docker |
| Contributors without Nix | Can't build it | Can't | Can |

**Pick `buildLayeredImage`** (medium):
- Build it natively on `ubuntu-24.04` and `ubuntu-24.04-arm`. Standard runners are "free and unlimited on public repositories", and the arm64 labels are in the public-repo table ([runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)).
- Push by digest with skopeo and join the two with `docker buildx imagetools create`, which needs source manifests "already exist[ing] in the registry" ([docs](https://docs.docker.com/reference/cli/docker/buildx/imagetools/create/)).
- Attest with `actions/attest` and `push-to-registry` ([action](https://github.com/actions/attest)).
- Move to nix2container only if push time starts to hurt.

**Image contents** (judgment):
- The server package, CA certificates, tzdata and `fakeNss`, and no shell.
- A non-root numeric `User`, and an `on-budget healthcheck` subcommand, since the image has no curl.
- `/var/lib/on-budget/backups` created and owned by UID 10001 through `fakeRootCommands` and `chown`. In dockerTools layers, "by default, all files in the layer will be owned by root" ([nixpkgs manual](https://github.com/NixOS/nixpkgs/blob/master/doc/build-helpers/images/dockertools.section.md)). Docker copies the image's directory into a new, empty named volume ([volumes](https://docs.docker.com/engine/storage/volumes/)), and its code "ensures the ownership is appropriately set" ([moby](https://github.com/moby/moby/blob/master/daemon/container/container_unix.go)). Without the `chown`, expect a root-owned volume: the pre-migration dump then fails and §3.1 aborts the start.
- `pg_dump` and `pg_restore` for each supported major, picked at run time from the server's version. pg_dump "cannot dump from PostgreSQL servers newer than its own major version", and "it is not guaranteed that pg_dump's output can be loaded into a server of an older major version" ([pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html)). A 19 client alone could dump an 18 server, but its output might not restore into 18. On NixOS, the module takes them from the local server's package.
- The `org.opencontainers.image.source` label, which links the GHCR package to the repository ([GHCR](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)); headscale sets it too ([.goreleaser.yml](https://github.com/juanfont/headscale/blob/main/.goreleaser.yml)).
- A separate `-debug` tag that adds busybox, for troubleshooting (§2.7).

### 2.3 Flake layout and language packaging

```text
flake.nix
  packages.<system>.default   server: API, built web app, CLI, embedded SQL migrations
  packages.<linux>.image      OCI image (buildLayeredImage), one per architecture
  nixosModules.default        services.on-budget
  checks.<linux>.*            unit tests; NixOS VM tests: fresh install, upgrade from the
                              previous release, backup → wipe → restore; compose smoke test
  devShells.default           the toolchain CI uses
deploy/compose/               compose.yaml + .env.example, rendered with the image digest per release
```

- **Language** (as of Nov 2026). The backend is still TypeScript or Swift:
  - TypeScript builds with `buildNpmPackage` (`npmDepsHash`, or `importNpmLock` straight from the lockfile) or with `fetchPnpmDeps` ([JS docs](https://github.com/NixOS/nixpkgs/blob/master/doc/languages-frameworks/javascript.section.md)).
  - Swift on NixOS 26.05 is 5.10.1, with only `swiftpm2nix` for dependencies ([sources.nix](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/pkgs/development/compilers/swift/sources.nix), [26.05 Swift docs](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/doc/languages-frameworks/swift.section.md)). `fetchSwiftPMDeps` and Swift 6.4.0 exist only on master ([Swift docs](https://github.com/NixOS/nixpkgs/blob/master/doc/languages-frameworks/swift.section.md), [swift-version](https://github.com/NixOS/nixpkgs/blob/master/pkgs/development/compilers/swift/swift-version)), so they arrive with 26.11. A Swift 6 backend therefore pins the flake to 26.11 or later, as do hosts whose nixpkgs `follows` it, or it wraps a Static Linux SDK binary ([nixos-vps-infra.md](nixos-vps-infra.md)).
  - `importNpmLock` needs no hash: "There is no need to specify a `hash`, since it relies entirely on the integrity hashes already present in the `package-lock.json` file" (JS docs). `npmDepsHash` (`fetchNpmDeps`), `fetchPnpmDeps` and `fetchSwiftPMDeps` each take a hash that must change with every lockfile bump, Dependabot PRs included (judgment).
- **Flakes** are still an experimental feature in Nix 2.35.4 ([manual](https://nix.dev/manual/nix/latest/development/experimental-features)). Determinate Nix offers its own stability guarantee (secondary, [NixOS Discourse](https://discourse.nixos.org/t/determinate-nix-3-0/61202)). NixOS stable is 26.05; 26.11 is planned for 2026-11-30 ([schedule](https://github.com/NixOS/nixpkgs/issues/567567)).
- **VM tests**: `pkgs.testers.runNixOSTest` works outside nixpkgs ([NixOS manual](https://nixos.org/manual/nixos/stable/#sec-nixos-tests)). `cachix/install-nix-action` v31.11.3 enables KVM "on supported machines" for NixOS tests ([README](https://github.com/cachix/install-nix-action)).

### 2.4 `services.on-budget` (sketch, judgment)

```nix
services.on-budget = {
  enable = true;
  package = inputs.on-budget.packages.${pkgs.system}.default;  # same flake revision as the module
  settings = {                                   # RFC 42 freeform settings
    public_url = "https://budget.example.org";   # https only; also the passkey RP ID, so choose it once
    listen = "127.0.0.1:8080";                   # or a Unix socket
    trusted_proxies = [ "127.0.0.1/32" ];
  };
  credentials = {                                # LoadCredential=, never copied into the Nix store
    session_key = config.age.secrets.on-budget-session.path;   # agenix or sops-nix
    kek = config.age.secrets.on-budget-kek.path;
    vapid_key = config.age.secrets.on-budget-vapid.path;
  };
  database.createLocally = true;                 # ensureUsers, peer auth; asserts a supported major
  backup = { enable = true; startAt = "hourly"; keepHourly = 48; keepDaily = 14; ageRecipients = [ "age1…" ]; };
  caddy.enable = true;                           # optional vhost; private by default: DNS-01 or a *.ts.net name
};
services.postgresql.package = pkgs.postgresql_18; # host-wide: set by the owner, never by the module
```

- **Units, in order**: `on-budget-migrate.service` (a oneshot: pre-migration dump, then migrate), `on-budget.service` (sandboxed like the nixpkgs modules), then `on-budget-backup.timer`.
- **CLI**: an `on-budget-admin` wrapper runs the CLI as the service user, like `nextcloud-occ`.
- **Defaults**: `openFirewall` stays false.
- **PostgreSQL major.** On 26.05 the default package follows `system.stateVersion`: `postgresql_17` for 25.11 and later. `postgresql_18` becomes the default only on master, the future 26.11, for stateVersion 26.11 and later ([26.05 module](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/nixos/modules/services/databases/postgresql.nix), [master](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/databases/postgresql.nix)). The option is host-wide, and `dataDir` follows the major, so changing it on a host starts an empty cluster in a new directory. The module therefore asserts a supported major instead of setting it, and the install doc sets `pkgs.postgresql_18`, as nixos-vps-infra.md does.
- **TLS.** The reference deployment is Tailscale-only (ADR 0003), so the vhost gets its certificate without inbound traffic: `security.acme` with a lego DNS provider (`dnsProvider`) plus Caddy's `useACMEHost`, which nixpkgs documents as "mostly useful if you use DNS challenges but Caddy does not currently support your provider", or `services.tailscale.serve` (in 26.05) for a `*.ts.net` name ([acme module](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/nixos/modules/security/acme/default.nix), [Caddy vhost options](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/nixos/modules/services/web-servers/caddy/vhost-options.nix)).
- **Maintainer's instance**: ADR 0003 settles it. The reference deployment runs this module from a private flake, chosen over Docker-based deployment (which covers [factory-ci-quality.md](factory-ci-quality.md)'s Kamal path), and "no container registry is needed".
- **Credentials**: systemd credentials, unlike environment variables, are "not propagated down the process tree" and sit "in non-swappable memory". They can also be encrypted with a TPM2 or host key ([systemd](https://systemd.io/CREDENTIALS/)). Secret files come from [agenix](https://github.com/ryantm/agenix) or [sops-nix](https://github.com/Mic92/sops-nix).

### 2.5 Compose bundle (sketch, judgment)

```yaml
services:
  on-budget:
    image: ghcr.io/fveauvy/on-budget:1.4.2@sha256:…   # rendered per release
    environment:
      ON_BUDGET_PUBLIC_URL: https://budget.example.org
      ON_BUDGET_DATABASE_PASSWORD_FILE: /run/secrets/db_password
      ON_BUDGET_SESSION_KEY_FILE: /run/secrets/session_key
      ON_BUDGET_KEK_FILE: /run/secrets/kek
    secrets: [db_password, session_key, kek]   # host files must be readable by UID 10001
    ports: ["127.0.0.1:8080:8080"]          # for a host proxy or Tailscale Serve; Docker bypasses ufw
    volumes: ["backups:/var/lib/on-budget/backups"]   # the image creates this directory, owned by 10001
    read_only: true
    tmpfs: [/tmp]
    user: "10001:10001"
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    healthcheck: { test: ["CMD", "/bin/on-budget", "healthcheck"], interval: 30s, start_period: 5m }
    depends_on: { db: { condition: service_healthy } }
    networks: [db, egress, proxy]           # egress: bank API, APNs, Web Push; proxy: shared with caddy
    restart: unless-stopped
  db:
    image: postgres:18                      # major pinned; 18+ mounts /var/lib/postgresql; checksums on by default
    environment: { POSTGRES_USER: on_budget, POSTGRES_PASSWORD_FILE: /run/secrets/db_password }
    secrets: [db_password]
    volumes: ["pgdata:/var/lib/postgresql"]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U on_budget"], interval: 10s }
    networks: [db]                          # never published
  caddy:                                    # `--profile proxy`: a public instance only (HTTP-01, TLS-ALPN-01)
    image: caddy:2                          # standard modules only, so no DNS-01
    profiles: [proxy]
    ports: ["80:80", "443:443", "443:443/udp"]
    volumes: ["./caddy:/etc/caddy:ro", "caddy_data:/data", "caddy_config:/config"]   # Caddyfile: reverse_proxy on-budget:8080
    networks: [proxy]
    restart: unless-stopped
networks: { db: { internal: true }, egress: {}, proxy: {} }
secrets: { db_password: { file: ./secrets/db_password }, session_key: { file: ./secrets/session_key }, kek: { file: ./secrets/kek } }
volumes: { pgdata: {}, backups: {}, caddy_data: {}, caddy_config: {} }
```

- **Docker Engine 28.0.0 or later.** Docker says "Publishing container ports is insecure by default", and binding to `127.0.0.1` means "only the Docker host can access the published container port". But "In releases older than 28.0.0, hosts within the same L2 segment … can reach ports published to localhost" ([ports](https://docs.docker.com/engine/network/port-publishing/)), which matters on a VPS's shared network. With ufw, "Packets are routed before the firewall rules can be applied" ([firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/)).
- **Networks.** A service without `networks` joins only `default`, and services that share no network can't talk: "Compose doesn't warn you about a configuration mismatch" ([services](https://docs.docker.com/reference/compose-file/services/)). Docker's own example: "The `proxy` service is isolated from the `db` service, because they do not share a network in common" ([networking](https://docs.docker.com/compose/how-tos/networking/)). Hence the `proxy` network that both `caddy` and `on-budget` join. It isn't `internal`, because Caddy must reach the ACME CA.
- **Secrets.** Compose mounts secrets at `/run/secrets/<name>`, and `_FILE` is a convention of the official postgres image ([secrets](https://docs.docker.com/compose/how-tos/use-secrets/)). File-sourced secrets are bind mounts: for `uid`, `gid` and `mode`, "these attributes are silently ignored" ([services](https://docs.docker.com/reference/compose-file/services/)). So `on-budget secrets generate ./secrets` must leave each file owned by UID 10001 with mode 0400 (judgment). Postgres is unaffected: its entrypoint reads `POSTGRES_PASSWORD_FILE` as root, then exports the value before `gosu` drops privileges ([entrypoint](https://github.com/docker-library/postgres/blob/master/18/trixie/docker-entrypoint.sh)).
- **Roles.** `POSTGRES_USER` is created "with superuser power" ([image docs](https://github.com/docker-library/docs/blob/master/postgres/content.md)). on-budget should run migrations as that role but serve through a non-superuser role it creates, which a restore must recreate (§4; judgment).
- **Checksums.** PostgreSQL 18's initdb enables data checksums: "This is enabled by default" ([initdb](https://www.postgresql.org/docs/18/app-initdb.html)), so `POSTGRES_INITDB_ARGS: --data-checksums`, which Immich's Postgres 14 compose still sets, is redundant.
- **Proxy.** The stock image carries only Caddy's "standard" modules ([caddy image](https://github.com/docker-library/docs/blob/master/caddy/content.md)), so it can't do DNS-01 and needs public 80/443. Mount the Caddyfile's folder, not the file: "Do not mount the Caddyfile directly at `/etc/caddy/Caddyfile`" (same page). A private install instead runs `tailscale serve` on the host in front of `127.0.0.1:8080`, or a Caddy image rebuilt with xcaddy and the DNS provider's caddy-dns module (§6).
- **Postgres volume.** The official image moved `PGDATA` to `/var/lib/postgresql/18/docker` and its volume to `/var/lib/postgresql` from 18 on, so that `pg_upgrade --link` works ([image docs](https://github.com/docker-library/docs/blob/master/postgres/content.md)).

### 2.6 Release pipeline

1. **PR.** `nix flake check` runs the unit and VM tests, then a compose smoke test runs against the freshly built image.
2. **Tag `vX.Y.Z`.**
   - Build both architectures, push them by digest and create the index.
   - Tag it `X.Y.Z`, `X.Y`, `X` and `latest`, then attest it. The attestation gives SLSA v1.0 Build Level 2 and works in public repos on every current plan ([attestations](https://docs.github.com/en/actions/concepts/security/artifact-attestations), [action](https://github.com/actions/attest-build-provenance)).
   - Render the compose files and publish a GitHub Release with upgrade notes.
   - **First release only:** make the package public, then check an anonymous pull. "When you first publish a package, the default visibility is private" ([container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)). A package linked to its repository "inherits the access permissions (but not the visibility)" of that repository, and "Once you make a package public, you cannot make it private again" ([visibility](https://docs.github.com/en/packages/learn-github-packages/configuring-a-packages-access-control-and-visibility)). Public container images "can be pulled without authentication" (same page). The image carries the `org.opencontainers.image.source` label from the first push (§2.2).
3. **Pre-releases** `X.Y.Z-rc.N` get only their own tag, as Immich's metatags skip RCs. `main` publishes `edge` and `sha-<short>`.
4. **Verify**: `gh attestation verify oci://ghcr.io/fveauvy/on-budget:1.4.2 -R fveauvy/on-budget` ([docs](https://docs.github.com/en/actions/how-tos/secure-your-work/use-artifact-attestations/use-artifact-attestations)).
5. **GHCR cost.** "GitHub Packages usage is free for public packages", which applies once the package is public. Container registry storage and bandwidth are "currently free", with at least one month's notice of any change ([billing](https://docs.github.com/en/billing/concepts/product-billing/github-packages)).

### 2.7 Trade-offs of Nix-first

- **For.**
  - One definition yields the package, the module and the image, and the module always matches its package.
  - VM tests cover install, upgrade and restore on every PR.
  - Images are small, with no shell or package manager.
- **Against.**
  - Contributors can't run `docker build .`, and flakes are still experimental upstream.
  - Dependency hashes churn with every lockfile bump, except with `importNpmLock` (§2.3).
  - A Nix image has no debugging tools. A debug variant is the established fix: attic's image adds busybox as "Debugging utilities for `fly ssh console`" ([packages.nix](https://github.com/zhaofengli/attic/blob/main/flake/packages.nix)), and headscale publishes `-debug` tags on distroless's debug base ([.goreleaser.yml](https://github.com/juanfont/headscale/blob/main/.goreleaser.yml)).
- **Alternative.** A Dockerfile plus buildx for images, and the flake for NixOS. That means two build definitions, but it is friendlier to contributors. headscale is no precedent for it: its released images come from goreleaser's `kos:` and its `main` images from `ko build` ([container-main.yml](https://github.com/juanfont/headscale/blob/main/.github/workflows/container-main.yml)), and ko is "a simple, fast container image builder for Go applications" ([ko](https://github.com/ko-build/ko)). Its Dockerfiles serve integration tests, derper and similar tools. Revisit if contributors without Nix appear (judgment).

## 3. Upgrades, migrations and client versions

### 3.1 Migrations (high for the pattern, judgment for the details)

- **Embedded and forward-only.** SQL migrations are numbered and compiled into the binary.
  - `on-budget migrate` applies them under `pg_advisory_lock`, and `serve` runs pending ones at start, as Immich, Paperless-ngx, Ghostfolio and Firefly III do.
  - Before the first pending step, the server writes a `pg_dump -Fc` to the backup volume and aborts if that fails. pg_dump "makes consistent exports even if the database is being used concurrently" ([pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html)).
  - Before any dump or migration, it reads the server's major and refuses an unsupported one with a clear message, rather than failing inside `pg_dump` (§2.2).
- **Refuse a newer schema.** Gitea stops with "Your database ... is for a newer Gitea, you can not use the newer database for this old Gitea release" ([PR 19504](https://github.com/go-gitea/gitea/pull/19504)). This turns an accidental rollback into a clear error.
- **Expand/contract** applies to anything a client or the previous server version reads.
  - Contract steps ship at least one minor release later.
  - squawk lints the migrations in CI ([factory-ci-quality.md](factory-ci-quality.md)).
- **Tests.**
  - Upgrade fixture databases from each of the last two minor releases.
  - Meanwhile, a client simulator on version N−1 pushes mutations, which extends the simulator in [sync-engine.md](sync-engine.md).
- **Upgrade path.** Any release in the support window upgrades directly. A major release may declare required stops, the way GitLab does ([upgrade paths](https://docs.gitlab.com/update/upgrade_paths/)).

### 3.2 Rollback

- **Rolling back the binary alone** is safe only when the release added no migration:
  - On NixOS, use `nixos-rebuild switch --rollback` ([manual](https://nixos.org/manual/nixos/stable/)); on compose, use the previous image tag.
  - Each release note states "migrations: yes/no".
- **Otherwise:**
  1. Stop the server, restore the pre-migration dump, and start the old version.
  2. Everything written after the upgrade is lost: server jobs' writes, and members' writes the server already acknowledged, because clients drop a mutation once it is confirmed (sync-engine.md). After the epoch change, clients re-push only what is still pending, unless they keep acknowledged mutations as §4 proposes.
  3. Immich says downgrades aren't supported, and none of the other references documents a downgrade path.

### 3.3 Old offline clients against a newer server, and the reverse

- **Facts.**
  - TestFlight builds expire after 90 days ([apple-client-stack.md](apple-client-stack.md)), while App Store users may never update.
  - Web clients come from the server, but stale tabs remain ([web-client-sync.md](web-client-sync.md)).
  - Immich ties the server to its major version and supports the prior major in the app. Vaultwarden warns of breakage when clients auto-update ahead of the server.
- **Contract** (medium, building on sync-engine.md's versioned mutations and web-client-sync.md's `minClientVersion`):
  - `GET /api/v1/meta` returns `version`, `api`, `sync`, `minClient` per platform, and `features`.
  - The server accepts the mutation versions of the current and previous minor release, for at least 90 days.
    - An older client gets a typed error, such as `409` with code `client_too_old` and the minimum version, and keeps its outbox. Not `426 Upgrade Required`, which asks for a protocol switch: "The server MUST send an Upgrade header field in a 426 response to indicate the required protocol(s)" ([RFC 9110 §15.5.22](https://www.rfc-editor.org/rfc/rfc9110#section-15.5.22)).
  - Each app carries a `minServerVersion`. Below it, the app stays usable offline, read-only, and says the owner must upgrade the server.
  - Wire-contract breaks happen only in major releases.
- **Order** (judgment): upgrade the server first, then the apps, within the window. Immich asks for the reverse because its server drops the previous major.

### 3.4 PostgreSQL major upgrades

- **What PostgreSQL requires.**
  - "Minor releases never change the internal storage format". Majors need dump and restore, `pg_upgrade`, or logical replication ([upgrading](https://www.postgresql.org/docs/current/upgrading.html)).
  - 18.6 is current. PostgreSQL 14 reaches its final release on 2026-11-12, and 18 on 2030-11-14 ([versioning](https://www.postgresql.org/support/versioning/)).
  - PostgreSQL 19 RC 1 came out on 2026-10-15. RC 2 is planned for 2026-11-12, and general availability for 2026-11-19 ([RC 1 announcement](https://www.postgresql.org/about/news/postgresql-19-rc-1-released-3409/)).
- **NixOS.** The default version follows `system.stateVersion` (on 26.05, 17 for stateVersion 25.11 and later; §2.4), and "upgrades must happen by setting `services.postgresql.package` explicitly", followed by a documented `upgrade-pg-cluster` script ([module docs](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/databases/postgresql.md)).
- **Policy** (judgment):
  - Vanilla Postgres with no extensions, so distro or managed Postgres also works. Immich shows the cost of a custom image: its v3.3.6 compose still pins a Postgres 14 base.
  - Support two adjacent majors: 18 and 19. 19 enters the window with the first on-budget release after 19.0 ships, once the VM tests and the compose smoke test pass on both majors. Hosts on 17, such as NixOS hosts left on 26.05's stateVersion default, upgrade to 18 first (`upgrade-pg-cluster` on NixOS).
  - The image bundles the client tools of both majors (§2.2), because an 18 `pg_dump` can't dump a 19 server: on an external 19 server, the scheduled backups and the pre-migration dump would fail, and the start would abort (§3.1).
  - Dropping 18, or moving the compose default from `postgres:18` to 19, waits for a server major release, with an `on-budget db upgrade` helper that dumps, then restores into a new volume.

### 3.5 Versioning and channels

| | SemVer (Immich, Paperless-ngx, Vaultwarden, Firefly III, Ghostfolio) | CalVer (Actual, Home Assistant) |
|---|---|---|
| Signals | Compatibility: a breaking change means a new major | Age: the release month |
| Fit for on-budget | Self-hosters need to know whether they can skip a release, and apps need compatibility ranges | Hides breaking changes |

**Pick SemVer** (medium-high), with three channels: `stable` (`X.Y.Z`, `X.Y`, `X`, `latest`), `rc` (pre-release tags only) and `edge` (`main`). No backports: security fixes ship as patches of the latest minor, and Immich doesn't backport either.

## 4. Backups, restore and export (medium)

| Item | Default | Notes |
|---|---|---|
| Database | Hourly `pg_dump -Fc` to the backup volume, encrypted to the owner's `age` recipients, with the client of the server's major (§2.2). Kept by age: 48 h of hourly dumps, then 14 days of daily ones. A dump before every migration, kept apart: the last three | Immich makes daily dumps and keeps the last 14 by count, with manual dumps counting "toward your retention limit" ([backup](https://docs.immich.app/administration/backup-and-restore)); counting would let pre-upgrade dumps push scheduled ones out. On NixOS, `services.postgresqlBackup` exists ([module](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/backup/postgresql-backup.nix)), and the reference deployment streams hourly dumps into restic instead ([nixos-vps-infra.md](nixos-vps-infra.md)) |
| Roles | `pg_dumpall --globals-only --no-role-passwords` next to each dump; `on-budget restore` recreates the roles it owns: the serving role (§2.5) and a PowerSync replication role if adopted | pg_dump "only dumps a single database"; for "global objects … (such as roles and tablespaces), use pg_dumpall" ([pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html)). `--no-role-passwords` reads `pg_roles` instead of `pg_authid` ([pg_dumpall](https://www.postgresql.org/docs/18/app-pg-dumpall.html)) |
| Why `pg_dump` | A dump is enough here, and WAL archiving (pgBackRest, WAL-G) is the upgrade path if an hour is too much | PostgreSQL 18 adds: "except in simple cases, pg_dump is generally not the right choice for taking regular backups of production databases" ([pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html)); PostgreSQL 17's page still called it "a utility for backing up". on-budget is a simple case (judgment): one database of a few MB per household ([sync-engine.md](sync-engine.md)), restored whole, and dumps load into newer majors, while file-level and WAL backups are "extremely server-version-specific" ([§25.1](https://www.postgresql.org/docs/18/backup-dump.html)). WAL archiving's failure mode is a `pg_wal` that fills until PostgreSQL panics ([nixos-vps-infra.md](nixos-vps-infra.md)) |
| Secrets | An "emergency kit" printed at bootstrap: the KEK, session key, VAPID key, and where the `age` identity lives | Home Assistant's model. Without the KEK, stored bank credentials must be re-entered, but budgets survive |
| Off-site | Documented recipes: restic or rclone, and `services.restic.backups` on NixOS | The self-hoster's job, as at Immich |
| Restore | `on-budget restore <dump>` into an empty database, with its roles, which rotates `instance_epoch` | Clients see the new epoch, wipe their cache, re-bootstrap and re-push their outbox. What that recovers, and what it can't, is below |
| Household export | Versioned NDJSON + CSV that later versions can import; no provider secrets | Avoids Paperless-ngx's version lock. Also the GDPR access and portability path ([bank-aggregation.md](bank-aggregation.md) §7.2). It is not a backup, as Firefly III warns |
| Drill | `on-budget backup verify` restores into a temporary database and checks invariants. The admin UI shows the last good backup | Firefly III: "the first thing you must do is restore it" |

**What a restore recovers** (judgment; it amends [sync-engine.md](sync-engine.md)'s protocol):
- **Recovery point: up to one hour.** Everything the server committed after the dump is gone: server jobs' writes, and members' writes it already acknowledged. Clients can't replay those as the protocol stands: they drop a mutation once `mutation_id ≤ lastMutationId`, or once its acked `sync_id ≤ cursor` (sync-engine.md, client pull step 3), as in Replicache, where `lastMutationID` "is how the client knows which mutations have been processed authoritatively and can therefore have their optimistic versions dropped" ([Replicache](https://v12.doc.replicache.dev/byob/dynamic-pull)). With daily dumps, the loss would reach 24 h of every member's work.
- **Mutation IDs.** `/push` requires "mutation IDs to be contiguous" (sync-engine.md), and the restored `last_mutation_id` lags each client's counter, so the first re-push would block the queue. After an epoch change, the server accepts one gap per client: IDs at or below the restored value are skipped as already applied, and the first ID above it starts the new sequence. Re-registering under a new client ID would lose that duplicate check.
- **Optional narrowing.** Each pull reports the last `sync_id` contained in the newest dump, and clients keep acknowledged mutations until that value covers them. After a restore, they re-push the kept mutations whose `sync_id` is above the restored counter, then the pending ones, in order. That recovers members' work on devices that come back with their local database intact, at the cost of up to one dump interval of extra outbox. Some replays will be rejected, such as a categorization of a bank row not fetched again yet. A host lost together with its local dumps falls back to the off-site copy, so to an older point.
- **Bank data.** The worker re-fetches what the dump lacks, within the bank's history window (90 days without a new SCA) and at most four unattended fetches per day ([bank-aggregation.md](bank-aggregation.md) §3.4, §4.2).

## 5. Instance administration

### 5.1 Bootstrapping the owner

| Pattern | Used by | Risk | Verdict |
|---|---|---|---|
| First visitor registers and becomes admin | Immich, Ghostfolio, Firefly III, Actual | Anyone who reaches the URL first owns the instance | ✗ |
| Admin created from environment variables | Paperless-ngx | A password sits in the config; on-budget has no passwords anyway | ✗ |
| One-time setup token in the logs | Jenkins ([docs](https://www.jenkins.io/doc/book/installing/linux/)), Portainer ("look for the `setup_token=` line", [docs](https://docs.portainer.io/start/install-ce/server/setup)) | Needs log access, which only the operator has | Acceptable |
| **CLI on the host prints a one-time enrollment link** | — | Needs a shell, which only the operator has | **✓** (medium-high) |

- **Flow** (judgment):
  1. With an empty database, the server serves only a "not initialized" page.
  2. `on-budget admin bootstrap --household Production` creates the owner and the first household, and prints a single-use link and QR code valid for 1 hour.
  3. The owner opens it and registers a passkey on the instance's origin.
- **Recovery.** `on-budget admin recover-owner` issues a new link. Home Assistant instead requires a new onboarding.

### 5.2 Members, households and roles

- **Invites only.** Each invite is single-use, lasts 72 hours, and is bound to a household and a role ([apple-client-stack.md](apple-client-stack.md) §3.3).
  - Invites are hashed at rest, revocable and audit-logged.
  - The owner shares them as a link or QR code, so no SMTP server is needed (judgment).
- **Roles.**
  - The instance `owner` operates the instance: households, invites, settings, backups and the audit log.
  - Household roles (`admin`, `member`) and ledger access are separate ([apple-client-stack.md](apple-client-stack.md)).
  - The owner sees a household's data only as one of its members. Still, anyone with database or host access can read everything, and the docs must say so.
- **Households are hard tenants.**
  - Every row carries `household_id`, and every sync space stays inside one household.
  - Postgres row-level security adds defense in depth.
  - If PowerSync is adopted, its documented replication role is created `WITH REPLICATION BYPASSRLS` ([PowerSync](https://docs.powersync.com/configuration/source-db/setup)), so RLS then guards only the API path, and a restore must recreate that role (§4).
- **Test and production households.**
  - A `kind=test` household gets a banner, demo-data seeding and file import.
  - Don't link the same bank account in both households through one provider app: unattended fetches share the bank's limit of 4 per 24 h ([bank-aggregation.md](bank-aggregation.md)).
  - Rehearse upgrades on a staging instance restored from a backup. Migrations are instance-wide, so a test household proves nothing about them (judgment).

### 5.3 Admin UI or CLI

| Task | Web admin UI | CLI |
|---|---|---|
| Bootstrap, owner recovery, restore, key rotation | — | ✓ |
| Households, invites, members, roles; devices and sessions | ✓ | ✓ |
| Bank-connection health and consent expiry | ✓ | — |
| Backups: status, run now, verify | ✓ | ✓ |
| Migration status; `doctor` checks: https, secret strength, proxy headers, clock, backup age | Status | ✓ |
| Audit log; household export and import | View and export | ✓ |

Sensitive admin actions require a fresh passkey assertion ([apple-client-stack.md](apple-client-stack.md)).

## 6. Security defaults

| Area | Default (judgment unless cited) | Evidence |
|---|---|---|
| Secrets | Generated by the CLI and read from files (`*_FILE`, compose secrets, systemd credentials). The server refuses to start if a secret is missing, too short or equal to an example value | Docker: environment variables "risk unintentional information exposure". Paperless-ngx refuses to start without its key. Maybe's fallback key shows the hazard |
| Bank-provider credentials | The server generates each Enable Banking key pair, and the person pastes only the certificate into EB's Control Panel; nobody uploads a private key ([connectors-and-license.md](connectors-and-license.md) §4.1). Keys and session IDs: AES-GCM with per-household data keys wrapped by the KEK. They never reach clients. Only the owner's own app is clearly within EB's terms; members' apps wait for EB's written answer and use file import until then | Provider credentials never ship in an app ([bank-aggregation.md](bank-aggregation.md) §4.4). EB: "keep the private key in secret (e.g. don't expose it to client, share with anyone nor embed into mobile or other apps intalled to user devices)" ([API reference](https://enablebanking.com/docs/api/reference/)). Its terms (updated 2026-01-09) forbid granting "any right to access or use the Control Panel or the API to any third party" ([ToS](https://enablebanking.com/terms)). bank-alternatives.md §1 calls an uploaded `.pem` a "share with anyone" breach and rates a member's key on the owner's instance "Grey; ask EB" |
| Network | The app listens on loopback or a Unix socket, and Postgres is never published. `public_url` must be `https` (except on localhost). HSTS, `__Host-` cookies and the CSP come from [web-client-sync.md](web-client-sync.md). Serve at the root of a dedicated (sub)domain. By default, nothing is open to the internet: the instance is reached over Tailscale. A public instance is an opt-in that opens only 443, plus 80 for ACME and redirects. PowerSync, if adopted, needs a second hostname (P7) | ADR 0003: the reference deployment is "reachable only over Tailscale", and "Nothing may depend on inbound connections from the internet". PRODUCT.md asks the same of every feature. Docker port warnings (§2.5) |
| TLS without inbound traffic | A `*.ts.net` name with Tailscale's certificate: Tailscale Serve, or Caddy fetching it from `tailscaled`. Or a name under the owner's domain with a DNS-01 certificate: on NixOS, `security.acme` with a lego DNS provider (§2.4); on compose, a Caddy image rebuilt with the provider's caddy-dns module. The stock `caddy:2` image can't: it lacks DNS modules, and its other challenges need inbound traffic | Caddy: HTTP-01 "requires port `80` to be externally accessible", TLS-ALPN-01 the same for 443. DNS-01 "does not require any open ports", but "requires configuration. Caddy needs to know the credentials to access your domain's DNS provider", and "DNS provider support is a community effort". `.ts.net` certificates come "from the locally-running Tailscale instance" ([Caddy](https://caddyserver.com/docs/automatic-https)). Tailscale Serve "requires you to enable HTTPS certificates in your tailnet" ([Serve](https://tailscale.com/kb/1312/serve)). `*.ts.net` names appear in public certificate-transparency logs, and since `ts.net` is a public suffix, every machine in the tailnet shares the instance's site ([native-apps-auth.md](native-apps-auth.md) §6, §9) |
| Client IPs | `trusted_proxies` is explicit; `X-Forwarded-For` is ignored from anyone else | Four of the reference apps have the same setting |
| Rate limits | Built in, per IP and per account: sign-in, invite redemption, bootstrap and recovery links, step-up. Each limit returns `429` and writes an audit event | Vaultwarden defaults: login 60 s with a burst of 10, admin 300 s with a burst of 3; fail2ban optional ([hardening](https://github.com/dani-garcia/vaultwarden/wiki/Hardening-Guide)) |
| Audit log | On by default, append-only, visible to the owner, kept 13 months. It covers authentication success and failure, authorization failures, user administration, privilege changes, import and export. It never records tokens, session IDs or bank data | [OWASP logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html). Firefly III's audit log is off by default |
| Logs | No tokens in URLs | Vaultwarden warns that proxy access logs capture `access_token` |
| Container | Non-root, read-only root, `cap_drop: ALL`, `no-new-privileges`, no shell; a separate `-debug` tag adds busybox (§2.7) | Ghostfolio's compose does the same |
| Telemetry | None. If PowerSync is adopted, set `disable_telemetry_sharing: true` | PowerSync shares anonymous usage metrics "unless you opt-out" ([docs](https://docs.powersync.com/maintenance-ops/self-hosting/usage-reporting)) |
| Updates | GitHub Releases and security advisories, with private vulnerability reporting and a `SECURITY.md` ([docs](https://docs.github.com/en/code-security/how-tos/report-and-fix-vulnerabilities/configure-vulnerability-reporting/configure-for-a-repository)). A weekly `flake.lock` bump rebuilds images on patched nixpkgs. [Diun](https://github.com/crazy-max/diun) (v4.33.1) sends update notices | Watchtower was archived on 2025-12-17 ([repo](https://github.com/containrrr/watchtower)). NixOS `system.autoUpgrade` exists, but only for patch-level pins (judgment) |

**Encryption at rest**

| Option | Protects against | Doesn't protect against | Default |
|---|---|---|---|
| Host disk or volume encryption (LUKS, provider volumes) | Stolen or recycled disks | Attacks "while the file system is mounted" ([PostgreSQL](https://www.postgresql.org/docs/current/encryption-options.html)); the provider's hypervisor | Recommended in the docs |
| App-level envelope encryption of secrets and raw bank payloads | Leaked dumps or backups; table reads through injection | A compromised running host, which holds the KEK in memory | On |
| Encrypted backups (`age`) | A leaked off-site copy | A host compromised at backup time | On |
| `pgcrypto` columns | — | Keys pass through the server ([PostgreSQL](https://www.postgresql.org/docs/current/encryption-options.html)) | No |
| End-to-end encryption of budgets | The operator | — breaks server-side ingestion, alerts and push ([sync-engine.md](sync-engine.md)) | No |

## 7. What one instance can't do alone (Apple, bank)

- **Native passkeys** (medium).
  - Apple: "Your app can't use passkeys to authenticate for services that you haven't configured as the app's associated domains" ([passkeys](https://developer.apple.com/documentation/authenticationservices/supporting-passkeys)). An app built by the maintainer can list only the domains known at build time.
  - The workaround: `ASWebAuthenticationSession` authenticates "through a web service, including one run by a third party" ([docs](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession)).
    - The passkey belongs to the instance's RP ID.
    - A custom-scheme callback carries a one-time code, which the app exchanges for the device-bound refresh token from [apple-client-stack.md](apple-client-stack.md).
  - Precedent: Nextcloud's iOS app merged "Add support for passkeys on login" through `ASWebAuthenticationSession` on 2026-02-21 ([PR 3996](https://github.com/nextcloud/ios/pull/3996)).
  - This changes apple-client-stack.md's plan, which used associated domains on a single maintainer domain. [native-apps-auth.md](native-apps-auth.md) turns it into a built-in authorization server with PKCE and DPoP-bound tokens.
- **Push** (medium).
  - APNs keys belong to the publisher's team ([apple-client-stack.md](apple-client-stack.md) §4.3). Since 2025-02-17 they can be limited to one environment (team-scoped keys) or to specific bundle IDs (topic-specific keys) ([Apple news](https://developer.apple.com/news/?id=wy4tb0uo)). Either way the signing key "must remain private to prevent anyone else from generating those tokens" ([Apple](https://developer.apple.com/documentation/usernotifications/establishing-a-token-based-connection-to-apns)), so self-hosters still can't push to the maintainer's builds without a relay.
  - Bitwarden: "only Bitwarden is allowed to send notifications to the store-distributed Android and iOS mobile applications", so self-hosted servers relay through its cloud, and "No decrypted data is ever sent in push notification payloads" ([Bitwarden](https://contributing.bitwarden.com/architecture/deep-dives/push-notifications/mobile/)).
    - Vaultwarden asks each self-hoster for an installation ID and key from bitwarden.com/host ([wiki](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-Mobile-Client-push-notification)).
  - Home Assistant pushes through Firebase, at most 500 per device per day, and the contents "are not encrypted" there ([companion](https://companion.home-assistant.io/docs/notifications/notification-details/)).

  | Push option | Needs | Verdict |
  |---|---|---|
  | Web Push (VAPID) | Nothing from Apple ([web-client-sync.md](web-client-sync.md)); on iOS only for installed web apps | ✓ in v1 |
  | The self-hoster's own APNs key | Their own Apple Developer Program membership and app build | ✓ in v1, and the maintainer's own setup |
  | A maintainer-run relay with encrypted payloads | A service run for other people, which PRODUCT.md rules out | Owner decision |

- **Bank consent.**
  - Each person registers their own Enable Banking restricted-mode app with a certificate whose key the instance generated and keeps. Nobody uploads a private key (§6; [connectors-and-license.md](connectors-and-license.md) §4.1).
  - That settles the owner's own app. A member's app run by an instance the owner administers stays "Grey; ask EB" ([bank-alternatives.md](bank-alternatives.md) §1.1), so members use file import until EB answers in writing (that note's spike 1).
  - The provider redirects to the instance's `https` callback. That should work on a private hostname as long as the user's browser can reach it; Firefly III's docs say the URL "does not have to be public" (secondary, cited in connectors-and-license.md §4.3; spike P6).

## Recommendation for on-budget

| Layer | Choice | Confidence |
|---|---|---|
| Build | One flake: package (API, web app, CLI, migrations), `nixosModules.default`, and an `image` from `buildLayeredImage` | Medium-high |
| Distribution | For compose installs: `ghcr.io/fveauvy/on-budget`, amd64 + arm64, attested, made public once; release-attached `compose.yaml` pinned by digest. The reference deployment uses no registry (ADR 0003) | High for the facts, medium for the choice |
| Runtime | on-budget plus vanilla PostgreSQL 18, and 19 once it passes CI; client tools for both majors in the image; jobs on Postgres (swift-jobs has a Postgres driver; TypeScript queues not checked); no Redis; Caddy or Tailscale Serve in front | Medium |
| Versions | SemVer; `stable`, `rc` and `edge` channels; no backports | Medium-high |
| Migrations | Forward-only, at start, under a lock, after an automatic dump; refuse a newer schema or an unsupported Postgres major; expand/contract | High |
| Client compatibility | Current and previous minor, at least 90 days; `/api/v1/meta`; a typed `client_too_old` error on one side, `minServerVersion` on the other | Medium |
| Backups | Hourly encrypted dumps (RPO up to 1 h) kept by age, a roles dump, pre-upgrade dumps kept apart, household export, a restore epoch with one accepted ID gap per client, a verify command | Medium |
| Administration | CLI bootstrap link, invites, households as tenants, the UI and CLI split of §5.3 | Medium-high |
| Security | §6 defaults: secrets from files, KEK, server-generated bank keys, loopback behind a TLS proxy and private by default, rate limits, audit log, a hardened container | Medium-high |
| Apple | `ASWebAuthenticationSession` sign-in; Web Push and own-key APNs; the relay is deferred | Medium |

**Minimal docs for self-hosters** (one page each; the configuration reference is generated from the config schema):

| Doc | Must cover |
|---|---|
| Self-hosting overview | Requirements: a Linux host, an HTTPS hostname (a `*.ts.net` name or one under the owner's domain), PostgreSQL 18 (19 once supported). Private by default (Tailscale). The security model: the operator can read all data, and there is no E2EE. What the instance can't do alone (§7) |
| Install with Docker Compose | Docker Engine 28.0.0 or later. Download the release bundle, `secrets generate` (files readable by UID 10001), `up`, bootstrap; Tailscale Serve, or the proxy profile for a public instance; the ufw warning |
| Install on NixOS | The flake input, a module example, agenix or sops-nix, a local Postgres with `services.postgresql.package = pkgs.postgresql_18` set explicitly |
| Configuration reference | Every key, its env var, its `_FILE` form and its NixOS option, with defaults |
| Reverse proxy and TLS | Private access first: Tailscale Serve or a `*.ts.net` certificate, DNS-01 for the owner's domain (lego on NixOS, a custom Caddy build on compose). Then public Caddy, nginx, Traefik; headers, SSE and WebSockets; trusted proxies; root path only |
| First run and administration | Bootstrap, households, invites, roles, devices, owner recovery, the emergency kit |
| Bank connections | Each person's own provider app with a server-generated key (certificate only), the EB terms caveat for members, the redirect URL, consent renewal, file import |
| Backups and restore | What is backed up (database and roles), encryption, retention, the one-hour RPO, off-site recipes, restore steps and what clients lose, the drill |
| Upgrading | Release notes (migrations yes/no, minimum client), the support window, rollback through a restore, Postgres majors and when 19 joins |
| Security checklist | Exposure, firewall, updates, audit log, `doctor`, reporting vulnerabilities |
| Troubleshooting | Health endpoints, `doctor`, logs, common proxy mistakes |

**Trade-offs the owner must accept.**
- Only Nix builds the image, so contributors without Nix depend on CI. The maintainer also keeps two install paths, NixOS and compose, tested in CI.
- There is no downgrade without a restore. A restore loses everything committed after its dump, members' acknowledged writes included, unless clients keep acknowledged mutations until a dump covers them (§4). Clients re-push what is still pending.
- The support window keeps N−1 mutation handlers alive, and contract migrations wait a release.
- The operator can read every household. Isolation protects against members and mistakes, not against whoever runs the host.
- Other self-hosters who use the maintainer's app builds get no APNs push unless someone runs a relay.
- Built-in backups are local; off-site copies are the self-hoster's job.
- Private by default means every member's device runs Tailscale, or reaches the instance another private way; the watch goes through the iPhone ([native-apps-auth.md](native-apps-auth.md)).

## Open questions and spikes

**Spikes** (each 1 day or less; pass means built by an agent and checked from CI):
- **P1 Image.** `buildLayeredImage` on both arches; measure size (with client tools for two Postgres majors) and push time; check the index, `gh attestation verify`, the source label, an anonymous pull after the one-time switch to public, and the compose smoke test. The smoke test starts on a fresh named volume, checks that `/var/lib/on-budget/backups` belongs to UID 10001 and that the pre-migration dump succeeds, and runs against both `postgres:18` and 19. Pass: one tag produces the image and the compose asset in under 15 minutes.
- **P2 VM tests.** A fresh install, an upgrade with migrations from the previous tag, and backup → wipe → restore with an epoch change, all in `nix flake check` on a GitHub runner with KVM. One test runs the image under Docker in the VM on a fresh volume, so a root-owned backup volume fails CI.
- **P3 Compatibility.** An N−1 client pushes during an upgrade; an attempted rollback is refused; a restore triggers re-bootstrap, the one-gap rule lets re-pushes through, and kept acknowledged mutations come back. The simulator measures what is still lost and how many replays are rejected.
- **P4 Native sign-in.** `ASWebAuthenticationSession` against a self-hosted origin on iOS and macOS, through TestFlight: create and use a passkey, plus the custom-scheme callback. Include a Tailscale-only hostname and a Mac whose default browser isn't Safari. native-apps-auth.md's S1 extends it.
- **P5 Push.** Web Push from a private instance to an installed iOS web app, and APNs with a self-built app.
- **P6 Bank callback.** An Enable Banking redirect to a private or Tailscale-only `https` URL.
- **P7 Footprint.** RAM and CPU of the two-container stack on a 1 vCPU / 1 GB VPS, and with PowerSync added ([architecture](https://docs.powersync.com/maintenance-ops/self-hosting/deployment-architecture)):
  - PowerSync's "Minimal Setup" is a "development" one ("e.g. for a staging or a QA environment") that "has no redundancy": a 512 MB / 1 vCPU compute container, "A single MongoDB node in replica set mode, 2GB memory, 1 vCPU", and a TLS load balancer.
  - Its production baseline: one replication container and 2+ API containers at 1 GB and 1 vCPU each, "A 3-node MongoDB replica set, 2+GB memory each", and "a daily compact job".
  - "It is currently required to host the API container on a dedicated subdomain", so every instance, Tailscale-only ones included, needs a second hostname and certificate.
  - The MongoDB variant can't fit the target. P7 therefore measures Postgres bucket storage: a separate database, which may share the server on Postgres 14 and later ([instance config](https://docs.powersync.com/configuration/powersync-service/self-hosted-instances)).

**Questions for the owner.**
1. How will other self-hosters get the native apps: the maintainer's TestFlight or App Store builds, or their own? This decides the push relay and the app's compatibility window.
2. Will the maintainer run a push relay for others, contrary to "no service run for other people", or stay with Web Push and own-key APNs?
3. Nix-only images, or a Dockerfile as well for contributors?
4. Is "current and previous minor, at least 90 days" the right support window?
5. Should off-site backup (S3 or restic) be built into v1, or only documented?
6. Should the owner see household data for support, or only metadata?
7. The licence is undecided. AGPL-3.0 is the most common among these references. It affects image labels and whether modified hosted copies must publish their source.
8. Is Docker Compose plus NixOS enough for v1, with no Kubernetes or Podman docs?
9. Is the reference deployment Tailscale-only, as ADR 0003 states, or public on 80/443, as [nixos-vps-infra.md](nixos-vps-infra.md) plans? This note follows the ADR.
10. Is an RPO of up to one hour acceptable, or should clients also keep acknowledged mutations until a dump covers them (§4), at the cost of a sync-engine.md amendment?

**Not verified.**
- Passkeys in `ASWebAuthenticationSession` on macOS with a non-Safari default browser; the Nextcloud precedent is iOS.
- Enable Banking redirects to non-public hostnames, and whether EB accepts a member's app run by the owner's instance (bank-alternatives.md spike 1).
- Nix image sizes for a Node or Swift server, and what client tools for two Postgres majors add (P1).
- That Docker's copy into a new volume keeps the directory's owner with every storage driver and in rootless mode. moby and continuity copy the owner, but I only read the code (P1, P2).
- Whether Tailscale Serve passes the client IP to the backend. Its page documents only Tailscale identity headers, and per-IP rate limits and EB's PSU headers depend on it.
- How often a restore's replayed mutations would be rejected, and the outbox cost of keeping acknowledged ones (P3).
- KVM on standard public-repo runners: the Nix installer actions enable it where available, but GitHub's runner page mentions only Android acceleration.
- Home Assistant tags beyond `stable`, Paperless-ngx and Firefly III tags beyond `latest`, Ghostfolio's backup guidance, and Home Assistant's secrets handling.
- TypeScript Postgres job queues (pg-boss, graphile-worker).

## Sources

Accessed 2026-11-07. Earlier notes: [sync-engine.md](sync-engine.md) · [web-client-sync.md](web-client-sync.md) · [apple-client-stack.md](apple-client-stack.md) · [bank-aggregation.md](bank-aggregation.md) · [bank-alternatives.md](bank-alternatives.md) · [connectors-and-license.md](connectors-and-license.md) · [factory-ci-quality.md](factory-ci-quality.md) · [nixos-vps-infra.md](nixos-vps-infra.md) · [native-apps-auth.md](native-apps-auth.md). Decisions: [ADR 0001](../adr/0001-self-hosted-single-owner-instances.md) · [ADR 0003](../adr/0003-nixos-just-tailscale-infrastructure.md).

- **Actual Budget:** [Docker install](https://actualbudget.org/docs/install/docker) · [HTTPS](https://actualbudget.org/docs/config/https) · [config](https://actualbudget.org/docs/config/) · [multi-user](https://actualbudget.org/docs/config/multi-user) · [backup](https://actualbudget.org/docs/backup-restore/backup) · [compose](https://github.com/actualbudget/actual/blob/master/packages/sync-server/docker-compose.yml) · [Dockerfile](https://github.com/actualbudget/actual/blob/master/sync-server.Dockerfile) · [releases](https://github.com/actualbudget/actual/releases) · sync-server source: [app-account.js](https://github.com/actualbudget/actual/blob/master/packages/sync-server/src/app-account.js), [account-db.js](https://github.com/actualbudget/actual/blob/master/packages/sync-server/src/account-db.js), [app.ts](https://github.com/actualbudget/actual/blob/master/packages/sync-server/src/app.ts) (mounts them at `/account`)
- **Firefly III:** [compose](https://github.com/firefly-iii/docker/blob/main/docker-compose.yml) · [.env.example](https://github.com/firefly-iii/firefly-iii/blob/main/.env.example) · [upgrade](https://docs.firefly-iii.org/how-to/firefly-iii/upgrade/docker/) · [backup](https://docs.firefly-iii.org/how-to/firefly-iii/advanced/backup/) · [FAQ](https://docs.firefly-iii.org/references/faq/firefly-iii/general/) · [what it isn't](https://docs.firefly-iii.org/explanation/more-information/what-its-not/) · [releases](https://github.com/firefly-iii/firefly-iii/releases)
- **Immich:** [v3.3.6 compose](https://github.com/immich-app/immich/releases/download/v3.3.6/docker-compose.yml) · [example.env](https://github.com/immich-app/immich/releases/download/v3.3.6/example.env) · [upgrading](https://docs.immich.app/install/upgrading) · [backup and restore](https://docs.immich.app/administration/backup-and-restore) · [post-install](https://docs.immich.app/install/post-install) · [reverse proxy](https://docs.immich.app/administration/reverse-proxy) · [releases](https://github.com/immich-app/immich/releases)
- **Paperless-ngx** (read from repo sources): [administration](https://docs.paperless-ngx.com/administration/) · [configuration](https://docs.paperless-ngx.com/configuration/) · [setup](https://docs.paperless-ngx.com/setup/) · [compose](https://github.com/paperless-ngx/paperless-ngx/blob/main/docker/compose/docker-compose.postgres.yml) · [releases](https://github.com/paperless-ngx/paperless-ngx/releases)
- **Vaultwarden and Bitwarden:** [.env.template](https://github.com/dani-garcia/vaultwarden/blob/main/.env.template) · [image tags](https://github.com/dani-garcia/vaultwarden/wiki/Which-container-image-to-use) · [HTTPS](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS) · [backups](https://github.com/dani-garcia/vaultwarden/wiki/Backing-up-your-vault) · [admin page](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-admin-page) · [hardening](https://github.com/dani-garcia/vaultwarden/wiki/Hardening-Guide) · [push](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-Mobile-Client-push-notification) · [Dockerfile](https://github.com/dani-garcia/vaultwarden/blob/main/docker/Dockerfile.debian) · [Bitwarden push relay](https://contributing.bitwarden.com/architecture/deep-dives/push-notifications/mobile/)
- **Home Assistant:** [Linux install](https://www.home-assistant.io/installation/linux) · [backups](https://www.home-assistant.io/common-tasks/general/backups/) · [authentication](https://www.home-assistant.io/docs/authentication/) · [companion notifications](https://companion.home-assistant.io/docs/notifications/notification-details/) · [releases](https://github.com/home-assistant/core/releases)
- **Ghostfolio, Maybe:** [Ghostfolio README](https://github.com/ghostfolio/ghostfolio/blob/main/README.md) · [compose](https://github.com/ghostfolio/ghostfolio/blob/main/docker/docker-compose.yml) · [entrypoint](https://github.com/ghostfolio/ghostfolio/blob/main/docker/entrypoint.sh) · [Maybe repo](https://github.com/maybe-finance/maybe) · [final release](https://github.com/maybe-finance/maybe/releases/tag/v0.6.0) · [Docker guide](https://github.com/maybe-finance/maybe/blob/main/docs/hosting/docker.md) · [compose example](https://github.com/maybe-finance/maybe/blob/main/compose.example.yml) · [Sure fork](https://github.com/we-promise/sure)
- **Other precedents:** [Gitea PR 19504](https://github.com/go-gitea/gitea/pull/19504) · [GitLab upgrade paths](https://docs.gitlab.com/update/upgrade_paths/) · [Jenkins unlock](https://www.jenkins.io/doc/book/installing/linux/) · [Portainer setup](https://docs.portainer.io/start/install-ce/server/setup) · [Nextcloud iOS PR 3996](https://github.com/nextcloud/ios/pull/3996) · [headscale flake](https://github.com/juanfont/headscale/blob/main/flake.nix) · [headscale goreleaser](https://github.com/juanfont/headscale/blob/main/.goreleaser.yml) · [headscale container-main workflow](https://github.com/juanfont/headscale/blob/main/.github/workflows/container-main.yml) · [ko](https://github.com/ko-build/ko) · [attic packages](https://github.com/zhaofengli/attic/blob/main/flake/packages.nix) · [attic push script](https://github.com/zhaofengli/attic/blob/main/.ci/build-and-push-images.sh)
- **Nix:** [dockerTools](https://github.com/NixOS/nixpkgs/blob/master/doc/build-helpers/images/dockertools.section.md) · [nix2container](https://github.com/nlewo/nix2container) · [NixOS manual](https://nixos.org/manual/nixos/stable/) · [PostgreSQL module docs](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/databases/postgresql.md) · PostgreSQL module [26.05](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/nixos/modules/services/databases/postgresql.nix), [master](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/databases/postgresql.nix) · [security.acme module (26.05)](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/nixos/modules/security/acme/default.nix) · [Caddy vhost options (26.05)](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/nixos/modules/services/web-servers/caddy/vhost-options.nix) · [tailscale-serve.nix (26.05)](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/nixos/modules/services/networking/tailscale-serve.nix) · [NixOS 26.11 schedule](https://github.com/NixOS/nixpkgs/issues/567567) · [postgresqlBackup](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/backup/postgresql-backup.nix) · [restic module](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/backup/restic.nix) · modules: [immich](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/web-apps/immich.nix), [actual](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/web-apps/actual.nix), [vaultwarden](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/security/vaultwarden/default.nix), [firefly-iii](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/web-apps/firefly-iii.nix), [paperless](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/misc/paperless.nix), [home-assistant](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/home-automation/home-assistant.nix), [nextcloud](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/web-apps/nextcloud.nix) · [RFC 42](https://github.com/NixOS/rfcs/blob/master/rfcs/0042-config-option.md) · [JS packaging](https://github.com/NixOS/nixpkgs/blob/master/doc/languages-frameworks/javascript.section.md) · Swift packaging [master](https://github.com/NixOS/nixpkgs/blob/master/doc/languages-frameworks/swift.section.md), [26.05](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/doc/languages-frameworks/swift.section.md) · Swift version [master](https://github.com/NixOS/nixpkgs/blob/master/pkgs/development/compilers/swift/swift-version), [26.05](https://github.com/NixOS/nixpkgs/blob/nixos-26.05/pkgs/development/compilers/swift/sources.nix) · [experimental features](https://nix.dev/manual/nix/latest/development/experimental-features) · [Determinate Nix 3.0 thread](https://discourse.nixos.org/t/determinate-nix-3-0/61202) (secondary) · [agenix](https://github.com/ryantm/agenix) · [sops-nix](https://github.com/Mic92/sops-nix) · [install-nix-action](https://github.com/cachix/install-nix-action) · [nix-installer-action](https://github.com/DeterminateSystems/nix-installer-action)
- **GitHub and containers:** [Packages billing](https://docs.github.com/en/billing/concepts/product-billing/github-packages) · [hosted runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners) · [artifact attestations](https://docs.github.com/en/actions/concepts/security/artifact-attestations) · [verifying attestations](https://docs.github.com/en/actions/how-tos/secure-your-work/use-artifact-attestations/use-artifact-attestations) · [actions/attest](https://github.com/actions/attest) · [attest-build-provenance](https://github.com/actions/attest-build-provenance) · [private vulnerability reporting](https://docs.github.com/en/code-security/how-tos/report-and-fix-vulnerabilities/configure-vulnerability-reporting/configure-for-a-repository) · [buildx imagetools](https://docs.docker.com/reference/cli/docker/buildx/imagetools/create/) · GHCR [container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry), [visibility](https://docs.github.com/en/packages/learn-github-packages/configuring-a-packages-access-control-and-visibility) · [Docker port publishing](https://docs.docker.com/engine/network/port-publishing/) · [Docker and firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/) · [Docker volumes](https://docs.docker.com/engine/storage/volumes/) · [moby volume copy](https://github.com/moby/moby/blob/master/daemon/container/container_unix.go) · [continuity `CopyDir`](https://github.com/containerd/continuity/blob/main/fs/copy.go) · [Compose services](https://docs.docker.com/reference/compose-file/services/) · [Compose networking](https://docs.docker.com/compose/how-tos/networking/) · [Compose secrets](https://docs.docker.com/compose/how-tos/use-secrets/) · [postgres image docs](https://github.com/docker-library/docs/blob/master/postgres/content.md) · [postgres 18 Dockerfile](https://github.com/docker-library/postgres/blob/master/18/trixie/Dockerfile) · [postgres 18 entrypoint](https://github.com/docker-library/postgres/blob/master/18/trixie/docker-entrypoint.sh) · [caddy image docs](https://github.com/docker-library/docs/blob/master/caddy/content.md) · [Diun](https://github.com/crazy-max/diun) · [Watchtower](https://github.com/containrrr/watchtower)
- **PostgreSQL and PowerSync:** [upgrading](https://www.postgresql.org/docs/current/upgrading.html) · [versioning](https://www.postgresql.org/support/versioning/) · [19 RC 1 announcement](https://www.postgresql.org/about/news/postgresql-19-rc-1-released-3409/) · pg_dump [18](https://www.postgresql.org/docs/current/app-pgdump.html), [17](https://www.postgresql.org/docs/17/app-pgdump.html) · [pg_dumpall](https://www.postgresql.org/docs/18/app-pg-dumpall.html) · [SQL dump (§25.1)](https://www.postgresql.org/docs/18/backup-dump.html) · [initdb 18](https://www.postgresql.org/docs/18/app-initdb.html) · [encryption options](https://www.postgresql.org/docs/current/encryption-options.html) · [PowerSync self-hosting](https://docs.powersync.com/intro/self-hosting) · [source database setup](https://docs.powersync.com/configuration/source-db/setup) · [instance config](https://docs.powersync.com/configuration/powersync-service/self-hosted-instances) · [deployment architecture](https://docs.powersync.com/maintenance-ops/self-hosting/deployment-architecture) · [usage reporting](https://docs.powersync.com/maintenance-ops/self-hosting/usage-reporting)
- **Security and Apple:** [systemd credentials](https://systemd.io/CREDENTIALS/) · [OWASP logging cheat sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) · [Caddy automatic HTTPS](https://caddyserver.com/docs/automatic-https) · [Tailscale Serve](https://tailscale.com/kb/1312/serve) · [RFC 9110 §15.5.22](https://www.rfc-editor.org/rfc/rfc9110#section-15.5.22) · [Replicache dynamic pull](https://v12.doc.replicache.dev/byob/dynamic-pull) · Enable Banking [ToS](https://enablebanking.com/terms), [API reference](https://enablebanking.com/docs/api/reference/) · APNs [key types (news, 2025-02-17)](https://developer.apple.com/news/?id=wy4tb0uo), [token-based connection](https://developer.apple.com/documentation/usernotifications/establishing-a-token-based-connection-to-apns) · [Supporting passkeys](https://developer.apple.com/documentation/authenticationservices/supporting-passkeys) · [ASWebAuthenticationSession](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession)
