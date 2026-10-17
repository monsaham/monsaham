---
name: Agent task
about: One task, one agent, one branch, one PR following WORKFLOW.md
title: ""
labels: ""
assignees: ""
---

## Task title and description
Describe the expected behavior and reference the WORKFLOW issue/wave.

## Assigned agent
Choose Claude Code / Codex / Cline / Antigravity and the matching label:
claude-code / codex / cline / antigravity. GitHub assignee is the owner-selected account.

## Scope and ownership
- Allowed files (exact paths):
- Out of scope:
- Branch: claude/<issue>-<name>, codex/<issue>-<name>, cline/<issue>-<name>, or anti/<issue>-<name>
- Owner-only document work uses docs/<name>.
- Preserve AGENTS.md §2 and TESTING.md §4 ownership.

## Acceptance criteria
- [ ] Expected behavior and edge cases specified
- [ ] SPEC/ARCHITECTURE/ERD references listed
- [ ] No unapproved dependency, product, or shared-contract changes

## Dependencies and contracts
- Required issues merged first:
- ARCHITECTURE.md §12 functions consumed/provided:
- Unresolved owner decisions (use blocked; contract changes use kontrak):

## Testing requirements and evidence
- Test scenarios and applicable TESTING.md coverage thresholds:
- For implementation: test-first RED output, then GREEN and coverage output
- For documentation/configuration: relevant non-destructive validation
- No real provider/AI/Telegram calls in CI; no secrets in evidence
- PR follows AGENTS.md §5; split changes around the ~200-line guidance
