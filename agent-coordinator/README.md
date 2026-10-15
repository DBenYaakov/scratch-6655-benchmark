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
# agent-coordinator

`agent-coordinator` is a Go MCP HTTP tool for coordinating external AI agents during repository workflows.

## Supported workflows

- `run_pr_review`: hand a diff, metadata bundle, context files, and validation logs to an external agent for review.
- `run_cross_check`: hand deployment artifacts, logs, and instructions to an external agent for a structured cross-check.

## Agents

- `claude`: live adapter that invokes the local `claude` CLI in non-interactive print mode.
- `opencode`: live OpenCode V2 adapter using a coordinator-owned private API runtime and fresh read-only reviewer sessions. It never invokes Claude or uses the operator's service for reviewer execution.
- `fixture`: deterministic adapter for local testing and offline validation.

## Build

This tool lives in the `ayode-institute/shared-tools` repository, consumed by
downstream repositories (e.g. `codermerlin.academy-backend`) as a git
submodule mounted at `./shared-tools`. It depends on the sibling
`../shared/mcpargs` package and the repo-root `../common-make.sh` helper via
relative paths, so build it from within a full checkout of this repository
(standalone or as an initialized submodule) — not by copying this directory
out on its own:

```bash
cd shared-tools/agent-coordinator   # or agent-coordinator/, if this is a standalone checkout of shared-tools itself
./make.sh
```

## Direct CLI

```bash
./run.sh list-agents
./run.sh run-pr-review --agent fixture --title "Fixture review" --diff-file /tmp/change.diff
./run.sh run-cross-check --objective "Cross-check deployment logs" --artifact-file /tmp/deploy.log
```

## MCP HTTP mode

```bash
./run.sh --mcp --mode api-key --listen 127.0.0.1:8082
```

The MCP endpoint is `POST /mcp` and follows the same transport contract used by `tools/api-check` and `tools/rca`.

The HTTP server sets `ReadTimeout`, `ReadHeaderTimeout`, and `IdleTimeout`. It intentionally does not set `WriteTimeout`, because live external-agent tool calls can run for multiple minutes and must be allowed to stream until completion.

## Artifacts

Each run writes artifacts under `.artifacts/agent-coordinator` by default:

- `request.json`
- `prompt.txt`
- `raw-response.json`
- `stdout.log`
- `stderr.log`
- `result.json`
- `usage.json` (issue #5275 — see "Usage and cost capture" below)
- `shards.json` (issue #6655 — sharded reviews only; see "Sharded review of large declared inputs")
- `shards.json` sidecar `shards.json.readable.txt` when (and only when) the authoritative bytes contain an over-long line

Before these artifacts are written, the coordinator redacts exact secret values sourced from sensitive environment variables such as `CLAUDE_CODE_OAUTH_TOKEN` and `ANTHROPIC_API_KEY`, and it redacts common token-bearing fields such as `Authorization`, `x-api-key`, `token`, `password`, and related assignments in prompt/transcript content. (redaction.go's regex-based defensive redaction of `anthropic_auth_token`-shaped assignments in free-form text is untouched by the removal of `auth-token` as a mode below — it is a values-in-text safety net, not a mode-enum restatement.)

Override the root with `--artifact-root` or `AGENT_COORDINATOR_ARTIFACT_ROOT`.

**Natively readable generated artifacts (issue #6655, COORDINATOR-ARTIFACT-RENDER/1).** The native read tool renders at most 2000 UTF-16 code units per physical line, so a generated artifact containing one over-long physical line could never be fully read and accepted. The coordinator therefore splits over-long lines losslessly:

- **Category A — rendered in place.** The ref-generated diff (`inputs/git-diff.patch`/`inputs/diff-inline.patch`) and the generated context JSON (`request.json`) are rendered in place when any line exceeds the bound: every over-long line becomes one marker line `[coordinator-line-split v<N>: sha256=<16-hex> utf16=<U> segments=<N>]` (N = the SELECTED framing generation) followed by N segment lines of at most 1000 UTF-16 units that concatenate with no separator back to the original line exactly, under an in-band header `[coordinator-artifact-render v1: generation=<N> … original sha256/bytes/lines …]` that binds the selected generation explicitly. The original bytes remain authoritative and SHA-bound; reconstruction is `decode(render(x)) == x`, verified before the renderer writes, and the decoder decodes ONLY the header-bound generation's grammar: every other generation's marker-shaped line — a fully valid foreign-generation marker with its matching payload included — is preserved verbatim as data, never validated and never consumed (it can neither establish framing nor force the
  collision refusal). `request.json` is redacted FIRST and rendered second; below the bound the renderer is a byte-for-byte no-op. When the diff is rendered, the generated context JSON's diff payload carries a machine-visible `render` object (`kind`, `originalSHA256`, `originalBytes`, `splitLines`). A content line that mimics the marker grammar escalates through the fixed v1→v8 marker-generation chain; if a line mimics every generation the render refuses with `generated artifact line-split marker collision; refusing to render`. When rendering occurs, the on-disk `request.json` is no longer directly parseable JSON (its physical lines were split): no adapter code machine-parses the rendered form, and reconstruction restores the original bytes exactly.
- **Category B — authoritative bytes unchanged + SHA-bound readable views.** `prompt.txt`, `result.json`, `raw-response.json`, `usage.json`, `shards.json`, `stdout.log` and `stderr.log` are NEVER transformed. Where a reviewer may need to read them, the coordinator additionally writes a sidecar `<name>.readable.txt` — only when an over-long line exists — whose header `[coordinator-artifact-view v1: source=… original sha256/bytes/lines …; non-authoritative readable view; the source bytes are unchanged and remain authoritative]` binds the source name and digest. Views are explicitly non-authoritative: machine consumers keep reading the unchanged originals. The `prompt.txt` view is generated/refreshed only AFTER the authoritative admitted-prompt write (`ocPersistAdmissionPrompt`), from those exact final bytes, so its source digest always equals `promptSHA256` and it can never describe the pre-finalize core prompt; shard prompts pass through the same write site. The `shards.json` view is generated only after the sharded path's final aggregate snapshot, so its source digest always equals the on-disk `shards.json`.
- Acceptance is unchanged: it validates the on-disk rendered bytes (`ocManifest` line/byte/SHA counts and the native source/render comparison), the marker never matches the native continuation grammar `[Output truncated. Continue reading with offset: N]`, and no input-dropping, word-diff substitution or manual approval exists.

### Usage and cost capture (`usage.json`)

Every `run_pr_review`/`run_cross_check` execution writes `<artifactDir>/usage.json` alongside `stdout.log`, additively — `stdout.log` itself is unchanged, so anything that already re-parses it (e.g. the backend repo's `tools/cost-ledger`) keeps working byte-for-byte. `usage.json` is a provider-neutral envelope (`schemaVersion`, `provider`, `run`, `cost`, `tokens`, `models`, `timing`, `subagents`, `terminal`, `capture`, `raw`): every field outside `raw` is shaped so a future non-Claude adapter (e.g. codex) can populate the identical structure, and `raw.result` holds the provider's verbatim terminal payload for anyone who needs a provider-specific field. One additive field exists on sharded runs only (see below): `sharded`, absent (`null`) on every single-pass run.

**Sharded usage capture (issue #6655 security-review rounds 2 and 4).** A sharded call captures usage for EVERY native session — each shard/slice attempt, each replacement attempt, and every failed or timed-out attempt whose native run created a fresh session — EXACTLY ONCE: each session writes its own `<artifactDir>/shards/<unitId>/usage.json` through the same `WriteUsageArtifact`/`ocNativeUsage` machinery and the same schema as the single-pass path, tagged with `run.requestId` = `<requestId>/shards/<unitId>` (a replacement carries its own id, e.g. `S01r1`, so the tag names the attempt). Session start is the runner's EXPLICIT fresh-native-session fact, set exactly at session creation (the runner's execution object exists from the moment owned setup begins, which is earlier), never derived from execution allocation. The call additionally writes the top-level `<artifactDir>/usage.json` whose `cost` and `tokens` are the SUM across all recorded session attempts (failed and replaced attempts included, never double-counted; a partially reporting attempt set is marked in
`capture.status`/`capture.missingFields`), with `models[]` merged by provider/model/canonical identity. The top-level envelope gains the additive `sharded` field: `sharded.attempts` is the recorded native-session count and `sharded.records[]` lists one entry per session — `unitId`, `lineageId`, `attempt` (1 = parent, 2/3 = replacements), `accepted`, the relative `usagePath` of its per-attempt `usage.json`, and that attempt's own `costUSD`/`tokens`/`captureStatus` — so the per-attempt breakdown is discoverable from the aggregate file itself; the recorded sessions always equal the actual native-session creations. A pre-session owned-setup attempt that created NO native session (owned server spawn, initial process ownership verification, readiness/settlement, admission, ...) is disclosed as its OWN category: `sharded.setupAttempts` counts it and `sharded.setupRecords[]` carries `unitId`/`lineageId`/`attempt` plus the fixed `category` `"owned-setup"` — it never produces a per-session `usage.json`, never contributes cost/tokens and is never counted as a session. The aggregate is written
on every exit path where at least one native session started OR at least one pre-session owned-setup attempt was recorded (success, refusal or failure); the documented pre-start refusal (context expired before owned setup began) records nothing, exactly like the single-pass path. Usage capture remains optional accounting: a per-attempt or aggregate write failure is a stderr warning only and can never fail or gate a review.

A second, otherwise fully invisible `claude` run also gets its own usage record: the docs-readiness preflight (`ensureClaudeDocsReadiness`) spawns a complete `claude` subprocess whose cost was previously discarded on every path. When it actually runs (not on a 5-minute TTL cache hit), its usage.json and stdout.log are written to `<repoPath>/.artifacts/agent-coordinator/<timestamp>Z-docs_readiness-<id>/` — a destination derived from the repository path itself (the same value the run is de-duplicated on via `singleflight`), not from any caller's own `--artifact-root`, so concurrent callers that share one preflight run necessarily agree on where its record lands. That run is tagged `function: "docs_readiness"` and `attribution: {scope: "repo_preflight", exactness: "shared_unapportioned"}`: it is a repo-scoped shared overhead cost attributed to no individual review (attributing it to the review that happened to trigger it would overstate that review and understate the others; dividing it across concurrent waiters is not implementable today, since `singleflight.Group.Do` reports only whether a run was shared, not how many callers shared it — recorded as `sharedRun: true|false`).

**Cost.** `cost` is nullable as a whole (`null`, never a zeroed object) whenever the underlying stream carries no cost data at all — zero dollars is a claim the stream did not make. For Claude, `cost.totalCostUSD` is Claude CLI's own `total_cost_usd`, and `cost.costBasis` is **derived**, not read off a top-level key — the CLI's `result` event carries no top-level `costBasis` field at all; the basis lives only on each `modelUsage` entry. The derivation (per directive 810):

- every `models[]` entry present and agreeing on a non-empty basis → that value (typically `"list"`)
- entries present with two or more distinct non-empty values → `"mixed"`
- any entry missing its basis, or no model entries at all → `"unknown"` (measured on a live host-wide corpus re-sweep, 2026-09-17: roughly 1 run in 3 to 1 in 2, depending on which CLI builds are represented — not a rare edge case)

`cost.isInvoice` is always `false`. Claude's `cost.note` reads: *"List-price computation, not an invoice. Good for comparing functions; approximate as absolute spend."* OpenCode accounting has its own native-source caveat and does not infer Claude's price basis or cache-write duration. Wherever a cost figure surfaces, the source-specific caveat travels with it.

**Models.** `models` is always an array (never a map keyed by model id, and never `null` — an empty run still marshals `models: []`). Claude's own `modelUsage` happens to be keyed by model name, but a provider stream with no model name at all (e.g. codex) cannot key a map, so every entry carries its own nullable `model` field instead.

**Tags.** `run.function` is one of `run_pr_review` / `run_cross_check` / `docs_readiness`, taken from the call site rather than guessed from content (`run.functionBasis: "coordinator_call_site"`, always). `run.prNumber` and `run.issueNumber` tag the run for downstream cost attribution: `prNumber` mirrors the existing `prNumber` MCP argument used for GitHub delivery confirmation (see below), and `issueNumber` is an independent optional MCP argument on both `run_pr_review` and `run_cross_check` for attributing spend to a GitHub issue. Both are `null` (not merely absent) when not supplied.

**Capture never gates a review.** `capture.status` is one of:

- `"ok"` — the terminal `result` event decoded cleanly (individual fields may still be individually absent; see `capture.missingFields`).
- `"no_result_event"` — the run was killed, timed out, or otherwise ended before a terminal `result` event was ever emitted. The run is recorded as unmeasurable rather than as a run that never happened; this is expected operational attrition and should never alert.
- `"stream_unreadable"` — not one line of the captured stream decoded as JSON at all.
- `"result_event_unparseable"` — a terminal `result` event WAS present but could not be decoded into the expected shape (`capture.parseError` and `capture.unknownTopLevelKeys` are populated). This is CLI schema drift in the extractor's own contract and is the one status that should alert.

A write failure for `usage.json` itself (disk full, read-only artifact directory, …) is a logged stderr warning only and is deliberately never joined into the run's own error — unlike `stdout.log`/`stderr.log`/`raw-response.json`, whose write errors ARE surfaced. Raw transcript artifacts are evidence a verdict depends on; a cost record is not, so a usage-capture failure can never fail or gate a review.

## OpenCode V2 configuration

The OpenCode adapter is separate from Claude's authentication, settings,
transcript parser, and readiness cache. The default agent remains `claude` for
backwards compatibility. Select `--agent opencode` for an OpenCode-only launch;
no `--mode` or Claude credential is required on that path. A Claude-default
server retains its historical eager Claude startup preflight, even if a later
call selects OpenCode. Per-call backend selection does not undo a startup that
already happened.

### Model and startup settings

The supported native contract is OpenCode V2 **2.0.26**, using the running
server's `/openapi.json`, not V1 routes or the V1 configuration schema.

OpenCode requires an explicit OpenRouter **model** and an explicit **effort**
with **no default of any kind**: there is no built-in model, no built-in
effort, and no blank-input fallback. The provider is fixed to `openrouter`;
any other provider selector is refused. The raw environment selection is
validated independently of the flags: an explicitly supplied invalid
`AGENT_COORDINATOR_OPENCODE_MODEL` / `AGENT_COORDINATOR_OPENCODE_VARIANT`
value is refused even when `--opencode-model`/`--opencode-variant` or a
per-call override supplies the effective selection. Both values are validated
against live availability — never a compiled catalog or a compiled effort
enum:

- **Stage 1 (model):** the model must resolve in the live OpenRouter catalog
  (`GET https://openrouter.ai/api/v1/model/{author}/{slug}`, public, no auth,
  alias-resolving). Unknown, deprecated or unavailable models are refused,
  naming the requested model.
- **Stage 2 (effort):** the effort must appear in that model's
  `reasoning.supported_efforts` (checked verbatim, case-sensitive, with no
  normalization and no silent substitution). A model that advertises no
  selectable effort is refused for reviewer use, because an explicit effort
  cannot be satisfied.
- **Runtime catalog (post-start, pre-dispatch):** the private runtime's
  effective catalog (`GET /api/model`) must additionally expose the selected
  model with exactly one variant whose settings materialize
  `{"reasoning":{"effort":"<effort>"}}`. This check runs after the private
  runtime and credential setup for that execution have already started, but
  strictly **before any model dispatch**: an empty or absent inventory retries
  only within the bounded settlement window; a conclusive inventory lacking
  the model, or lacking/mismatching the variant, refuses immediately with a
  message naming the model (or the variant and model) and no model request is
  admitted.

Required-value, provider and public-availability refusals (stages 1-2 above)
happen before any private runtime starts: a refused request spawns no
runtime, reaches no credential broker and spends nothing. The runtime
effective-catalog refusals are different by design — they are evaluated
post-start but pre-dispatch, so that execution's private runtime and
credential setup have already run, yet no model request is admitted and
nothing is spent on the selected model. Messages name the offending value
(quoted in full) and, where known, list the supported efforts; they never
include credentials or raw provider error bodies. Availability changes over
time, so the catalog query is deliberately uncached and performed per
execution. Other provider/model/variant profiles, custom routing, auth
headers and endpoints remain refused rather than silently substituted. The
shared selector parser validates native reference syntax, while execution
validates the live catalogs above. This does not narrow Claude's existing
alias/model/effort choices.

| Startup flag | Environment variable | Meaning |
| --- | --- | --- |
| `--opencode-bin` | `AGENT_COORDINATOR_OPENCODE_BIN` | OpenCode executable; default `opencode` |
| `--opencode-model` | `AGENT_COORDINATOR_OPENCODE_MODEL` | Required `provider/model` or `provider/model#variant`; provider must be `openrouter`; validated against the live OpenRouter catalog |
| `--opencode-variant` | `AGENT_COORDINATOR_OPENCODE_VARIANT` | Required effort unless embedded as `#variant`; validated against the model's `supported_efforts` |
| `--opencode-max-duration` | `AGENT_COORDINATOR_OPENCODE_MAX_DURATION` | Positive finite wall-clock bound; default `30m`; zero, negative and malformed values fail |
| `--opencode-source-registration` | `AGENT_COORDINATOR_OPENCODE_SOURCE_REGISTRATION` | Trusted existing source-service registration, not a task-supplied URL or credential |
| `--opencode-policy-scope` | `AGENT_COORDINATOR_OPENCODE_POLICY_SCOPE` | Trusted launcher declaration of supported restriction coverage; not a permission bypass |
| `--docs-entrypoint` (repeatable) | — | Explicit nonempty startup-only readiness targets for the canonical startup repository |

The OpenCode lifecycle allowance begins before repository normalization, head
resolution, diff generation and input persistence. Readiness and native review
share the remaining budget; starting the adapter does not reset it. Repository
Git subprocesses honor cancellation and the earlier of the parent deadline or
the configured allowance. Synchronous filesystem work is charged to this budget,
with expiry checks before later stages; it is not claimed to be preemptible.

OpenCode source Git setup disables textconv, external diff and repository fsmonitor.
On Linux, each setup spawn owns a separate process group, retains its leader's
start identity until group cleanup, and bounds inherited-pipe draining to 500 ms.
OpenCode refuses unsupported platforms before setup or credential/runtime work.
Non-Linux Claude/fixture workflows retain an explicitly separate legacy Git path,
not an OpenCode fallback or a claim of Linux containment equivalence. This is not
containment for arbitrary code that changes process groups or transfers
descriptors to unrelated processes, and it does not authorize repository-authored
helpers.

The private native profile has a source-defined finite cap of **192 assistant
steps**. Acceptance remains strictly below that cap: a forced final at step 192
is incomplete even if native execution reports success and returns valid JSON.
This cap supports serial pagination without depending on model-selected parallel
calls. It allows more potential model requests and spend than the earlier
64-step profile, within the same configured lifecycle deadline; it does not reset
or extend that deadline. Tool diagnostics remain capped at 256 and all existing
transport, message-recovery, permission, credential and evidence bounds remain.

Both MCP tools accept `agent: "opencode"`, the existing `model` string, and
`variant`. There is no default at the call layer either: the effective
selection must be explicit from the startup configuration and/or the call
(model **and** effort), or the call fails closed with the required-value
refusal. Split the selector at its **first** slash: the model ID is
`deepseek/deepseek-v4.1-flash`, not `deepseek-v4.1-flash`. An embedded variant
and a separately supplied variant must agree when both are explicit at the
same override layer; a call model without an embedded effort inherits the
startup effort when one exists. Per-call selection overrides the selected
backend's startup settings, never Claude's model/effort. OpenCode rejects
nonempty `effort`; Claude/fixture reject `variant` rather than silently
pretending to use it. Direct review/cross-check subcommands also accept
`--model`, `--variant`, and backend-appropriate `--effort` overrides. Every
OpenCode selection is live-validated before any runtime starts, and docs
readiness runs the same explicit, validated selection — there is no hidden
second model pin.

### Credential-only inheritance and trusted source access

The provider process may use a genuinely inherited selected-provider
environment credential. Otherwise a trusted machine-local broker reads the
**already-running** local OpenCode service's native credential API and imports
only the intended active native OpenRouter key into fresh private reviewer
state. Exactly one active `integrationID: "openrouter"` key must be present;
missing, ambiguous, malformed or unsupported selected credentials fail closed.
The broker is internal, not a credential MCP tool or a per-call option.

The source registration must be trusted, private and owned by the current user,
and its literal loopback endpoint, live process identity and native version must
agree. The broker never calls service discovery, autostart, restart, reload,
login, account switching, source model/session execution, or source DB reads or
writes. A stale/unavailable/untrusted source is a blocker, not authorization to
repair it. Whole-database copies, even temporary bootstrap snapshots, are
prohibited: conversations, sessions, saved approvals and unrelated credentials
must not be copied. Private refreshes never propagate back to the source.

**Never use `GET /api/credential` as a diagnostic command.** That native endpoint
returns secret values, and its import reply echoes a secret too. Both belong
only in bounded volatile broker memory, never shell output, logs, prompts,
process arguments, artifacts, model context or a shared cache. Only the selected
key is passed to the private native import; other values are discarded without
being persisted or used. No new provider key or pasted secret is required by
this workflow. Errors contain safe stage diagnostics, not native response bodies.

### Read-only permissions and restriction coverage

Each execution owns private 0700 HOME/XDG/config/data/state/log/temp directories,
a fresh writable database, a private authenticated loopback server and a fresh
root session. It is not a fork or continuation of the implementer, operator or
another reviewer. The child receives no GitHub/Claude/other-provider credentials
or original plugin/agent/MCP directories. Project discovery is disabled through
the installed native V2 control; it must not be replaced by loading arbitrary
reviewed-repository plugins and then adding an allowlist.

Native process teardown retains the leader unreaped through every group signal,
rechecks its UID/start identity/group, then performs one final reap and joins
the owned observer/readers. Exit notification is not permission to signal a
recycled group. A late teardown ownership or reap failure invalidates verdict
acceptance; it is not silently discarded after an otherwise successful review.
Private-state removal is also an acceptance veto: only the captured created
directory may be removed. On failure, its path/identity remain in memory and
repeated closes retain a fixed safe error without retrying or claiming success.
Private paths, credentials and OS deletion-error bodies are not published.

Launch the coordinator with a process-local restrictive `umask 077`. Native
OpenCode creates additional database/cache/log directories and files under that
mask; a group-writable inherited mask correctly fails the generated-path
ownership guard before credential acquisition. Do not chmod operator directories
or weaken the guard to repair that refusal. The mask belongs to this launch,
not a global shell/service configuration change.

The reviewer starts deny-all with narrowly scoped native `read` grants. Shell,
edit, subagent, browser, web, MCP, Code Mode `execute`, and arbitrary session or
worktree utilities are not review capabilities. External artifact access grants
the required canonical files, not an entire home/temp/artifact directory.
Private/original credential, config, GitHub, SSH and process-secret paths remain
protected, including aliases/symlinks. This is a tool-level isolation boundary,
**not an OS sandbox against arbitrary same-user processes or plugin code**.

Original `GH_CONFIG_DIR` and `CLAUDE_CONFIG_DIR` directories, their default
locations, and canonical aliases are protected before required-input reads or
grants. Missing configured roots retain a reserved namespace; ambiguous root
resolution fails closed rather than silently dropping protection. Removing a
credential variable from the native child is not a substitute for protecting
the original credential files. Known GH `hosts.yml` and Claude
`.credentials.json` children are also resolved, including outward file symlinks
and regular-file identity aliases. These checks use metadata, not credential
contents or directory enumeration; they do not promise an immutable filesystem
transaction against arbitrary same-user changes.

Native wildcard matching normalizes backslashes in both grants and incoming
resources, while Linux filenames retain them literally. Required input spellings
containing a backslash are refused. Before any authored-config or input-content
read or grant, metadata-only checks also refuse every already-existing
slash/backslash reverse spelling of each canonical target, including directory,
symlink and hard-link aliases. The complete deduplicated set is checked first;
failed or ambiguous metadata never produces a partial policy. The generated
private-tree ownership walk likewise refuses backslash-containing names.

This preflight supports at most 12 nonleading separators and 4,096 bytes per
target, 65,536 derived spellings per input set, and 1,048,576 prefix metadata
calls per validation phase. Excess depth/work or uncertain traversal fails
closed rather than sampling aliases. Original target paths, digests and actual
native read receipts remain unchanged; no source content is staged as a substitute.
The check closes static alias conflicts under the existing trusted/stable-source
boundary, not arbitrary concurrent same-user filesystem mutation.

Disabling project discovery does not authorize dropping project/operator denies.
Actual native OpenCode restrictions and applicable explicit structured tool/file
denials and asks are collected as bounded data without executing plugins,
substitutions, legacy loaders or auth hooks, then intersected with the coordinator
scope. Asks are not auto-approved. Unknown/unsupported actual native policy or
ambiguous structured restrictions fail before prompt admission; changed applicable
sources invalidate the run.

Supported restrictive `**/` file patterns include their zero-directory cases
through bounded conservative expansion: at most five optional globstars, 32
variants per origin, and the existing 1,024-rule limits. Unsupported syntax or
overflow is refused before input contents are read. Native single-star matching
can overdeny across directories; this restrictive-only projection never creates
additional grants or claims universal Claude-pattern equivalence.

Configuration substitution expressions are refused in decoded strings and keys,
including JSON-escaped spellings and nested provider/model data. The exact final
private configuration bytes are checked again before executable lookup or
runtime startup; native `{file:...}` / `{env:...}` expansion is not a read grant.

**Runtime policy limitation:** this adapter uses native OpenCode policy, not
Claude Code's natural-language `autoMode` classifier. It does not emulate or
migrate Claude-specific soft-deny prose, inherited prose defaults, environment
descriptions, or enabled Claude plugin behavior. Those excluded values are not
loaded/executed or sent to model context, logs or artifacts. The exclusion is an
explicit runtime-policy choice, **not a claim that the excluded constraints are
harmless or semantically equivalent**. Applicable explicit structured
`permissions.deny`/`ask` restrictions remain enforced; the exclusion is not an
unknown native-source exemption. Operator settings remain untouched.

Native higher-authority/Console/managed/remote policy coverage must still be
established and its enforcement must not be disabled. `authored-only` is a
trusted launcher declaration that such managed native restrictions do not apply
to that established deployment; it is not a default or a waiver. A calling agent
cannot supply it through MCP, and an empty private database does not prove it.

In installed V2 2.0.26 the native hard-deny policy action is `tool.use`, matching
`<action>:<resource>` after ordinary permission evaluation. Current public docs
use `permission` for a newer contract; sending that incompatible field to this
installation can drop a denial. Preserve the version-pinned native schema and
test effective denials. Profile/policy initialization and post-import reload are
asynchronous: a listening port or first agent response is not admission readiness.

### Sharded review of large declared inputs

**Threshold and path.** The single-pass path is unchanged for a call whose
role-derived canonical PRIMARY target set has at most 30 entries. Above 30 the
same coordinator run automatically takes the first-class sharded path: one
`Execute`, one `runTask`, one aggregate `Result`, no additional MCP calls, no
new concurrency surface and exactly one posting opportunity. Reviewers never
see the partition as separate calls.

**S0 and the frozen generation identity.** Before any native session the
sharded path reuses the unchanged complete-set guard (trusted `authored-only`
scope; canonicalization/protection; the 256-target count bound; the 1 MiB
per-file bound; the 8 MiB aggregate bound) over the COMPLETE canonical input
set, validates pair declarations, and freezes the generation identity: the S0
manifest digests plus the S0-built policy generation. The frozen generation is
re-validated with the existing rebuild-equality policy check BEFORE every shard
AND replacement (any applicable policy/settings change refuses, even when every
declared input is byte-identical), and every shard/replacement policy is bound
to the S0-frozen digests and the same source-derived policy identity — a unit
can only refuse more, never widen a limit or establish a fresh baseline. A
globally oversized/invalid call refuses with the same named errors as always,
with zero review sessions.

**Sizing and placement.** Each unit's EFFECTIVE manifest is the canonical
UNION cardinality of its owned primaries plus the common context (the full
generated diff and the four readiness guidance targets, independently read in
every unit) plus integration support reads, de-duplicated by canonical path: a
path that is both primary and common counts once, and the generated diff
document is never a primary (a caller-declared role naming it or a guidance
target keeps it primary and de-duplicates). The effective cap of 30 is always
binding; the preferred [25,30] band is reached wherever the arithmetic admits
it, and an arithmetic-impossible below-floor shard is disclosed in the plan
output, `shards.json` and the aggregate summary — never silently reshaped.
Placement is a bounded exact backtracking search over canonical units (dead
ends backtrack; k increments only after a COMPLETED infeasible search; the
canonical exploration order is stable under shuffled inputs, with a plan digest
over shard→owned paths, shard→effective size and pairs→assignment). A search
that exhausts its node bound without deciding refuses the whole plan with
`shard placement search bound exhausted; refusing to plan` (zero native
sessions, no partial plan).

**Pair declarations (declared contract seams).** Both tools accept an optional
`pairDeclarations` argument (CLI: `--pair-declarations-file <path>` on
`run-pr-review`/`run-cross-check`): an array of at most 64 objects
`{ "pairId": "<1–64 chars [A-Za-z0-9._:-]>", "members": ["<path>", …], "class": "<optional label>" }`
whose 2–64 members must canonicalize to members of the call's declared primary
set. Absent or empty declarations are a valid no-pairs call. Malformed supplied
declarations — a bad or duplicate `pairId`, an empty/oversized member list, a
duplicate member, or a member outside the declared set — are refused BEFORE any
native session. A declared pair is an atomic component: co-locatable components
(union with the common context within 30) are placed in ONE shard with explicit
pairing evidence (`pairId`, `class`, `members`, `assignedOwner`,
`coLocation: shard|integration`, `supportReads`). A partition that would
separate a declared pair is refused fail-closed with
`declared contract pair "<id>" was separated by the partition; refusing to run`;
a pair that can never fit any unit's effective manifest is refused by name at
S0. Undeclared semantic seams are a DOCUMENTED LIMITATION: the adapter cannot
infer contract semantics from arbitrary declared paths, so declarations are the
only pairing-evidence source, and a benchmark activation MUST supply them.

**Residual integration slices.** Only a component whose union with the common
context exceeds 30 is decomposed: its INDIVIDUAL pair/tuple obligations (each
tuple kept intact — never a whole component) are placed by the same bounded
union-aware search, so overlapping members become receipted SUPPORT READS
(real full native reads that do not imply file ownership) and FILE ownership
stays exactly-once (a shared member is owned by the slice of the
lexicographically-first `pairId` that includes it). Slices are `INT01..`,
genuine native reviews with the same acceptance and aggregation rules.

**Per-shard acceptance, retries and budget.** Each shard/slice runs serially
under the unchanged acceptance checks plus the exact disclosure token
`COORDINATOR-SHARDED-REVIEW <unit>/<total>: NO SINGLE reviewer sees the whole
source.` in its final summary; receipts are never stitched across attempts.
Per-shard transcripts (`stdout.log`, `raw-response.json` and their SHA-bound
readable views) are MANDATORY evidence: a persistence failure refuses the whole
review as incomplete (`shard_incomplete`) with zero further sessions — a shard
can never be accepted, and the aggregate can never post, on missing native
transcripts. Native-run failures are classified: only RECOVERABLE failures —
under-coverage, timeout/cancellation, incomplete verdict and other non-lifecycle
stage classes (the returned error tree carries no lifecycle/isolation stage,
which also establishes that the owned runtime's teardown, reaping and
private-state removal reported no error) — may shrink/retry. A NON-RETRYABLE
native lifecycle/isolation failure — failed process/group ownership
verification or proof, failed teardown or final reaping, or failed
private-state removal (the documented acceptance vetoes below) — refuses the
WHOLE review as `shard_incomplete` with zero further sessions, no aggregate
acceptance and no posting; the exact stage is recorded on the attempt outcome
IMMEDIATELY after classification — before usage capture and before the
mandatory transcript persistence — and then in the result provenance
(`shardedNativeFailure`), so neither a successful replacement nor a concurrent
transcript-persistence failure can mask it (the transcript failure stays
visible as its own refusal stage). Session accounting reads the runner's
EXPLICIT fresh-native-session fact (set exactly at session creation; the
runner's execution object exists from the moment owned setup begins), never
execution allocation: every STARTED native session — including a
failed/timed-out attempt whose runner returned no execution object — is
accounted exactly once (a nil-execution return contributes an explicit
unknown/unmeasurable usage record; sanitized messages it did return are still
measured), so the aggregate session count always equals the actual
native-session creations. An owned-setup attempt that created NO native session
(owned server spawn, initial process ownership verification, readiness/
settlement, admission, ...) is a SETUP attempt: it is never counted as a
session and is disclosed as its own category (`sharded.setupAttempts` /
`sharded.setupRecords`; no per-session usage artifact), while the documented
pre-start refusal (context expired before owned setup) records nothing. A replacement carries the
explicit frozen common set (a dual-role guidance path stays common for BOTH
halves) and partitions the failed unit's file ownership exactly once, keeping
shared tuple members as receipted support reads. A failed unit is re-run ONLY
itself with its atomic units split
deterministically into at most two halves (a single-unit unit cannot split and
fails the whole review closed); one shrink level, at most two replacements, at
most 3k native runs in total (k includes integration slices). The overall
`OpenCodeMaxDuration` deadline is set once by `runTask` and is NEVER extended:
before each attempt the attempt's context deadline is
`min(overall deadline, now + remaining_budget / remaining_expected_attempts)`
(the retry-aware divisor charges every pending unit 3 attempts and a failed
unit its remaining replacements). The refusal predicate is purely structural —
an attempt starts only when its computed share is positive (equivalently
`remaining_budget > 0`); there is no per-attempt duration constant and no
measured gating threshold. A required replacement that cannot receive a
positive share makes the whole review incomplete. Usage/cost is captured
exactly once per native session — every shard, replacement and failed/timed-out
attempt — as a per-attempt `shards/<unitId>/usage.json` plus the summed
top-level aggregate with its additive `sharded` breakdown (see "Usage and cost
capture"); usage remains warnings-only accounting and never gates acceptance.

**Aggregation and exactly one posted review.** After the last unit completes,
`shards.json` is written once (machine-readable plan/assignments/pairing
evidence/verdicts/coverage; each unit's effective manifest recorded as its
COMPLETE canonical read union with support reads included; authoritative bytes
with a SHA-bound readable sidecar), then the whole-call generation recheck
re-manifests the COMPLETE original S0 target set and rebuilds the complete-target
policy generation with full rebuild equality (policy/settings sources,
permissions and provider projection included): ANY difference — including an
already-accepted shard's owned file changing while a later shard ran, or a
policy/settings source changing while every declared input stayed byte-identical
— makes the aggregate `incomplete` with zero posts, and the baseline is never
silently rebuilt. Only every unit accepted, every verdict in
{approve, pass}, coverage exactly-once (co-located pairs C/C, integration pairs
I/I with receipted support reads, common context read in every unit), the
disclosure verified and the recheck clean yields an aggregate verdict (any
accepted `request_changes`/`fail` ⇒ `request_changes`; otherwise `approve`, or
`pass` for `run_cross_check`). ANY other outcome — missing unit, incomplete,
informational, unsure, budget-refused replacement — is `incomplete` with
`incompleteReason: shard_incomplete`: no posting, no approval. All findings and
all open questions from accepted units are preserved verbatim in the aggregate
result; the aggregate `Summary` additionally embeds every finding's COMPLETE
content (severity, title, details, suggestion, category and location) verbatim
before the unchanged posting renderer, and the whole-complete body is bounded
so it survives inside the unchanged `githubReviewBodyMaxBytes` (60000); a
complete body that cannot fit refuses as incomplete rather than truncating
findings or the disclosure. Posting is exactly one review of record through the
existing strict path, bound to the live head as the configured reviewing
identity (see the posting section below).

### Readiness, terminal evidence and accounting

The finalized manifest includes metadata-only sequential read instructions:
start each required file at offset 1 with a requested limit of 2000 lines, then
continue from the actual returned end line plus 1. Native byte limits can return
short pages, so neither a requested page size nor a short result proves EOF.
In V2 2.0.26 the model sees the rendered canonical path/range header, numbered
contents and any continuation marker, not native `state.metadata` or an EOF
banner. It must inspect all actual contiguous content, follow matching markers
and remaining manifest suffixes, then stop without a past-end EOF probe; a last
line/count or absent marker alone is not proof. Contradictions, clamped lines and
failed reads remain incomplete. Separately, the coordinator validates real native
`metadata.truncated=false` at EOF, source/render SHA-256, EOF/full coverage,
provenance, verified cleanup and verdict integrity before acceptance.
Empty files and the generated diff still require actual completed native reads.
Page-count estimates and work orders are not read receipts or approval; missing
coverage, capacity, context or deadline failures remain incomplete.

Issue #6655: line-split marker lines and reconstruction headers are ordinary
data under this unchanged acceptance model, and no generated input line exceeds
2000 UTF-16 units; nothing in the reader, coverage, digest or terminal
validation is relaxed. An unrendered artifact that still carries an over-long
line fails closed exactly as before with `native read overlong source line`.

Default consuming-backend readiness still requires genuine reads of all four:

- `docs/what-and-why/index.md`
- `docs/silicon-based-how/README.md`
- `docs/silicon-based-how/index.md`
- `docs/silicon-based-how/skills/README.md`

A standalone shared-tools launch explicitly supplies `README.md` and
`agent-coordinator/README.md` with repeatable `--docs-entrypoint`. That override
is bound to the canonical startup repository; a different per-call `repoPath`
must not inherit it. There is no MCP readiness skip/target override, automatic
missing-doc fallback, or hidden Claude preflight. Unproven account/policy/cache
identity uses a fresh OpenCode preflight rather than another backend's cache.
Each actual preflight has separate repository-scoped evidence and
`shared_unapportioned` usage attribution.

`POST .../prompt` admits an input, not a completed verdict; idle is not success.
Acceptance requires owned fresh-session/single-input provenance, successful
native terminal outcome, completed final assistant text with `stop`, the observed
provider/model/variant, required successful native reads, and strict locally
validated verdict JSON. Missing/duplicate/unknown fields, missing recommendations,
null arrays, fences, trailing text, invalid findings and malformed output cannot
be normalized into approvals. Native verdicts also pass the existing substantive
summary and contamination checks, without inventing Claude `StructuredOutput`
tool attempts. Timeout, cancellation, provider/capture/read/model errors or
ambiguous terminal evidence produce incomplete/non-positive results regardless
of any partial approval text.

`execution` provenance distinguishes requested and observed runtime/version,
provider/model/variant, routed model, session, admitted input, final message,
outcome and evidence. `externalModel` is observed, not a requested-selector echo.
Native usage is recorded in the provider-neutral envelope without fake Claude
transcripts. Missing/unmeasured token or cost data is null/unknown, not invented
zero; native session initial totals are not measured spend. Unique reported
messages must not be counted twice. Native cache-write totals do not establish
Claude's five-minute/one-hour split. Legacy Claude-only downstream ledger parsers
do not automatically account for this backend; native `usage.json` is the owning
record. Optional accounting failures warn and never turn a valid review into an
execution failure.

### Head-bound posting and downstream activation

OpenCode reviewers never authenticate or post to GitHub. The coordinator owns
posting under `githubUser`, with a pinned-token login assertion. The strict
OpenCode path checks the canonical reviewed head against a successfully resolved
live PR head **before idempotency or writing**, creates the review with explicit
`commit_id`, and confirms the exact author, requested state, commit, review ID
and current head. Unknown, moved or ambiguous delivery remains nullable and
cannot be a successful requested-posting gate. `postReview: false` is explicitly
advisory, not merge approval. Legacy Claude/fixture delivery semantics remain
compatible. No atomic GitHub check-and-write transaction is claimed; a head move
after sampling still invalidates current-head confirmation.

Issue #6655 hardens that identity without relaxing any guard, and applies
identically to single-pass and sharded runs. BEFORE the existing-match
idempotency lookup, the run fixes: (1) the EXPECTED numeric author UID, resolved
from the authenticated pinned reviewing identity itself (never from the review
payload; a missing, nonpositive or unparseable UID is refused as indeterminate),
and (2) the EXPECTED exact body — the bytes `renderReviewBody(result, headSHA)`
produces for THIS run — with `expectedBodyHash = SHA-256(expectedBody)` (the
trailer embeds the run's `requestId`, so the binding is run-scoped by
construction). The create-review record, the existing-match qualification and
the confirmation poll all require numeric equality `review.user.id == expectedUID`
AND byte-exact body hash equality, in addition to the existing login/state/commit
checks. The latest author/head verdict review is selected FIRST: an older
same-state review cannot hide a newer opposite-state one, and an opposite-state
review never blocks the fresh commit-bound POST for this run's requested state
(its body normally differs from this run's by verdict text and run-scoped
requestId). The exact-body qualification applies only to a candidate carrying
the requested state; a legacy-criteria candidate (same author login, state and
commit) whose numeric UID or body differs is INDETERMINATE: no second POST, no
success claim. A sharded aggregate whose complete body would
exceed `githubReviewBodyMaxBytes` (60000) refuses rather than posting truncated
findings or a truncated disclosure. Any GitHub-side byte normalization or a
record without a numeric user ID therefore reports indeterminate
(`githubPostingConfirmed: null`) rather than a false success; this is measured
at consuming activation.

Example standalone shared-tools launch, **after** trusted policy scope is
established and an independently verified free coordinator port is chosen
(the Sol-high review-gate selection; the cost-oriented example is
`--opencode-model openrouter/deepseek/deepseek-v4.1-flash#max`):

```sh
./make.sh
umask 077
./run.sh --mcp --agent opencode \
  --repo-path /path/to/shared-tools-worktree \
  --opencode-model openrouter/openai/gpt-6.1-sol#high \
  --opencode-max-duration 30m \
  --opencode-policy-scope authored-only \
  --docs-entrypoint README.md --docs-entrypoint agent-coordinator/README.md \
  --listen 127.0.0.1:<verified-free-port>
```

Example advisory MCP call (ordinary artifact/head fields retain their meanings;
this example uses the issue's cost-tier `deepseek/deepseek-v4.1-flash#max`
selection, explicitly at the call layer):

```json
{
  "agent": "opencode",
  "model": "openrouter/deepseek/deepseek-v4.1-flash",
  "variant": "max",
  "objective": "Independently check the supplied evidence",
  "repoPath": "/path/to/reviewed-worktree",
  "artifactFiles": ["path/to/evidence.json"],
  "headSha": "<full-40-character-reviewed-commit>",
  "postReview": false
}
```

Build from the owning full shared-tools checkout; start only the intended
coordinator on its own verified port. A new source revision requires a fresh
coordinator process and capability-stamp/checkout ownership verification; a
binary rebuilt on disk does not update an already-running process. For backend
consumers, the consuming train repins its **isolated** shared-tools gitlink,
builds, restarts only its own coordinator and performs its own mandatory gates.
Never edit the shared backend checkout/submodule, alter other fleet servers or
equate source PR merge with consuming activation or plan approval. No production
deployment or tracking-issue closure follows from an adapter merge.

## Claude configuration

The coordinator assumes `claude` is available in the environment and requires `--mode` (or `AGENT_COORDINATOR_CLAUDE_MODE`) for live Claude runs (issue #3860; `auth-token` removed and `ambient` added by issue #3872). `--mode` selects how the `claude` CLI authenticates, and is REQUIRED with no built-in default: `oauth-token` (injects `CLAUDE_CODE_OAUTH_TOKEN`), `api-key` (injects `ANTHROPIC_API_KEY`), or `ambient` (injects no credential env var at all — see below). Omitting `--mode` fails fast with `claude auth mode is required for live claude runs: pass --mode (or set AGENT_COORDINATOR_CLAUDE_MODE) to one of oauth-token, api-key, or ambient`; selecting `oauth-token` or `api-key` with its credential env var unset fails fast with `<ENV_VAR> is required for live claude runs when --mode=<mode> is selected` — mode selection is not a fallback chain, so a different credential var being present does not help.

The live Claude adapter strips competing auth inputs such as `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `CLAUDE_CODE_OAUTH_TOKEN`, and `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` before launching `claude` — unconditionally, regardless of mode — then injects only the selected mode's own env var and value. `ambient` mode is the exception on the injection side only: it strips the same competing auth inputs but injects nothing, so authentication for `oauth-token`/`api-key` relies on exactly one of the stripped variables being restored, while `ambient` relies on none of them.

#### `ambient` mode

`ambient` mode (issue #3872) authenticates the spawned `claude` subprocess using the host's own existing interactive `claude` login instead of an injected credential environment variable. Concretely: with `CLAUDE_CONFIG_DIR` unset, `ambient` mode defaults the Claude config directory to `~/.claude` (via `os.UserHomeDir()`) and copies it into a fresh per-call temporary directory, giving the subprocess access to `~/.claude/.credentials.json` from the operator's own interactive login session. An explicit `CLAUDE_CONFIG_DIR` still wins over this default, exactly as for the other modes.

This design accepts two properties as intentional tradeoffs, not open questions:

- Usage under `ambient` mode draws against the host account's own rate limit and session, rather than a dedicated service credential.
- If the underlying refresh token rotates during a call, that rotation does not propagate back to `~/.claude` (the run only ever touches its own throwaway copy) and does not carry forward to the next call either, since each call re-copies `~/.claude` fresh.

The coordinator relies on Claude CLI support for `--strict-mcp-config` and `--include-partial-messages`. The current repository workflow has been validated with `claude --version` reporting `2.1.76 (Claude Code)`.

When GitHub CLI access is needed, the coordinator clones the current `gh` config into a temporary `GH_CONFIG_DIR` for the Claude run. If `CLAUDE_CONFIG_DIR` is already set, that directory is also cloned into a temporary workspace before launch. This keeps review runs from mutating the operator's active GitHub or Claude CLI state.

Before any live Claude-backed work proceeds, the coordinator asks Claude to confirm that it has read `<repo-root>/docs` to understand the repository's defined skills and guidance. The run halts if Claude cannot confirm that readiness check.

During live Claude runs, the coordinator streams progress to stderr from Claude's `stream-json` output and emits periodic "still running" heartbeats during quiet periods so operators can distinguish active analysis from a stuck process. The live stream reports lifecycle events and heartbeats, but it does not echo Claude's partial tool-input JSON fragments.

Optional overrides:

- `--claude-bin`
- `--claude-model`
- `--claude-effort`
- `--mode` (required for live Claude runs; `oauth-token`, `api-key`, or `ambient`; no default — also settable via `AGENT_COORDINATOR_CLAUDE_MODE`)
- `--claude-permission-mode`
- `--claude-tools`
- `--claude-max-budget-usd`
- `--claude-max-duration`

`--claude-model` defaults to `claude-sonnet-5` and `--claude-effort` defaults to `high`, so live coordinator reviews run on Sonnet 5 at high effort unless overridden (issue #3105 withdrew Fable as the coordinator default). `--claude-model` accepts either a latest-model alias (`fable`, `opus`, `sonnet`) or a full model name (`claude-fable-5`); `--claude-effort` accepts `low`, `medium`, `high`, `xhigh`, or `max` and is rejected at startup if invalid. Both can also be set through `AGENT_COORDINATOR_CLAUDE_MODEL` / `AGENT_COORDINATOR_CLAUDE_EFFORT`, or overridden per MCP call via the `model` and `effort` arguments on `run_pr_review` and `run_cross_check`.

`--claude-max-duration` defaults to `30m`. Set it to `0` only when you intentionally want to disable the wall-clock safety bound for long-running reviews. When the bound is enabled (non-zero), the coordinator also tells the reviewer about it: the prompt states the hard wall-clock limit plus a soft self-imposed deadline at 80% of that duration, instructing the reviewer to stop exploring and emit `StructuredOutput` with whatever it has established once the soft deadline is reached.

### Incomplete runs (timeout, execution error, unparseable stream)

A run that is killed by `--claude-max-duration`, exits with an execution error, or produces a stream the coordinator cannot parse no longer discards the review payload (issue #3053). Instead `RunPRReview`/`RunCrossCheck` return a non-nil `*coordinator.Result` alongside a non-nil error:

- `Status` is `"incomplete"` (as opposed to `"completed"`).
- `Recommendation` is forced to `"incomplete"` **unconditionally** — even if the reviewer's own output had already parsed as `approve`/`pass` before the run failed. This is a safety property, not an oversight: `"incomplete"` falls outside any `{approve, pass}` gate allowlist, so a killed or errored run can never satisfy a review gate.
- `AgentRecommendation` preserves whatever the reviewer actually parsed as its recommendation, verbatim, for human inspection.
- `IncompleteReason` is one of `timeout`, `execution_error`, or `parse_error`.
- `Error` carries the underlying error text.
- `Summary` and `RawOutput` carry the reviewer's (redacted) output, bounded to 8 KiB and 64 KiB respectively; both fields are redacted against the FULL, untruncated text before any bounding is applied, so a secret cannot survive by straddling the truncation boundary. `RawOutputTruncated` is `true` when `RawOutput` was clipped; the complete, redacted copy always remains on disk at `Artifacts.RawStdoutLog`.

Over MCP, `run_pr_review`/`run_cross_check` surface this the same way they surface any other failed call: `isError: true`, with `structuredContent` now carrying the full incomplete `Result` (including `RawOutput`) instead of a bare error string.

As of issue #3220, a cleanly parsed live verdict can still be forced onto the
incomplete path by the integrity gate, before normalization. Reasons include
`degenerate_verdict` (short/empty/placeholder summary) and
`envelope_contamination` (unquoted envelope fragments in string fields).
OpenCode applies these native single-verdict substance/contamination checks too;
the deterministic fixture remains exempt.

Claude additionally collects real `StructuredOutput` attempts and can report
`verdict_downgraded` when an earlier attempt had more findings, higher maximum
severity at equal finding count, or a summary whose accepted replacement
collapsed below 5% of its bytes. Issue #3727 replaced a bare byte-shortfall test
that falsely rejected cleaner, more-complete retries. The best-scoring recovered
attempt supplies incomplete `Summary`/`Findings`, while `AgentRecommendation`
preserves the actual accepted parse. `StructuredOutputAttempts` and
`StructuredOutputContentBearingAttempts` count Claude's actual tool invocations,
not occurrences in schema echoes or native OpenCode text. OpenCode never
fabricates these counters or Claude attempts; ambiguous native final emissions
are rejected by its own terminal validator.

When a run needs Claude to use GitHub CLI, pass `--github-user <login>` to the `run-pr-review`/`run-cross-check` direct-CLI subcommand. The coordinator supplies the selected named token and instructs Claude to verify `gh api user --jq .login` before any other `gh` command. It must report a mismatch rather than switch accounts or attempt to repair the supplied identity. OpenCode reviewers receive no GitHub credentials and leave posting entirely to the coordinator.

If the run must actually post GitHub state, such as a PR review or PR comment,
**as of issue #3736 the coordinator process itself posts it** — see the
"Server-side GitHub review posting" section below — needing no `Bash` grant.
Including `Bash` in `--claude-tools` remains useful as a legacy fallback: it
lets the spawned reviewer subprocess ALSO shell out to `gh` and post its own
review if the coordinator's own post is skipped or refused, but it is no
longer the only mechanism, and it is not required for a review verdict to
reach GitHub.

### Server-side GitHub review posting: `postReview`, `githubReviewPosted`, `githubReviewPost`

As of issue #3736, `agent-coordinator` posts the verdict review itself,
directly from Go (`exec.Command("gh", ...)`, no spawned subprocess involved),
as the configured `githubUser`. `postReview` (boolean, tri-state decode) on
both `run_pr_review` and `run_cross_check` defaults to `true`: whenever
`prNumber` and `githubUser` are both supplied and the run passes its
eligibility gate (`evaluatePostEligibility` in `github_review_post.go`) —
`Status == "completed"`, an empty `IncompleteReason`, a recommendation that
maps to a verdict state (`approve` -> `APPROVED`, `request_changes` ->
`CHANGES_REQUESTED`; every other value, including the #3220 `"incomplete"`
sentinel, refuses), and `ReviewedHead.Verified == true` — the coordinator
posts that review as `githubUser` via `GH_TOKEN="$(gh auth token --user
<githubUser>)" gh pr review <n> --approve|--request-changes --body-file
<tmp>`. None of those four verdict-safety conditions is weakened elsewhere:
they are what keeps a #3220-downgraded or #3063-unbound verdict from ever
being posted. Pass `postReview: false` to opt a genuinely advisory-only call
out of posting entirely.

Before posting, the coordinator checks whether the reviewer subprocess
already posted a qualifying review at the live head (the pre-#3736 compliant
case) and skips a redundant post rather than double-posting.

The result carries `GitHubReviewPosted` (`*bool`, never `omitempty`, the
same tri-state discipline as `GitHubPostingConfirmed` below: `null` means no
post was attempted because the gate refused or the caller opted out,
`false` means an attempt was made and failed, `true` means either this
server posted or a qualifying review already existed) and `GitHubReviewPost`
(an attempt record: `state`, `skipped`, `reason`, `error`). A failed post
(GitHub rejects self-approval with `422`, network failure, auth failure,
missing `gh` on `PATH`, or no `gh auth token --user <githubUser>` could be
resolved) sets `GitHubReviewPosted: false` with the error text in
`GitHubReviewPost.Error`. The analysis itself is still valid — `Result` (and
its `Status`) are carried in `structuredContent` completely unchanged, so a
manual relay of the verdict remains possible — but as of issue #5477, a
DEFINITE posting failure (`GitHubReviewPosted == false`) now also sets the
MCP envelope's own `isError: true`, the same mechanism `mcp_partial.go` uses
for a different failure shape, so no caller can silently read a failed post
as a clean success. `GitHubReviewPosted: null` (no attempt was made at all —
the gate refused, or the caller opted out) does **not** set `isError`; only a
genuine, attempted-and-failed post does. A successful post
(`GitHubReviewPosted == true`) leaves the envelope byte-identical to before
issue #5477. Every branch where nothing verdict-shaped was posted also
appends a loud sentence to `content[0].text`: "this result is advisory only
and satisfies no merge gate."

### Repository resolution for `gh`: `repoPath`, `GH_REPO`, and the submodule cwd trap (issue #5477)

Every `gh`-invoking call this package makes — the post above, and the
delivery-confirmation lookup below — runs with `cmd.Dir` set to the
**effective** repository path: the caller-supplied `repoPath` MCP argument
when non-empty, else the coordinator server's own launch-time
`AGENT_COORDINATOR_REPO_PATH` (`mcpHTTPServer.effectiveRepoPath`,
`mcp_http.go`). Before issue #5477, the six gh-facing call sites in
`mcp_http.go` used the raw, un-normalized `repoPath` MCP argument directly,
so a caller that omitted it left `cmd.Dir` empty and the spawned `gh` process
inherited the coordinator server's OWN working directory — which, because
`agent-coordinator`'s own source lives in the `shared-tools` git submodule,
is normally `.../shared-tools/agent-coordinator`, a **different repository**
from the one under review. `gh` then resolved every request against
`ayode-institute/shared-tools` instead.

As defense-in-depth on top of `effectiveRepoPath`, `runGHCommand`
(`github_review_confirm.go`) also sets `GH_REPO=<owner>/<name>` in the
environment it passes to `gh`, derived from the effective repoPath's own
`origin` git remote (`resolveGitHubRepoSlug`, `github_repo_slug.go`).
`GH_REPO` covers a case a `-R/--repo` flag cannot: `gh api`'s
`{owner}`/`{repo}` placeholder substitution (used by the delivery-confirmation
lookup) has no per-call `--repo` flag equivalent, and honors `GH_REPO`
instead. Derivation is deliberately soft-failing — an unparseable or
non-GitHub remote simply leaves `GH_REPO` unset, falling back to `cmd.Dir`
resolution, rather than hard-failing the call.

### Delivery confirmation: `githubPostingEnabled` vs. `githubPostingConfirmed`

`run_pr_review`/`run_cross_check` results carry two independent GitHub
delivery signals, computed at different times and answering different
questions:

- `GitHubPostingEnabled`/`GitHubPostingNote` (issue #3176) is a fact known
  **before** the run, and describes ONLY the legacy subprocess posting path:
  `true` means the SUBPROCESS could have posted (`Bash` present in
  `--claude-tools` and `githubUser` non-empty), never that anything was
  actually posted, and never a statement about the coordinator's own
  server-side post (`GitHubReviewPosted` above), which needs no `Bash` grant
  at all. The two fields are deliberately never merged.
- `GitHubPostingConfirmed`/`GitHubReviewConfirmation` (issue #3212) is
  computed **after** the run, by querying GitHub's own state over `gh`.
  Supply the optional `prNumber` argument (and, when known, `headSha`, which
  as of issue #3063 also binds the review itself — see the next section) on
  either MCP tool call to opt in. `GitHubPostingConfirmed` is a three-state
  `*bool` that is **always present** in `structuredContent` (never
  `omitempty`): `null` means not checked/indeterminate (no `prNumber`
  supplied, NEITHER the subprocess path (`GitHubPostingEnabled`) NOR the
  server's own post attempt (`GitHubReviewPosted`) was possible, `gh`
  unavailable, or the live query itself failed — never manufactured from a
  failure into `false`); `false` means the query succeeded and found no
  qualifying review; `true` means a qualifying review was found, with
  `GitHubReviewConfirmation` carrying its id, state, author, commit SHA,
  submission time, and the live `reviewDecision`. A qualifying review is a
  real **verdict** — `state` `APPROVED` or `CHANGES_REQUESTED` at the pull
  request's live head SHA, authored by `githubUser` when supplied — so a
  plain `gh pr comment`, or a `COMMENTED` review, does not qualify (the exact
  shape of the incident that motivated this issue: the review text reached
  GitHub, the review verdict did not). As of issue #3736, confirmation is
  attempted whenever EITHER capability was present, so a Bash-less server
  that posted its own verdict is never left unable to confirm its own post.

`GitHubPostingConfirmed: true` is delivery confirmation only, never verdict
confirmation — see issue #3220 for why a confirmed-delivered placeholder
verdict is still a bad verdict.

As of issue #5477, `GitHubPostingConfirmed: false` (the query succeeded and
found no qualifying review) — like `GitHubReviewPosted: false` above — sets
the MCP envelope's `isError: true`, alongside the unchanged
`structuredContent`. `GitHubPostingConfirmed: null` (indeterminate) does not.

### Head-SHA binding: what the review is bound to

Issue #3063 closed a distinct gap from the delivery confirmation above:
`resolveDiffPayload` computes a ref-based diff with a bare, unverified
`git diff <baseRef>...<headRef|HEAD>` inside the checkout the coordinator was
launched against. If that checkout's ref lags a push made elsewhere (e.g. the
main clone that pushed the real PR head), the review silently reviews the
WRONG commit and reports a verdict as if it were the PR's real head — and
until this issue, nothing recorded which commit was actually reviewed.

Every `run_pr_review`/`run_cross_check` result now carries a `ReviewedHead`
record (`reviewedHead` in `structuredContent`), **always present** as a
non-null object (never `omitempty` — the same tri-state discipline as
`GitHubPostingConfirmed` above):

- `method`: `"ref-resolve"` (strong — the diff came from resolving a ref in
  this checkout, and the resolved SHA was compared against the expected one),
  `"containment"` (weaker — the diff was precomputed, or the workflow is
  `cross_check`; the checkout provably HAS the expected commit, but the
  patch's own provenance is unproven), or `"none"` (no expected SHA was
  supplied at all).
- `verified` (`bool`): whether the binding held. **This is the field a
  caller/gate must read** — never infer binding from the human-readable text.
- `expectedSHA`/`expectedSource` (`"caller"` or `"pr-lookup"`) and
  `resolvedSHA`/`ref` record the evidence.
- `reason`: populated whenever `verified` is `false`, explaining why (no SHA
  supplied, or a `pr-lookup` derivation failed).

**Fail-closed on mismatch, fail-open on absence.** Supplying `headSha` (or
`prNumber`, which derives it via a pre-run GitHub lookup) that disagrees with
what this checkout actually resolves is a **hard, pre-execution error** — the
call fails with `isError: true` before any reviewer subprocess is spawned, no
artifact claims a verdict, and no cost is incurred. Omitting both `headSha`
and `prNumber` preserves prior behavior exactly: the run proceeds, and
`reviewedHead.verified` is simply `false`. A `pr-lookup` derivation that
itself fails (e.g. a transient `gh` outage) also does **not** fail the run —
hard-failing every review on a `gh` outage would trade a silent-approval
defect for an availability defect — the run proceeds UNBOUND and
`reviewedHead.reason` names the lookup failure.

Note that `headSha` now does double duty: the SAME effective value (caller-
supplied, or `pr-lookup`-derived when omitted) is used for both this pre-run
binding and the post-run `GitHubPostingConfirmed` staleness guard described
above — one field, one meaning, covering both ends of the run.

If `--claude-tools` is not provided, the coordinator defaults Claude to `Read,Glob,Grep,StructuredOutput` and explicitly disallows `Agent`, `Task`, and `TaskOutput` so startup verification and the reviewer SUBPROCESS stay read-only and bounded. This no longer means the RUN as a whole cannot post to GitHub: as of issue #3736, the coordinator process itself (not the subprocess) can post the verdict review directly, independent of the subprocess's own tool grants — see "Server-side GitHub review posting" above. Claude is launched with `--setting-sources local`, so repository-controlled `.claude` settings files are not loaded into review subprocesses.

If you explicitly enable `Bash` in `--claude-tools` while leaving `--claude-permission-mode` at the default `bypassPermissions`, the Claude subprocess gets unrestricted shell execution within the exposed `--add-dir` paths. The coordinator emits a stderr warning in that configuration.

### Operator-authorization provenance (analysis only, not implemented)

`instructions` (see above) is rendered under a heading that denies it
authority as a human-operator ruling (issue #3560), and today that is the
*only* caller-supplied channel this tool has — there is no second channel
that actually carries a verifiable operator attestation. Issue #3639
analyzed candidate designs for such a channel and found that the calling
agent and this server are the same OS principal on the same host, so no
purely on-host trust root is unforgeable by the caller. See
[`docs/what-and-why/architecture/operator-authorization-provenance.md`](../../docs/what-and-why/architecture/operator-authorization-provenance.md)
for the full analysis, the candidate designs, and the recommendation. That
document's recommendation is marked pending an operator ruling and **is not
adopted**: this tool has no attestation channel, no related MCP argument,
and no related configuration surface today.
