# Shoestring

## Read first

- `PRODUCT.md` (what and why), `SPEC.md` (behavior and numbers), `GLOSSARY.md` (terms), `docs/adr/` (decisions).
- `docs/architecture.md` maps the runtime design; `docs/factory.md` owns tooling, CI and releases.
- Precedence: `PRODUCT.md` and the ADRs win, then `SPEC.md`, then `docs/factory.md` and `docs/architecture.md`. `docs/research/` is background: follow a note only where one of those adopts it.

## Public repository

This repository is public, and Shoestring is self-hosted open source. Write docs and code for any self-hoster: refer to people by role (maintainer, owner, member, partner) and to banks as supported banks. Personal details (who banks where, home infrastructure, the maintainer's instance address, devices, emails, account identifiers) and plaintext secrets stay out of commits, issues and pull requests. Spike results about household devices, banks, networks or members go to the private ops repository; the public ticket gets only the generic outcome and the decision.

## Pull requests

Titles are Conventional Commits and become the squash commit. The body carries `Closes #<n>`.

## Agent skills

### Issue tracker

GitHub Issues on `fveauvy/shoestring`, via `gh`. See `docs/agents/issue-tracker.md`.

### Triage labels

The five canonical triage roles, each label named after its role. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: root `GLOSSARY.md` + `docs/adr/`. See `docs/agents/domain.md`.
