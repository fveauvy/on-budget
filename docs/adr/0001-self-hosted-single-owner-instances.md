---
status: accepted
---

# Self-hosted instances with one owner and no public sign-up

Shoestring is open source and self-hosted. Each instance (server and database) belongs to one owner, who creates its households, such as a test household and a production household, and adds their members. There is no public sign-up and no service run for other people; anyone who wants Shoestring hosts their own instance. The maintainer chose this over a hosted service for friends and family: each household's bank data stays on its own server, and no one provides an account-information service to third parties.

## Consequences

- The repository is public. People appear by role, and personal details and plaintext secrets stay out of commits (see `AGENTS.md`).
- Households in one instance are isolated from each other.
