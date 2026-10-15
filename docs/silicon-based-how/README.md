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

# Silicon-Based How

This section is for AI agents and other automation acting on the repository.
Its task guidance now lives primarily in formal skills under
[skills/README.md](skills/README.md).

Read [../what-and-why/index.md](../what-and-why/index.md) first for system
context. Then use this directory for task execution.

If no silicon-specific procedure exists for a task, fall back to the matching
guidance in [../carbon-based-how/README.md](../carbon-based-how/README.md)
instead of inventing a workflow.

## Start Here

- [skills/README.md](skills/README.md)
- [operations/mcp-tools.md](operations/mcp-tools.md)
- [skills/api-check-run-mcp/SKILL.md](skills/api-check-run-mcp/SKILL.md)
- [skills/api-check-rca-cli-handoff/SKILL.md](skills/api-check-rca-cli-handoff/SKILL.md)
- [skills/api-check-run-rca/SKILL.md](skills/api-check-run-rca/SKILL.md)
- [skills/rca-run-mcp/SKILL.md](skills/rca-run-mcp/SKILL.md)
- [skills/rca-db-analyze/SKILL.md](skills/rca-db-analyze/SKILL.md)
- [skills/db-hypothesis-test/SKILL.md](skills/db-hypothesis-test/SKILL.md)
- [skills/code-implementation-validate/SKILL.md](skills/code-implementation-validate/SKILL.md)
- [skills/api-endpoint-create/SKILL.md](skills/api-endpoint-create/SKILL.md)

## Architecture Guides

- [implementation/merlin-domain-facades.md](implementation/merlin-domain-facades.md)

## Skill Catalog

- [skills/README.md](skills/README.md)
- [skills/pr-precheck-development/SKILL.md](skills/pr-precheck-development/SKILL.md)
- [skills/pr-create-development/SKILL.md](skills/pr-create-development/SKILL.md)
- [skills/git-commit-message-generate/SKILL.md](skills/git-commit-message-generate/SKILL.md)
- [skills/git-commit-create/SKILL.md](skills/git-commit-create/SKILL.md)
- [skills/deploy-development-squash-merge/SKILL.md](skills/deploy-development-squash-merge/SKILL.md)
- [skills/api-implement/SKILL.md](skills/api-implement/SKILL.md)
- [skills/api-check-rca-cli-handoff/SKILL.md](skills/api-check-rca-cli-handoff/SKILL.md)
- [skills/api-check-run-rca/SKILL.md](skills/api-check-run-rca/SKILL.md)
- [skills/rca-run-mcp/SKILL.md](skills/rca-run-mcp/SKILL.md)
- [skills/rca-db-analyze/SKILL.md](skills/rca-db-analyze/SKILL.md)
- [skills/db-hypothesis-test/SKILL.md](skills/db-hypothesis-test/SKILL.md)

## Session State

- [skills/session-notes-consolidate/SKILL.md](skills/session-notes-consolidate/SKILL.md)
- [skills/session-todo-update/SKILL.md](skills/session-todo-update/SKILL.md)
- [skills/session-notes-update/SKILL.md](skills/session-notes-update/SKILL.md)

## Source Guides

- The prior markdown guides remain in `delivery/`, `implementation/`,
  `operations/`, `session/`, `standards/`, `validation/`, and `workflows/`.
- The repository-wide MCP inventory and canonical startup commands are in
  [operations/mcp-tools.md](operations/mcp-tools.md).
- The accepted Merlin domain-facade/runtime package model is documented in
  [implementation/merlin-domain-facades.md](implementation/merlin-domain-facades.md).
- Each skill in `skills/` links back to one of those source guides through
  `references/guide.md`.

## Guardrails

- Use [operations/mcp-tools.md](operations/mcp-tools.md) when you need the
  full list of MCP-capable repository tools and their startup commands.
- Prefer MCP over CLI when executing `api-check` from an agent.
- Prefer [skills/api-check-run-rca/SKILL.md](skills/api-check-run-rca/SKILL.md)
  when the task is endpoint validation plus RCA on failure.
- Prefer [skills/api-check-rca-cli-handoff/SKILL.md](skills/api-check-rca-cli-handoff/SKILL.md)
  when a human already ran `api-check` and pasted the CLI command plus failure
  excerpt.
- Prefer MCP over CLI when executing `rca` from an agent.
- Prefer [skills/rca-db-analyze/SKILL.md](skills/rca-db-analyze/SKILL.md)
  when RCA needs read-only database inspection through `db-analyzer`.
- Prefer [skills/db-hypothesis-test/SKILL.md](skills/db-hypothesis-test/SKILL.md)
  when the investigation needs scratch-schema database experiments through
  `db-hypothesizer`.
- Prefer [skills/git-commit-message-generate/SKILL.md](skills/git-commit-message-generate/SKILL.md)
  when asked to draft a git commit message.
- Prefer [skills/git-commit-create/SKILL.md](skills/git-commit-create/SKILL.md)
  when asked to stage and create a commit.
- Prefer the formal skills in `skills/` over browsing the source guides
  directly.
- If no silicon-specific procedure exists, use the corresponding carbon-based
  procedure as the fallback.
- Use the human CLI guides in `carbon-based-how` when MCP is unavailable or
  when the task is explicitly about human/operator usage.
