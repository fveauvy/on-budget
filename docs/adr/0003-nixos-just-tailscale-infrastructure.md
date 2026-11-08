---
status: accepted
---

# NixOS flake, just and Tailscale for the reference deployment

Shoestring's server is packaged in the project's Nix flake with a NixOS module. The reference deployment runs it on an EU VPS whose whole system is declared in a separate private ops repository that imports Shoestring as a flake input, deployed with `just` recipes and reachable only over Tailscale, following [dreamsofcode-io/nixos-vps](https://github.com/dreamsofcode-io/nixos-vps). The maintainer chose this over Docker-based deployment because the configuration stays the single source of truth: a failed build leaves the server untouched, previous generations can be rolled back, and no container registry is needed.

## Consequences

- Every instance must work when it is reachable only over a private network, and nothing may depend on inbound connections from the internet: bank-consent returns go through the member's browser, and push notifications go out to APNs and Web Push.
- The reference deployment's host answers only Tailscale on its public interface and publishes nothing, so its certificate comes from ACME DNS-01 for the permanent instance address (ADR 0011).
- Host details, network settings and encrypted secrets live in the private ops repository, never in this one. The ops template's defaults, which any self-hoster may adopt, are described in `docs/architecture.md` (The reference deployment), so tuning them needs no ADR.
- Updating the deployment means pointing the ops repository at a new release tag, then running `just deploy`, by hand for now. Unattended deploys from CI would need GitHub Pro on the private repository, for environment secrets and a `main`-only rule.
- SP-05 confirms or amends the reference deployment's design.
