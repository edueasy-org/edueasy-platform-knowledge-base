# EduEasy Knowledge Base (KB)

**Start here: [`EDUEASY-KNOWLEDGE-BASE.md`](./EDUEASY-KNOWLEDGE-BASE.md)** — the single source of truth (KB v1.1.0).

This repository (`edueasy-platform-knowledge-base`) is the **only** home of EduEasy platform knowledge. No service repo keeps its own KB copy.

```
edueasy-platform-knowledge-base/
├── EDUEASY-KNOWLEDGE-BASE.md   ← THE file: what / why / how / journeys / law
├── CHANGELOG.md                ← KB version history (SemVer)
├── README.md                   ← this index
├── references/                 ← point-in-time audits (superseded by KB body on conflict)
│   ├── architecture-review.md
│   ├── clean-architecture-ddd-review.md
│   ├── frontend-architecture-review.md
│   └── agent-rules-placement.md
└── per-api/                    ← service-level guides (chatbot-api suite: spec, architecture,
    ├── architecture.md            security, deployment, testing, setup, env template)
    ├── api-specification.md
    ├── api-usage-guide.md
    ├── security-guide.md
    ├── deployment-guide.md
    ├── setup-and-configuration-guide.md
    ├── testing-guide.md
    ├── test-cases-specification.md
    ├── production-readiness-checklist.md
    ├── quick-reference.md
    └── application-env-template.yml
```

Versioning rules: see KB §9. Any refactor/feature touching architecture, topology, flows, journeys, or observability updates the KB and bumps the version in the same PR; bumps are tagged `kb-vX.Y.Z`.
