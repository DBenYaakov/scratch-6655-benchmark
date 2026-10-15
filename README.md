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
# shared-tools

Tooling shared across `ayode-institute` repositories (e.g.
`codermerlin.academy-backend`, `codermerlin.academy-frontend`). Consumer
repositories mount this repo as a git submodule at `./shared-tools`.

## Contents

- `agent-coordinator/` — Go MCP HTTP tool for coordinating external AI
  agents during repository PR-review workflows. See its own README.
- `shared/mcpargs/` — small shared package for strict MCP tool-call argument
  decoding, used by `agent-coordinator`.
- `common-make.sh` — shared bash build helpers used by `agent-coordinator`'s
  `make.sh`.

## Adding a new shared tool

Follow the same layout: each tool gets its own top-level directory with its
own `go.mod`/`make.sh`/`run.sh`, following the pattern established by
`agent-coordinator/`. Any dependency shared across multiple tools in this
repo belongs under `shared/`, referenced via relative-path `replace`
directives, the same way `agent-coordinator/go.mod` references
`../shared/mcpargs`.
