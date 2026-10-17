# monsaham
sebuah bot telegram untuk memantau perkembangan saham


## Preparation status

Phase 0–1 local preparation only; application code and package manifests do not exist yet.
`WORKFLOW.md` is the immutable execution reference. Read `SPEC.md`, `ARCHITECTURE.md`,
`ERD.md`, `STYLE.md`, `TESTING.md` and the synchronized agent rules before implementation.
`SPEC 1.1.md` and the unapproved `PRD_MON_SAHAM_REVISED.md` are not automatically adopted.

- [Audit, conflicts and readiness](docs/PHASE-0-1-AUDIT.md)
- [Provider evidence and official IDX verification](docs/PROVIDER-READINESS.md)
- [Environment and Telegram owner steps](docs/ENVIRONMENTS.md)
- [GitHub setup and deferred CI ownership](docs/GITHUB-SETUP.md)

Use `.env.example` for the safe variable inventory; secrets belong only in ignored
private environment files or an environment-specific secret store. The existing empty
`env-example` is a legacy filename retained for owner review. Do not assume the template
is a working runtime configuration. No install/start/test commands are available yet.
Provider/licensing, IDX verification, contract decisions, three bot identities and
live GitHub configuration remain unresolved. Phase 2 has not started.
