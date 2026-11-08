# AGENTS.md — edueasy-platform-knowledge-base

Instructions for AI coding agents (Claude Code, Cline, Copilot, Codex). `CLAUDE.md` imports this file, so keep everything here.

## What this repo is
The **only** home of EduEasy platform knowledge. `EDUEASY-KNOWLEDGE-BASE.md` is the single source of truth for what the platform is, how the 9 APIs fit together, user journeys, observability and the architecture law. Service repos link here instead of keeping their own copies.

- `EDUEASY-KNOWLEDGE-BASE.md` — the KB body (wins on conflict)
- `CHANGELOG.md` — KB version history (SemVer)
- `references/` — point-in-time audits; superseded by the KB body on conflict
- `per-api/` — per-service reference material

## Rules
- Every content change bumps the KB version (SemVer) in the KB header **and** adds a `CHANGELOG.md` entry: patch = wording/corrections, minor = new facts or sections, major = changes to the architecture law.
- §6.3 non-negotiables change only via a version bump plus review — never as a side effect of another edit.
- State facts that can be verified in the code (ports, schemas, flows) and keep them in sync with the service repos; when you find a mismatch, fix the KB or flag it — don't guess.
- Keep journey wording precise (§5.1 invariant: choices before payment; submission gated on payment verification or sponsorship; EFT proof pending until approved).
- Dated audit findings go in `references/`; only the conclusions move into the KB body.
