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

# Silicon-Based How Index

This index covers the agent-oriented skills in the repository:
implementation guardrails, validation checklists, delivery workflows, and
MCP-first execution guidance.

## Start Here

- [README.md](README.md)
- [skills/README.md](skills/README.md)
- [operations/mcp-tools.md](operations/mcp-tools.md)
- [skills/api-check-run-mcp/SKILL.md](skills/api-check-run-mcp/SKILL.md)
- [skills/api-check-rca-cli-handoff/SKILL.md](skills/api-check-rca-cli-handoff/SKILL.md)
- [skills/api-check-run-rca/SKILL.md](skills/api-check-run-rca/SKILL.md)
- [skills/rca-run-mcp/SKILL.md](skills/rca-run-mcp/SKILL.md)
- [skills/rca-db-analyze/SKILL.md](skills/rca-db-analyze/SKILL.md)
- [skills/db-hypothesis-test/SKILL.md](skills/db-hypothesis-test/SKILL.md)
- [skills/code-implementation-validate/SKILL.md](skills/code-implementation-validate/SKILL.md)

## Skill Groups

- Delivery:
  [skills/git-commit-create/SKILL.md](skills/git-commit-create/SKILL.md),
  [skills/git-commit-message-generate/SKILL.md](skills/git-commit-message-generate/SKILL.md),
  [skills/pr-cycle-development-bundle/SKILL.md](skills/pr-cycle-development-bundle/SKILL.md),
  [skills/pr-cycle-development/SKILL.md](skills/pr-cycle-development/SKILL.md),
  [skills/pr-precheck-development/SKILL.md](skills/pr-precheck-development/SKILL.md),
  [skills/pr-create-development/SKILL.md](skills/pr-create-development/SKILL.md),
  [skills/pr-create-production-promotion/SKILL.md](skills/pr-create-production-promotion/SKILL.md),
  [skills/pr-review-development/SKILL.md](skills/pr-review-development/SKILL.md),
  [skills/pr-review-production/SKILL.md](skills/pr-review-production/SKILL.md),
  [skills/pr-cycle-production/SKILL.md](skills/pr-cycle-production/SKILL.md),
  [skills/deploy-development-squash-merge/SKILL.md](skills/deploy-development-squash-merge/SKILL.md),
  [skills/deploy-production-fast-forward/SKILL.md](skills/deploy-production-fast-forward/SKILL.md)
- Implementation:
  [implementation/merlin-domain-facades.md](implementation/merlin-domain-facades.md),
  [skills/api-endpoint-create/SKILL.md](skills/api-endpoint-create/SKILL.md),
  [skills/db-procedure-implement/SKILL.md](skills/db-procedure-implement/SKILL.md),
  [skills/db-table-create/SKILL.md](skills/db-table-create/SKILL.md),
  [skills/openapi-documentation-update/SKILL.md](skills/openapi-documentation-update/SKILL.md),
  [skills/api-check-scenario-create/SKILL.md](skills/api-check-scenario-create/SKILL.md)
- Validation:
  [skills/api-check-rca-cli-handoff/SKILL.md](skills/api-check-rca-cli-handoff/SKILL.md),
  [skills/api-check-run-rca/SKILL.md](skills/api-check-run-rca/SKILL.md),
  [skills/code-implementation-validate/SKILL.md](skills/code-implementation-validate/SKILL.md),
  [skills/rca-run-mcp/SKILL.md](skills/rca-run-mcp/SKILL.md),
  [skills/rca-db-analyze/SKILL.md](skills/rca-db-analyze/SKILL.md),
  [skills/db-hypothesis-test/SKILL.md](skills/db-hypothesis-test/SKILL.md),
  [skills/rca-determine/SKILL.md](skills/rca-determine/SKILL.md)
- Standards:
  [skills/code-style-apply/SKILL.md](skills/code-style-apply/SKILL.md),
  [skills/code-error-handling-apply/SKILL.md](skills/code-error-handling-apply/SKILL.md),
  [skills/db-migration-patterns/SKILL.md](skills/db-migration-patterns/SKILL.md),
  [skills/db-sql-conventions/SKILL.md](skills/db-sql-conventions/SKILL.md)
- Workflows:
  [skills/api-implement/SKILL.md](skills/api-implement/SKILL.md)
- Session:
  [skills/session-notes-consolidate/SKILL.md](skills/session-notes-consolidate/SKILL.md),
  [skills/session-notes-update/SKILL.md](skills/session-notes-update/SKILL.md),
  [skills/session-todo-update/SKILL.md](skills/session-todo-update/SKILL.md)

## Next Stops

- System context: [../what-and-why/index.md](../what-and-why/index.md)
- Human fallback procedures: [../carbon-based-how/README.md](../carbon-based-how/README.md)

## Reading Strategy

1. Read `what-and-why` first for system context.
2. Start here with `README.md`, then pick the relevant skill from `skills/`.
3. Use [operations/mcp-tools.md](operations/mcp-tools.md) when you first need
   to know which repository MCP server to start and which command to use.
4. Read the selected skill's `SKILL.md`, then open its `references/guide.md`
   only as needed.
5. Prefer `skills/api-check-run-mcp/SKILL.md` for `api-check` execution.
6. Prefer `skills/api-check-rca-cli-handoff/SKILL.md` when the user supplies a
   human-run `api-check` command and terminal excerpt.
7. Prefer `skills/api-check-run-rca/SKILL.md` for validation workflows that
   must continue into RCA on failure.
8. Prefer `skills/rca-run-mcp/SKILL.md` for direct `rca` execution.
9. Prefer `skills/rca-db-analyze/SKILL.md` when RCA needs live database
   inspection through read-only tables or views.
10. Prefer `skills/db-hypothesis-test/SKILL.md` when the next step
   is validating candidate database fixes in scratch schemas before Flyway.
11. Fall back to `carbon-based-how` only when no silicon-specific procedure exists.
