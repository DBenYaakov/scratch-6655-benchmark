<!--
######################################################################################################
© 2024–2026 ĀYŌDÈ: Organization for the Advancement of Youth through Opportunities in Digital Education.
All rights reserved.

This software is confidential and proprietary information of ĀYŌDÈ.
It is intended solely for internal use by authorized personnel within ĀYŌDÈ.
Any unauthorized use, distribution, or reproduction of this software is strictly prohibited.

https://ayode.institute
######################################################################################################
-->

# Silicon Skills

This directory contains formal agent skills derived from the silicon-based
repository procedures. Each skill lives in its own directory with a
`SKILL.md`; most link their underlying procedure as `references/guide.md`.
In the index below, a skill whose canonical guidance lives entirely under
its own directory (its `SKILL.md`, plus any skill-local guide) lists the
literal value `Source: self-contained` instead of a guide path.

**Guide placement rule.** Three cases, exhaustive over every taxonomy
document:

1. **Owned, shared standard or reusable procedure** (code style, SQL
   conventions, migrations, PR mechanics, an MCP tool run) — the skill
   symlinks its `references/guide.md` into the matching
   `../{operations,delivery,implementation,standards,validation,workflows,session}/`
   taxonomy directory, and the index lists that path as its `Source:`.
2. **Owned, one-off workflow specific to that skill** — the skill keeps a
   real local `references/guide.md` and is listed as `Source: self-contained`.
3. **Unowned shared reference** — a document that lives in a taxonomy
   directory but backs no skill's `references/guide.md`. This is not a
   defect; it is registered, with its declared readers, in the
   [`## Shared reference documents (no owning skill)`](#shared-reference-documents-no-owning-skill)
   section below. Adding a new taxonomy document that backs no skill requires
   adding a row there in the same change.

Every guide-backed skill (cases 1 and 2) also carries a distilled
`## Workflow Essentials` block in its `SKILL.md` that must be kept consistent
with its guide. The index below is grouped by lifecycle phase to mirror the
CLAUDE.md task-to-skill routing table.

These skills are registered with Claude Code through symlinks in the repo-root
`.claude/skills/` directory, so they load into agent context automatically and
are invocable as `/<skill-name>`. This directory remains the canonical
location — edit skills here, never through the symlinks.

## Shared reference documents (no owning skill)

These are the taxonomy documents under
`../{operations,delivery,implementation,standards,validation,workflows}/`
that no `skills/*/references/guide.md` symlink targets (`docs/silicon-based-how/session/`
is out of scope by definition — see the guide placement rule above). Each is
a **shared reference** (case 3): changing one obliges updating every declared
reader in the same change (CLAUDE.md "Skills Maintenance"). The reader
column is the live output of
`grep -rl <basename> docs/silicon-based-how/skills/` — re-run it rather than
trusting this table for anything except line count and the fact of
registration. **That command finds PATH references only; a file that restates
this document's content without naming it will not appear** (observed twice on
train 20260809B), so pair it with a content grep for a distinctive phrase of
whatever you changed. DOC-5 in `implementation-validation.md`
(`docs/silicon-based-how/skills/guide-ownership-check.sh`) asserts this table
against the tree in both directions (a newly-unowned document missing here
fails, and a row that has since gained an owning skill also fails).

| Document | Lines | Declared readers |
|---|---|---|
| `operations/mcp-tools.md` | 4,963 | `aws-sso-cache-bootstrap/SKILL.md`, `aws-sso-cache-bootstrap/references/guide.md`, `daily-startup/SKILL.md`, `daily-startup/references/guide.md`, `deploy-cloudformation-template/SKILL.md`, `dispatcher-classify-issue/references/guide.md`, `dispatcher-enqueue-pending/SKILL.md`, `dispatcher-enqueue-pending/references/guide.md`, `github-issue-queue-resolve/SKILL.md`, `github-issue-queue-resolve/references/guide.md`, `pr-cycle-development/references/guide.md`, `pr-precheck-development/SKILL.md`, `pr-review-development/SKILL.md` |
| `operations/github-rest-api-version.md` | 407 | `code-implementation-validate/SKILL.md`, `github-issue-close-not-reproducible/SKILL.md`, `github-issue-create/references/guide.md`, `github-issue-resolve-reuse-branch/SKILL.md`, `github-issue-resolve/references/guide.md`, `github-issue-status-update/SKILL.md`, `github-project-query/SKILL.md`, `github-project-query/scripts/github_smart_query.py`, `github-sprint-api-check-group-resolve/SKILL.md`, `github-sprint-close-not-reproducible/SKILL.md`, `pr-cycle-development-bundle/references/guide.md`, `use-case-backend-analyze/references/guide.md`, `worktree-sweep/SKILL.md` |
| `operations/github-bulk-write-pacing.md` | snapshot; see file | `github-issue-status-update/SKILL.md`, `github-sprint-close-not-reproducible/SKILL.md`, `github-issue-close-not-reproducible/SKILL.md`, `github-sprint-api-check-group-resolve/SKILL.md`, `github-issue-queue-resolve/SKILL.md`, `github-issue-queue-resolve/references/guide.md`, `dispatcher-enqueue-pending/SKILL.md`, `dispatcher-enqueue-pending/references/guide.md`, `github-project-query/SKILL.md`; peer-doc consumers: `delivery/issue-status.md`, `operations/github-issue-lifecycle-webhook-manual-redelivery.md` |
| `delivery/coordinator-verdict-acceptance.md` | 415 | `github-issue-queue-resolve/SKILL.md`, `github-issue-queue-resolve/references/guide.md`, `pr-cycle-development-bundle/SKILL.md`, `pr-cycle-development-bundle/references/guide.md`, `pr-cycle-development/SKILL.md`, `pr-cycle-development/references/guide.md`, `pr-cycle-production/SKILL.md`, `pr-cycle-production/references/guide.md`, `pr-review-development/SKILL.md`, `pr-review-production/SKILL.md` |
| `operations/agent-memory-fold-ledger.md` | 833 | `agent-memory-consolidate/SKILL.md`, `daily-startup/SKILL.md`, `daily-startup/references/guide.md` |
| `implementation/merlin-domain-facades.md` | 228 | (none) |
| `operations/github-issue-lifecycle-webhook-manual-redelivery.md` | 668 | (none) |
| `operations/train-dispatcher-ttl-sizing.md` | 158 | `github-issue-queue-resolve/SKILL.md`, `github-issue-queue-resolve/references/guide.md` |
| `operations/aws-sso-sandbox-cache.md` | 128 | `aws-sso-cache-bootstrap/SKILL.md`, `aws-sso-cache-bootstrap/references/guide.md` |
| `standards/poc-conventions.md` | 72 | `cross-repo-issue-resolve/references/guide.md` |

Line counts are a snapshot as of issue #3599; they drift (`mcp-tools.md` grew
32% in the two days before this table was written) and are not themselves
checked. A document with zero declared readers today is still registered —
future readers are added here, not inferred.

## Operating Rules

The repository-wide agent operating rules that previously lived in this file
(MCP-first tool execution, deployed-state immutability, build and readiness
gates, deployment handoff, new-endpoint routing and Lambda readiness) have
moved to the repo-root [CLAUDE.md](../../../CLAUDE.md), which agents load
automatically at session start. Read and follow them before executing any
skill. CLAUDE.md also carries a task-to-skill routing table for
disambiguating overlapping skill families.

## Available Skills

Grouped by lifecycle phase (mirrors the CLAUDE.md routing table).

### Design & analysis

- [arfc-create/SKILL.md](arfc-create/SKILL.md)
  Source: `../implementation/arfc.md`
- [arfc-task-create/SKILL.md](arfc-task-create/SKILL.md)
  Source: `../implementation/arfc-tasks.md`
- [use-case-backend-analyze/SKILL.md](use-case-backend-analyze/SKILL.md)
  Source: self-contained

### Implement — API, code & docs

- [api-implement/SKILL.md](api-implement/SKILL.md)
  Source: `../workflows/api-implementation.md`
- [api-endpoint-create/SKILL.md](api-endpoint-create/SKILL.md)
  Source: `../implementation/endpoint-creation.md`
- [api-endpoint-privileges-map-update/SKILL.md](api-endpoint-privileges-map-update/SKILL.md)
  Source: self-contained
- [db-table-create/SKILL.md](db-table-create/SKILL.md)
  Source: `../implementation/table-creation.md`
- [db-procedure-implement/SKILL.md](db-procedure-implement/SKILL.md)
  Source: `../implementation/stored-procedure.md`
- [db-migration-patterns/SKILL.md](db-migration-patterns/SKILL.md)
  Source: `../standards/migrations.md`
- [db-sql-conventions/SKILL.md](db-sql-conventions/SKILL.md)
  Source: `../standards/sql-conventions.md`
- [code-style-apply/SKILL.md](code-style-apply/SKILL.md)
  Source: `../standards/code-style.md`
- [code-error-handling-apply/SKILL.md](code-error-handling-apply/SKILL.md)
  Source: `../standards/error-handling.md`
- [code-copyright-header-apply/SKILL.md](code-copyright-header-apply/SKILL.md)
  Source: `../implementation/copyright-header.md`
- [docs-standards-apply/SKILL.md](docs-standards-apply/SKILL.md)
  Source: `../standards/documentation.md`
- [openapi-documentation-update/SKILL.md](openapi-documentation-update/SKILL.md)
  Source: `../implementation/openapi-documentation.md`
- [openapi-operation-id-apply/SKILL.md](openapi-operation-id-apply/SKILL.md)
  Source: `../standards/openapi-operation-ids.md`
- [deploy-cloudformation-template/SKILL.md](deploy-cloudformation-template/SKILL.md)
  Source: `../implementation/cloudformation-templates.md`

### Test & validate

- [api-check-scenario-create/SKILL.md](api-check-scenario-create/SKILL.md)
  Source: `../implementation/api-check-scenarios.md`
- [code-implementation-validate/SKILL.md](code-implementation-validate/SKILL.md)
  Source: `../validation/implementation-validation.md`
- [db-hypothesis-test/SKILL.md](db-hypothesis-test/SKILL.md)
  Source: self-contained

### Operate — run MCP tools & diagnose

- [api-check-run-mcp/SKILL.md](api-check-run-mcp/SKILL.md)
  Source: `../operations/api-check-via-mcp.md`
- [api-check-run-rca/SKILL.md](api-check-run-rca/SKILL.md)
  Source: `../operations/api-check-with-rca.md`
- [api-check-rca-cli-handoff/SKILL.md](api-check-rca-cli-handoff/SKILL.md)
  Source: `../operations/api-check-cli-handoff.md`
- [rca-determine/SKILL.md](rca-determine/SKILL.md)
  Source: `../validation/determine-rca.md`
- [rca-run-mcp/SKILL.md](rca-run-mcp/SKILL.md)
  Source: `../operations/rca-via-mcp.md`
- [rca-db-analyze/SKILL.md](rca-db-analyze/SKILL.md)
  Source: self-contained
- [openai-analyzer-run-mcp/SKILL.md](openai-analyzer-run-mcp/SKILL.md)
  Source: self-contained
- [aws-sso-cache-bootstrap/SKILL.md](aws-sso-cache-bootstrap/SKILL.md)
  Source: self-contained

### Deliver — commit → PR → review → merge

- [git-commit-create/SKILL.md](git-commit-create/SKILL.md)
  Source: `../delivery/git-commit.md`
- [git-commit-message-generate/SKILL.md](git-commit-message-generate/SKILL.md)
  Source: `../../carbon-based-how/delivery/commit-message.md`
- [pr-precheck-development/SKILL.md](pr-precheck-development/SKILL.md)
  Source: `../delivery/pr-precheck.md`
- [pr-create-development/SKILL.md](pr-create-development/SKILL.md)
  Source: `../delivery/pr-creation.md`
- [pr-create-production-promotion/SKILL.md](pr-create-production-promotion/SKILL.md)
  Source: `../delivery/pr-creation-production-promotion.md`
- [pr-review-development/SKILL.md](pr-review-development/SKILL.md)
  Source: `../delivery/pr-review-development.md`
- [pr-review-production/SKILL.md](pr-review-production/SKILL.md)
  Source: `../delivery/pr-review-production.md`
- [pr-test-results-upload/SKILL.md](pr-test-results-upload/SKILL.md)
  Source: `../delivery/upload-test-results.md`
- [deploy-development-squash-merge/SKILL.md](deploy-development-squash-merge/SKILL.md)
  Source: `../delivery/squash-merge-development.md`
- [deploy-development-rebase-merge/SKILL.md](deploy-development-rebase-merge/SKILL.md)
  Source: `../delivery/rebase-merge-development.md`
  Forked from `deploy-development-squash-merge` (issue #3666) for the
  `delivery/<datetime>` batch-PR → `release/development` second boundary
  only; F5 and the standalone single-PR case stay on the squash sibling.
- [deploy-production-fast-forward/SKILL.md](deploy-production-fast-forward/SKILL.md)
  Source: `../delivery/production-ff-promotion.md`
- [pr-cycle-development/SKILL.md](pr-cycle-development/SKILL.md)
  Source: self-contained
- [pr-cycle-development-bundle/SKILL.md](pr-cycle-development-bundle/SKILL.md)
  Source: self-contained
- [pr-cycle-production/SKILL.md](pr-cycle-production/SKILL.md)
  Source: self-contained
- [pr-create-feature-development/SKILL.md](pr-create-feature-development/SKILL.md)
  Source: `../delivery/pr-creation.md`
  **DEPRECATED** — merged into `pr-create-development`; stub redirects only.

### GitHub issue lifecycle

- [github-issue-create/SKILL.md](github-issue-create/SKILL.md)
  Source: self-contained
- [github-issue-status-update/SKILL.md](github-issue-status-update/SKILL.md)
  Source: `../delivery/issue-status.md`
- [github-issue-resolve/SKILL.md](github-issue-resolve/SKILL.md)
  Source: self-contained
- [github-issue-resolve-reuse-branch/SKILL.md](github-issue-resolve-reuse-branch/SKILL.md)
  Source: self-contained
- [github-issue-queue-resolve/SKILL.md](github-issue-queue-resolve/SKILL.md)
  Source: self-contained
- [dispatcher-enqueue-pending/SKILL.md](dispatcher-enqueue-pending/SKILL.md)
  Source: self-contained
- [dispatcher-classify-issue/SKILL.md](dispatcher-classify-issue/SKILL.md)
  Source: self-contained
- [cross-repo-issue-resolve/SKILL.md](cross-repo-issue-resolve/SKILL.md)
  Source: `../workflows/cross-repo-issue-resolution.md`
- [github-resolution-train/SKILL.md](github-resolution-train/SKILL.md)
  Source: self-contained
  **DEPRECATED** — superseded by `dispatcher-enqueue-pending`; stub redirects only.
- [github-sprint-api-check-group-resolve/SKILL.md](github-sprint-api-check-group-resolve/SKILL.md)
  Source: self-contained
- [github-issue-close-not-reproducible/SKILL.md](github-issue-close-not-reproducible/SKILL.md)
  Source: self-contained
- [github-sprint-close-not-reproducible/SKILL.md](github-sprint-close-not-reproducible/SKILL.md)
  Source: self-contained
- [github-project-query/SKILL.md](github-project-query/SKILL.md)
  Source: self-contained
- [backlog-issue-refine/SKILL.md](backlog-issue-refine/SKILL.md)
  Source: self-contained

### Report & maintain

- [daily-startup/SKILL.md](daily-startup/SKILL.md)
  Source: self-contained
- [aws-cost-report-monthly/SKILL.md](aws-cost-report-monthly/SKILL.md)
  Source: self-contained
- [session-notes-update/SKILL.md](session-notes-update/SKILL.md)
  Source: self-contained
- [session-notes-consolidate/SKILL.md](session-notes-consolidate/SKILL.md)
  Source: self-contained
- [session-todo-update/SKILL.md](session-todo-update/SKILL.md)
  Source: self-contained
- [agent-memory-consolidate/SKILL.md](agent-memory-consolidate/SKILL.md)
  Source: self-contained
- [worktree-sweep/SKILL.md](worktree-sweep/SKILL.md)
  Source: `../operations/worktree-lifecycle.md`
