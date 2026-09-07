# Agent Skills & Rules — Placement Convention

**Rule (do not repeat):** install agent **skills** under the platform's cline skills directory:
`.cline/skills/<skill-name>/` — the **same directory** as existing cline skills
(e.g. `.cline/skills/frontend-design/`). **Never** put agent rules/skills into a per-repo location
like `.cline/rules/` or `.agents/` inside a service repo — those should stay free of agent tooling.

For reusable compound skill sets that span many repos (like Clean Architecture + DDD + Clean Code +
Refactoring from `ciembor/agent-rules-books`), the intended home is a single shared skill folder in
`.cline/skills/`, e.g.:

```
.cline/skills/
  frontend-design/            # existing
  clean-architecture-ddd/     # compound skill (this repo's discipline rules)
    SKILL.md
    clean-architecture.nano.md
    clean-code.nano.md
    domain-driven-design.nano.md
    refactoring.nano.md
```

Clone the source (`ciembor/agent-rules-books`) **outside** the workspace (e.g. `/tmp` or `~/.cline/skills`)
so it never lands in a git repo, then copy the needed `.nano.md` files into the skill folder.
The skill folder may be committed to the parent repo (`edueasy-org`) — that is the intended single
shared location — but **must not** be scattered into the nested service repos.

See `docs/architecture-review.md` for the current review content those rules were applied to.