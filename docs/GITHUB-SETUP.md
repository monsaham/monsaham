# Phase 1 GitHub setup plan

Status: BLOCKED for live configuration. Remote is https://github.com/monsaham/monsaham.git. No GitHub connector tools or `gh` CLI were available; public web retrieval returned cache miss. Remote protections, labels, milestones and projects are UNKNOWN, not absent. No remote mutation attempted.

## Owner execution after explicit approval

1. Open the repository with an account allowed to inspect settings. Inspect existing rulesets/classic protection for `main` before editing. Require pull requests, require status checks and disallow force pushes, exactly as WORKFLOW Phase 1 specifies. Reconcile existing stricter settings with owner; do not replace them blindly. Select actual check names only after Antigravity's issue #4 produces runs. Protection with no required checks is not completion.
2. Inspect Issues → Labels. Reuse exact names; create only missing labels: `claude-code`, `codex`, `cline`, `antigravity`, `bug`, `blocked`, `kontrak`. Descriptions respectively: Claude Code ownership; Codex ownership; Cline ownership; Antigravity ownership; defect; blocked dependency/decision; ARCHITECTURE §12 contract change. Colors are optional presentation choices, not workflow policy.
3. Inspect open AND closed milestones. Reuse exact `v1.0 MVP`, the name in WORKFLOW (the request's `v1.0` is shorthand). Proposed description: “MVP defined by SPEC.md; delivery and ownership follow WORKFLOW.md.” No invented due date. Resolve any existing similarly named milestone before creating another.
4. Inspect repository-linked and owner/organization Projects, including archived boards. Reuse a suitable Mon Saham board and its existing Status field. Required status options: Todo, In Progress, Review, Done. Map existing items before any approved changes. Do not create a duplicate board while inventory is unknown.
5. Review `.github/ISSUE_TEMPLATE/task.md` locally. Owner must approve a checkpoint/PR before publishing it. No issues have been created and no GitHub account has been assigned to an agent label.
6. Reserve CI implementation for Antigravity, WORKFLOW Phase 3 wave 1 issue #4. This phase provides a specification only; `.github/workflows/` stays empty.

## CI ownership specification for issue #4 (not implemented)

| Branch prefix | Allowed paths / source of authority |
|---|---|
| claude/ | Exact Claude Code row in AGENTS §2, tests in TESTING §4; exclude Codex subtrees and cron |
| codex/ | server/market/**, server/ai/**, server/jobs/**, server/bin/cron.js; only Codex tests/mocks/fixtures/factories in TESTING §4 |
| cline/ | client/** |
| anti/ | .github/**, docs/**, README.md, root configuration |
| docs/ | Owner only: root Markdown, server/config/constants.js, server/config/marketHolidays.js, server/jest.config.js |

Preserve owner-only files from AGENTS §2; root configuration must not grant access to those server config files. Test ownership is per tested file, not unrestricted `server/__tests__/**`. Shared mocks retain their named owners. Flag the formatter/complianceFilter cross-owner contract gap in the audit for owner resolution.

Antigravity must specify classification for added/modified/deleted/renamed files and unknown prefixes, and resolve ambiguity with owner before enforcing policy. Owner identity cannot be established from the `docs/` prefix alone. WORKFLOW.md remains immutable for this request even though the general docs/ rule includes root Markdown.

Later CI runs install, lint, unit/integration/e2e, coverage and frontend build per TESTING. No fake passing checks, guessed required-check names, credentials, workflows or package files are added here. The existing workflow wave order is preserved.

## Git checkpoint recommendation (not executed)

Repository is already initialized: do not run git init or add a duplicate origin. Current branch is `aya`; local main/origin/main point to 674a871 and aya/origin/aya to 776a497 (local refs, not a live remote verification). The latter adds TESTING.md and WORKFLOW.md. Owner should review and reconcile that branch history through a PR; do not reset or rename branches here.

Inspect `git status --short`, `git diff --check`, `git diff` and every untracked candidate before staging exact approved paths. Preserve the pre-existing untracked PRD_MON_SAHAM_REVISED.md unless explicitly approved. Do not use broad git add -A without reviewing the index for secrets. Review `git diff --cached` before committing. Use an owner-approved docs/ branch for the preparation checkpoint; subsequent agent implementation retains its original prefixes. Split configuration/rule synchronization and readiness documentation into reviewable checkpoints if needed for AGENTS' ~200-line guideline. Commit, push and remote changes remain pending explicit authorization.
