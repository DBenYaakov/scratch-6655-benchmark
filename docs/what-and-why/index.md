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

# What-And-Why Index

This index covers the descriptive material for the repository: architecture,
runtime behavior, system boundaries, topology, and integration context.

## Start Here

- [README.md](README.md)
- [architecture/architecture.md](architecture/architecture.md)
- [architecture/project-structure.md](architecture/project-structure.md)

## Core Architecture

- [architecture/database.md](architecture/database.md)
- [architecture/backend-operational-decisions.md](architecture/backend-operational-decisions.md)
  Cross-cutting backend decisions consolidated from session history.
- [architecture/affected-stack-computation.md](architecture/affected-stack-computation.md)
  Why the affected-stack contract fails loudly rather than returning an ambiguous
  empty set, why a source-derived set is necessary but not sufficient for a deploy,
  and why the considered-path count — not the four per-path buckets — is what
  distinguishes an honest empty set from a comparison that was never performed.
- [architecture/cloudformation-drift-and-recovery.md](architecture/cloudformation-drift-and-recovery.md)
  Why CloudFormation drift detection cannot see the drift that motivated it and therefore
  reports coverage rather than a boolean, why rollback recovery must derive its skip set
  live, and why a system-schema migration must run identically in every flowstate.
- [architecture/resolution-trains.md](architecture/resolution-trains.md)
  What a resolution train is and why it is shaped that way: the Dispatcher/Conductor
  role split, the definition of a train's *cargo* and who decides what a train carries,
  why cargo is grouped by shared stack (and the file-footprint fallback when the
  stack set is empty), the coordination primitives and what each one prevents,
  and the operational limits (claim leases, directive TTLs, Conductor context
  life, parked-state ambiguity, handoff durability).
- [architecture/batch-integration-validity.md](architecture/batch-integration-validity.md)
  Why per-feature approval is not batch correctness: the defect classes that exist
  only in the integration of separately-approved features, and why a cross-cutting
  audit must be re-run at integration time rather than at issue time.
- [architecture/validation-run-correlation-boundary.md](architecture/validation-run-correlation-boundary.md)
  Why the #2756 cross-process validation-run correlation was declined: the
  dispatcher never opens api-check's private store, the one admissible direction's
  cost outweighs its retrospective-only benefit, and the checkable trigger that
  would reopen the decision.
- [architecture/verification-that-verifies.md](architecture/verification-that-verifies.md)
  The recurring defect of a check that reports success without having verified
  anything — in gates, in classifier defaults, in acceptance criteria, in doc
  comments, in swallowed command failures whose zero value is read as an observed
  fact, and in drift guards that have never been attacked — plus the sibling
  reporting-side defect of an indeterminate result rendered as a negative one
  (`hb 0s`, a `0` count, an omitted key) rather than as unknown — and the rules
  for not shipping another one.
- [architecture/operator-authorization-provenance.md](architecture/operator-authorization-provenance.md)
  Analysis (not a decision) of why no on-host channel from a calling agent to
  the `agent-coordinator` server can be an unforgeable trust root for a human
  operator's ruling — the caller and the server are the same OS principal —
  and the five candidate designs (C1–C5) considered, with C3 (server-fetched
  GitHub-identity attestation) recommended pending an operator ruling that
  has not yet been made (issue #3639, follow-up to #3560).
- [architecture/authentication-system.md](architecture/authentication-system.md)
- [architecture/create-readback-privilege-enforcement.md](architecture/create-readback-privilege-enforcement.md)
  Why a lessons-domain `X/create` also needs `X/read`: the create handlers read
  their row back through a separate procedure that runs its own authorization
  frame. Records the four-round under-granting history (#4829 -> #4881 -> #4971
  -> #4978), why each round's sizing source could not be complete, and the DOC-9
  gate that now enforces the rule mechanically — including its declared blind
  spot (it is directional and cannot see an over-declaration).
- [architecture/self-grant-privilege-actions.md](architecture/self-grant-privilege-actions.md)
  How to grant every verified user a bare, self-scoped privilege with no
  realm/role of its own (attach to the platform's existing Private-Users
  role, never `PreconfiguredPositionAssignments`), and the Private-vs-Public
  attachment-point decision rule.
- [architecture/sms-abuse-protection.md](architecture/sms-abuse-protection.md)
  Outbound-SMS country restriction via an account-default AWS End User Messaging protect
  configuration: the two-mechanism split CloudFormation cannot express alone, why full
  enumeration is load-bearing (undeclared codes default to ALLOW), the inert first phase of
  the bootstrap, and the abuse vector this control does NOT close (#3593).
- [architecture/merlin-messenger.md](architecture/merlin-messenger.md)
  Contextual chat, REST/SSE split, user-EID identity propagation, and deployment dependencies.
- [architecture/notification-delivery-events.md](architecture/notification-delivery-events.md)
  SES bounce/complaint/delivery event pipeline (`dynamic-api-system`), build-fan-out vs.
  functional-wiring radius for the shared kernel file, `_v2` suppression enforcement, per-recipient
  BCC-exclusion fan-out, worker error-handling shape, and the SNS-ingestion testing boundary (#4207).
- [architecture/merlin-kernel-boundaries.md](architecture/merlin-kernel-boundaries.md)
  Kernel-only runtime/database/filesystem function inventory.
- [architecture/session-management.md](architecture/session-management.md)
- [architecture/browser-observability.md](architecture/browser-observability.md)
- [architecture/job-orchestration.md](architecture/job-orchestration.md)
- [architecture/orchestration-system.md](architecture/orchestration-system.md)
- [architecture/jobasync-generator-script-contract.md](architecture/jobasync-generator-script-contract.md)
  The `jobAsync` generator phase contract ahead of implementation: no input beyond
  the bundled generator, exactly two declared outputs (`./input`/`./expected`),
  fresh-random by default with enclave-based opt-in stickiness, and a test-case
  count fixed at one by the orchestrator itself.
- [architecture/jobasync-comparator-contract.md](architecture/jobasync-comparator-contract.md)
- [architecture/jobasync-challenge-lifecycle.md](architecture/jobasync-challenge-lifecycle.md)
  `jobAsync` challenge-attempt dispatch, per-(student, ChallengeAssignment) job-group
  claim/reuse (concurrency-safe conditional-UPDATE pattern), `ChallengeAttempts.jobExecutionID`
  linkage, and the content-addressed job-execution-content EFS store (#4239, #4243, #4244,
  #4251, #4253, #4270).
- [architecture/jobasync-orchestrated-cpp-pipeline-spike.md](architecture/jobasync-orchestrated-cpp-pipeline-spike.md)
  Phase 0 spike (Epic #4644) for the first real three-phase orchestrated jobAsync pipeline
  (generate/enclave-runner -> run/cpp/lint-compile-run-stdin -> compare/standard): runner bundle
  handling, lint tooling, the `standard-compile-and-run` sandbox policy decision, and the
  reproducible hand-post api-check harness (#4645).
- [architecture/ai-orchestration.md](architecture/ai-orchestration.md)
- [architecture/ai-tuner.md](architecture/ai-tuner.md)
  Prompt inspection and A/B comparison CLI: deployed-registry enumeration, dual execution surfaces, mirror-fidelity invariants (#2498).
- [architecture/ai-http-endpoint-failure-semantics.md](architecture/ai-http-endpoint-failure-semantics.md)
- [architecture/agentic-stack.md](architecture/agentic-stack.md)
- [architecture/github-issue-lifecycle-webhook.md](architecture/github-issue-lifecycle-webhook.md)
  `black/dev-automation` GitHub webhook receiver, the Function-URL-to-API-Gateway pivot (org guardrail), and the ARFC-1008 pointer for enforcement invariants (#2434).
- [architecture/dev-ai-inference-endpoint.md](architecture/dev-ai-inference-endpoint.md)
  `black/dev-ai` self-hosted vLLM inference endpoint on a single g6e.12xlarge, reachable from both
  supervisors over the shared security group; the imperative volume-attach and stack-owned-bucket
  bootstrap patterns it reuses, and the scheduled hard-stop that bounds idle cost (#5202).
- [architecture/shared-operational-notifications.md](architecture/shared-operational-notifications.md)
  `black/notifications`: one shared routine/alert SNS topic pair per silo (development, production,
  and engagement per directive 1205), centrally derived recipients, declarative local SSM
  publication, the dual-routing migration of `black/dev-automation`/`black/dev-ops` alarms and its
  retirement (#6346), and the narrow ops-console engagement-silo scoping extension (#6321).
- [architecture/shell-access.md](architecture/shell-access.md)
- [architecture/shell-credential-lifetime.md](architecture/shell-credential-lifetime.md)
  The two credentials a shell session uses, why the pod's JWT went stale unnoticed, the
  600 s freshness gate at both seams, and why it ships disabled.
- [architecture/shell-protected-services.md](architecture/shell-protected-services.md)
  User-specific protected web services in shell environments: native sidecar placement, per-session
  subdomain routing, two-layer (definition/content) registered authorization model, and gateway-push
  delivery with zero credentials in the service container (#1875).
- [architecture/shell-service-seeded-mariadb.md](architecture/shell-service-seeded-mariadb.md)
  The published `seeded-mariadb` service definition for privilege exercises: the bounded data-only
  `seed.json` content contract, safe SQL construction, the allowlisted-grant security model, and
  the measured `/srv`/`/tmp` budget with its disclosed residual risk (#5979).
- [architecture/go-dependency-floor-sweeps.md](architecture/go-dependency-floor-sweeps.md)
  Why repo-wide Go module-floor sweeps exist (the `govulncheck` build gate), and why a sweep must also tidy the shared-library consumers it skipped — `common-make.sh` runs `go mod tidy`/`gofmt` in place on tracked sources with no guard.

## Deployment Topology

- [architecture/partition-routing.md](architecture/partition-routing.md)
- [architecture/dynamic-stack-splitting.md](architecture/dynamic-stack-splitting.md)
- [architecture/poc-hosting-publication.md](architecture/poc-hosting-publication.md)
  Why POC publication is owned by a shared orange S3 stack, how committed source
  refs and a directory allowlist make publication replayable, and why deployment
  must publish objects before the distributor begins routing `/_/pocs/*`.
- [architecture/engagement-account-foundation.md](architecture/engagement-account-foundation.md)
  The dedicated engagement AWS account/silo (issue #6242): the already-verified
  Identity Center/CloudTrail/budget prerequisites, the three pink singleton
  foundation stacks (schema/network/storage) and their cold-start, zero-
  cross-stack-read publication pattern, the `StorageBucketName` SSM
  contract #6243 depends on (superseding the retired `engagement-artifacts`/
  `OpsStateBucketName`, issue #6245), and the baseline-gap inventory
  CloudFormation cannot represent.
- [architecture/engagement-supervisor.md](architecture/engagement-supervisor.md)
  The on-demand, right-sized engagement supervisor (issue #6245): its storage
  and roles stacks, the engagement key pair (AWS KeyName `apollo`), the SSM-only access
  model and state-persistence design, the `engagement-storage` consolidation
  and `engagement-artifacts` retirement, and the E2 interim-naming
  decision (renamed by #6301).
- [architecture/sendy-host.md](architecture/sendy-host.md)
  The Sendy application host (issue #6248): topology, Elastic IP and apex A record; the bootstrap
  ordering and the TLS seam contract with #6309; the #6246 release-script interface contract; the
  single-executor design (single-attach EBS lease plus the `SchedulerEnabled` SSM switch plus
  `flock`), with failure-mode analysis for restart, replacement and restore; time-zone handling and
  the DST table; the IMDS owner restriction; bootstrap-only deploy mode with the delegated-mode
  follow-up; and the `mount-home-volume.sh` fork rationale.
- [architecture/engagement-sendy-mail.md](architecture/engagement-sendy-mail.md)
  Engagement SES marketing delivery (issue #6249): the stack-owned/Sendy-owned SES/SNS ownership
  split (ruling #6246 item 3), the DKIM/MAIL FROM/DMARC record inventory and alignment, why the
  default configuration set is used and why no `FeedbackAttributes` are set, the reputation-alarm
  thresholds and the account-level-metric caveat, and the `pink/sendy-mail` SSM contract #6246
  consumes.
- [architecture/sendy-certificate.md](architecture/sendy-certificate.md)
  The Sendy host's HTTPS certificate (issue #6309): Let's Encrypt DNS-01 (no port 80), the
  `SendyCertbotRoute53` IAM scope (ListHostedZones/GetChange, plus ChangeResourceRecordSets scoped by
  `ForAllValues` to exactly the `_acme-challenge` TXT record), the `/srv/sendy-data/letsencrypt`
  bind-mount design, the #6248 seam, the three certificate alarms, the key-material guard handling,
  and bootstrap-mode-only deployment with the delegated-mode follow-up.
- [architecture/services-account-foundation.md](architecture/services-account-foundation.md)
  The dedicated `services` AWS account/silo (issue #6333, epic #6332): the shared home for
  platform tooling state (starting with `red/ops-state`'s single DynamoDB table, replacing
  ops-console's per-checkout SQLite and whole-file S3 snapshot sync), why it is neither Primary
  nor an existing workload silo, the operator-run bootstrap procedure's account record (PENDING
  until the operator's read-back fold-back), and the interface #6335–#6338 build against.
- [architecture/sendy-credentials-and-release.md](architecture/sendy-credentials-and-release.md)
  Sendy's dedicated least-privilege IAM user and CloudFormation-generated access key (issue #6246):
  why an access key rather than a role, the grounded IAM action matrix, the stack-owned/Sendy-owned
  SES/SNS split, the folder-plus-manifest release contract and its three host scripts
  (install/configure/seed), rotation, failure behaviour, and the delegated-mode gap.
- [architecture/engagement-sendy-storage.md](architecture/engagement-sendy-storage.md)
  `pink/sendy-storage` (issue #6247): the private RDS MySQL 8.4 instance and retained data volume
  backing Sendy's subscribers/campaigns/uploads and Certbot's certificate state, the E1 MySQL
  8.0-versus-8.4 engine-version decision (Dispatcher directive 1275), the daily/weekly AWS Backup
  tiers and their alignment with native RDS PITR, the consumer SSM contract #6248/#6309 build
  against, and the shared `pink/engagement-supervisor-roles` grants this issue adds.

## Runtime Flows And Context

- [architecture/large-asset-uploads.md](architecture/large-asset-uploads.md)
- [architecture/lambda-multi-handler-dispatch.md](architecture/lambda-multi-handler-dispatch.md)
  Why a handful of Lambda functions dispatch to more than one handler based
  on request URI/headers, and the criteria for when that exception applies.
- [architecture/arfcs.md](architecture/arfcs.md)
- [architecture/generalized-user-deletion.md](architecture/generalized-user-deletion.md)
  Why the `Users` row is tombstoned rather than deleted, the resumable
  candidates/delete worker, the external-resource-first (`SQL-2`) and
  new-resource purge-wiring (`SQL-3`) invariants, and the fail-closed boundaries
  where a natural foreign-key violation is the correct outcome.
- [architecture/team-purge.md](architecture/team-purge.md)
  The team-scoped purge and the three ways it deliberately differs from user
  deletion: synchronous because the cascade is bounded, a hard `Teams` delete
  because nothing holds attribution to `Teams`, and an explicit delete order
  because no foreign key into `Teams(id)` cascades. Also records how it widens
  `userPurge`'s `ChallengeResults` deletion set, and why that is accepted.
- [architecture/flightpath-inperson-2026-mode-switch.md](architecture/flightpath-inperson-2026-mode-switch.md)
  The FlightPath InPerson 2026 operator Preview/Live mode-switch procedures
  (#5944): the named transaction-ownership lint exception, the singleton
  `FOR UPDATE` serialization under `REPEATABLE READ`, the immutable baseline,
  repeat semantics, the Q4a privilege-narrowing promotion gate and its
  frontend dependency, the authorization-audit follow-ups, and the two-tier
  (Docker/Aurora-scratch) database-test approach. Also covers the #5983
  `FlightPathInPerson2026Eligible` eligibility role and the operator
  `internal_FlightPath2026InPersonAddUser`/`RemoveUser` entry points that gate
  Live enrollment, the #5995 operator team-management entry points
  (create/remove team, add/remove member, set/clear captain), and the #6024
  operator entry points that amend the Live window after the event has
  started and adopt a post-seed challenge into the competition.
- [architecture/competitive-gaming-standings.md](architecture/competitive-gaming-standings.md)
  Why game standings (#5950) filter withdrawn/disqualified teams at read time
  rather than blocking the underlying writes, the `shouldIncludeInactive`
  opt-in, and the documented-but-not-fixed `totalPoints`/`finalRank`
  non-maintenance that makes the season leaderboard's ranking
  nondeterministic.
- [architecture/ai-assessment-generation-contract.md](architecture/ai-assessment-generation-contract.md)
- [architecture/ai-lesson-planning-poc.md](architecture/ai-lesson-planning-poc.md)
- [architecture/lesson-authoring-and-delivery.md](architecture/lesson-authoring-and-delivery.md)
- [architecture/scope-sequence-and-objectives.md](architecture/scope-sequence-and-objectives.md)
  Course-level curriculum planning: the plan/unit/objective ownership
  hierarchy, objective provenance and retire-not-delete lifecycle, the
  cross-course join hazard through a shared standard node, and the
  push-stub vs. pull-aggregate cross-domain-signal convention.
- [architecture/course-helm-teaching-context.md](architecture/course-helm-teaching-context.md)
  The instructor-scoped teaching-context read: the consolidated
  `GET /v2/users/~/courses` contract, the `TermsWithCurrent` /
  `SectionsWithLifecycle` classification views and their lifecycle precedence,
  why `isCurrentTerm` is not a singleton, the `self`-scope grant rules, and the
  shared row-scanner hazards. System of record for #2862/#2863/#2864/#2865.
- [architecture/merlin-terminal-component.md](architecture/merlin-terminal-component.md)
  The merlin-terminal component and MyST live-terminal embeds: click-to-connect, host auth chain, directive transform/sentinel placeholders, and the mount-wrapper class contract.
- [architecture/merlin-code-editor-component.md](architecture/merlin-code-editor-component.md)
  The merlin-code-editor CodeMirror 6 component: the v1 root.ayodeComponent contract, session model, separator-provenance design and invariants, excluded auto-editing features, @scope styling, accessibility, the test-only hook, bundle pinning and delivery facts, canvas.js/WebKit harness lessons, known follow-ups, and the ownership split with #5884.
- [architecture/structured-choices-forms.md](architecture/structured-choices-forms.md)
  Version-2 structured choice pools for select-mode variants and the forms model: deterministic deals, formCount semantics, answer-security and grading invariants.
- [architecture/api-check-external-login-smoke-tests.md](architecture/api-check-external-login-smoke-tests.md)
  Why live Google browser-login api-check steps record non-aborting failures, and the secret-free artifact rules.
- [architecture/api-check-spec-loading.md](architecture/api-check-spec-loading.md)
  Why discovery/listing commands isolate one bad suite file instead of aborting, and why every such isolated failure must surface loudly (source-level log + rendered WARNING) rather than depending on each caller to remember.
- [architecture/api-check-deprecation-sunset-fixtures.md](architecture/api-check-deprecation-sunset-fixtures.md)
  Why a scheduled deprecation reaching sunset inside a shared, prepended api-check fixture surfaces as a 403 or 404 in an unrelated subsystem, the triage rule that follows, why what a step's 409 tolerance guards against must be re-derived from that procedure family rather than induced from a sibling, and the check-deprecation-migration gate that replaced hand-measured call-site inventories.
- [architecture/api-check-dag-cross-scenario-dependencies.md](architecture/api-check-dag-cross-scenario-dependencies.md)
  Why a cross-scenario read binds to the first writer and is hoisted to the scenario's entry node
  (an ordering-only `dataflow-hoist` edge), enforced by a pre-launch guard rail: a started scenario
  must never stall part-way and outlive a credential it minted early.
- [architecture/api-check-async-poll-budgets.md](architecture/api-check-async-poll-budgets.md)
  Why api-check async-poll budgets must be sized to the underlying architecture's documented worst case (e.g. Lambda retry/timeout ceilings), not to one observed completion time.
- [architecture/api-check-assertion-integrity.md](architecture/api-check-assertion-integrity.md)
  Why a green api-check run must mean the assertions actually ran: independent
  per-path matcher evaluation, substitution as a family-wide property, negative
  tests that pin known state, artifacts whose emptiness is meaningful, and a
  build that checks canonical form instead of rewriting committed source.
- [architecture/api-check-tooling-extensibility.md](architecture/api-check-tooling-extensibility.md)
  Patterns for extending the api-check harness itself: why a self-managed
  `io.Pipe()` on `cmd.Stdin` deadlocks `cmd.Wait()` (use `cmd.StdinPipe()`
  instead), why a new merge gate must be proven by replaying the historical
  regression it targets, why a tail-only fixture cleanup leaks against the
  per-user NFS quota on any earlier assertion failure, and why a step handler
  returning a raw error stops the whole run under either concurrent scheduler.
- [architecture/documentation-tooling-integrity.md](architecture/documentation-tooling-integrity.md)
  Sharp edges in the repository's Markdown-linting shell scripts: why a
  fence-toggle idiom must reject a second fence-character occurrence on the
  same line (else a CommonMark literal-fence-escape span is mistaken for a
  real fence), and the identical defect shape's other known occurrence.
- [architecture/api-check-restored-expectation-manifest.md](architecture/api-check-restored-expectation-manifest.md)
  The 211 assertions across 35 spec files that had never executed — the evidence
  base for that claim, why it cannot be regenerated from current source, and how
  cross-reading it exposed a second, independent defect.
- [architecture/api-check-credential-resolution.md](architecture/api-check-credential-resolution.md)
  Why assumed-role credential refresh belongs at the awslookup choke point rather
  than per call site, and why a component that falls back on a failed dependency
  must name its own failure instead of surfacing a misleading downstream error.
- [architecture/api-check-runner-context-shared-variables.md](architecture/api-check-runner-context-shared-variables.md)
  Why `Context.Variables` is shared by reference across every concurrently-cloned
  scenario (unlike the independently-cloned per-step `variables` map), why it must
  be written only through `SetVariable`, and why four review rounds were needed to
  find every call site — in-package and out — that broke this invariant with its
  own private, non-intersecting lock.
- [integrations/google-drive-integration.md](integrations/google-drive-integration.md)

## Next Stops

- Human/operator procedures: [../carbon-based-how/README.md](../carbon-based-how/README.md)
- Agent procedures: [../silicon-based-how/README.md](../silicon-based-how/README.md)

## Reading Strategy

1. Start with `architecture/architecture.md` for the overall model.
2. Read the subsystem document that matches the area you are changing.
3. Use `project-structure.md` when locating code or templates.
4. Switch to `carbon-based-how` or `silicon-based-how` only when you need to
   perform a task.
