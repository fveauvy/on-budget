---
status: accepted
---

# Forward-only upgrades with a client support window

Releases follow SemVer as `vX.Y.Z` tags, with no channels and no backports, and come from the project's flake: one package (API, worker, host CLI, web build and SQL migrations) and the `services.shoestring` NixOS module, checked by VM tests of a fresh install, an upgrade from the previous release, a rollback to it, and a backup, wipe and restore. At start, the server dumps the database, then applies forward-only migrations under an advisory lock. Each migration records the schema's floor, the oldest server version that can run on the schema it leaves: an expand step leaves the floor where it is, and a contract step, shipped at least one minor release after its expand, raises it to the first release that no longer reads what the step removes. A server refuses to start on a schema whose floor is above its own version, or on an unsupported PostgreSQL major, so going back to the previous release needs no restore unless that release raised the floor beyond it, which its notes state. The instance serves apps of the current and previous minor release for at least 90 days, with the glance version and watch contract they use, through `/api/v1/meta`, a `client_too_old` error and, in each app, a minimum server version set to the previous minor. Why: devices stay offline for days, and the web app updates with each deploy while native builds reach devices only after it, so an instance and its apps always run mixed versions; and a NixOS generation rollback leaves the schema as it is.

## Considered Options

- Refusing any schema newer than the server (Gitea's rule): every rollback after a migration would then need a restore, the automatic ones included, such as deploy-rs's magic rollback, which would leave the instance down.
- Devices keeping acknowledged mutations until a dump covers them, with the last dumped sync ID in every pull (`self-hosting.md` §4): it would recover what a restore loses, at the cost of protocol state, simulator cases and a watermark that the backup job would have to read inside the dump's snapshot and publish only once every off-site copy succeeded. Restores are rare, and the recovery point is an hour.

## Consequences

- A restore rotates an instance epoch. Devices keep their client IDs and push their pending mutations, which the instance accepts with one gap in each device's mutation IDs, then bootstrap anew; what the instance had acknowledged after the restored backup is lost, and each app says how many of its own changes were.
- Releases whose notes say they raise the floor are deployed with magic rollback off (`docs/architecture.md`, The reference deployment).
- New features check `/api/v1/meta`'s `features`, so a newer app keeps working against the previous minor's server.
- ADR 0003 binds only the reference deployment. An OCI image and a compose bundle for other self-hosters follow once the server is stable, built from the same flake, with no Dockerfile.
- Contributors without Nix rely on CI to build the package.
