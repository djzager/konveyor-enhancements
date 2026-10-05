---
title: agent-usage-accounting
authors:
  - "@djzager"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-09-24
last-updated: 2026-09-24
status: provisional
see-also:
  - "/enhancements/agentic-platform-controller/README.md"
  - "/enhancements/llm-proxy/README.md"
  - "/enhancements/kai/agent-driven-migration/README.md"
replaces: []
superseded-by: []
---

# Agent Usage Accounting

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined and reviewed
- [ ] User-facing documentation is created
- [ ] Storage, authentication, and deployment design is validated
- [ ] Release owners approve any 0.11 backport

## Summary

Collect detailed runtime-reported token consumption and activity metadata from
agent workloads using OpenTelemetry. Retain this information in an accounting
service with a query API, independently of pod and AgentRun cleanup. Allow
customers to configure additional destinations for asynchronous export. Konveyor's
UI accesses accounting through Hub; other authorized clients use the accounting
API directly.

Accounting is best effort: delivery problems are visible independently of the
agent's execution result and do not fail otherwise successful work or block
workflow progression. Successful delivery means durable acceptance of the
records collected, not proof of complete provider billing coverage.

Target accounting and UI integration for 0.12, with accounting enabled by
default and an option to disable it. Pursue a complete, opt-in reporting and
storage backport to 0.11, disabled by default, subject to community acceptance
and release readiness. Token-budget enforcement for runaway-run containment
is a 0.12 feature and is independent of accounting-service availability.

This proposal records an agreed product direction. The backend, precise wire
schema, deployment topology, and final delivery-verification protocol remain
open. It is not yet an implementation-ready specification.

## Motivation

Developers and administrators need evidence of consumption across agent runs,
workflow stages, models, and Gateways. Live chat counters alone cannot support
historical comparisons after a browser disconnects or a workload is removed.
They also do not explain which tools and file operations a runtime reported.

Customers may use shared provider credentials, Vertex, customer-managed
gateways, or a future OpenShell execution path. Accounting must retain run
attribution without requiring a provider API key per run or control of every
upstream routing layer. Consumers should be able to apply their own pricing
to the recorded consumption through an API.

### Goals

- Retain granular runtime-reported token usage and activity metadata with
  explicit provenance and coverage limitations.
- Attribute observations to AgentRuns, workflow runs/stages, sessions, and
  Gateways, preserving useful historical identity after CR deletion.
- Use OpenTelemetry SDKs and OTLP for reporting, and reuse ecosystem components
  where they meet the accounting requirements.
- Provide a durable, independently usable accounting API and optional exports.
- Provide run/workflow detail and an aggregate usage overview in Konveyor.
- Support local, observed-token limits for runaway-run containment in 0.12.
- Follow the principle of least surprise: supported limits are enforced by
  the harness without making users understand provider-adapter differences.

### Non-Goals

- Monetary calculation, rate catalogs, invoice reconciliation, and chargeback.
- Complete visibility into hidden upstream retries or every billable token.
- Initial capture of full prompts, responses, shell command text, arbitrary
  tool arguments/results, stdout/stderr, or file contents.
- OS-level auditing of every subprocess or filesystem access.
- Workflow-wide budget allocation, shared team/Gateway quotas, or periodic
  allowances; add such policies when users request them.
- A strict token ceiling enforced through pre-request reservations.
- New token-budget stopping behavior in the 0.11 backport.
- Instrumenting the separate existing DLS/IDE execution path.

## Proposal

### User Stories

1. A developer opens a completed migration and sees token consumption by stage,
   model, and Gateway, plus a timeline of reported activity, even after its
   AgentRun and Sandbox have been cleaned up.
2. An administrator compares consumption across runs over a selected period
   and drills into the runs responsible for it.
3. A customer queries consumption through the API and overlays its own model
   rates, agreements, and allocation policies.
4. An operator configures external telemetry destinations while retaining the
   service's own queryable history. An external outage does not disrupt runs.
5. A user configures a supported per-run token limit. The harness winds down
   and stops based on locally observed consumption, including when accounting
   storage is disabled or temporarily unavailable.

### Release Scope

| Capability | Candidate 0.11 backport | 0.12 target |
| --- | --- | --- |
| Harness OTel usage/activity reporting | Included in a complete opt-in feature | Included |
| Accounting storage and independent API | Disabled by default | Enabled by default; can be disabled |
| Configured asynchronous export | Part of the proposed accounting feature | Included |
| Accounting delivery policy | Best effort when enabled | Best effort |
| Konveyor accounting UI | Not required for candidate backport | Run/workflow detail and usage overview |
| Runaway-run token containment | Excluded | Included for configured supported limits |
| Monetary calculation | Excluded | Excluded |

Default-enabled accounting does not imply an automatically assigned token
budget. Budget defaults and configuration precedence need a separate decision.
The 0.11 backport is not a promise: it must be a complete supported feature
within its declared scope, with installation and upgrade validation. Disabling
it by default does not eliminate storage, packaging, or compatibility risk.

### Responsibilities and Data Flow

```text
Runtime-reported observations
          |
          v
Harness adapter -----------------> local budget enforcement (0.12)
          |
          | OpenTelemetry / OTLP
          v
Accounting ingestion --> durable storage --> accounting query API
          |                                        ^
          v                                        |
Configured external destinations             Hub <-- tackle2-ui
                                                   |
                              Independent clients use the API directly
```

The accounting service is a logical service boundary, not a decision to build
a custom telemetry database. It may comprise a standard Collector, a supported
storage backend, and an accounting query layer. Alternatively, a smaller
service may implement OTLP ingestion and use a relational store. Compare these
before choosing a deployment.

The controller remains a domain-agnostic, stateless reconciler. It supplies
execution identity/configuration and observes CR status. It does not acquire
ACP connections or receive all telemetry inside its reconciliation loop.
Local budget enforcement does not require a controller-hosted OTel receiver.
Whether a controller-managed stop backstop is needed remains open.

The harness owns runtime adaptation. Goose is the first integration. A
versioned reporting contract must allow other harness implementations to
report observations without depending on Konveyor application concepts.
The accounting backend must not need to parse Goose session databases.

### Observation Fidelity and Identity

Retain individual reported usage observations where available; explicitly
identify cumulative snapshots where those are the available source. A
runtime ledger row is not automatically equivalent to one provider request.

The contract should carry:

- A stable observation identity, source/schema version, and observation time.
- AgentRun identity and execution attempt, workflow/stage association when
  applicable, runtime session identity, and subagent relationships when known.
- Gateway identity and relevant historical configuration attribution, plus
  requested and response-reported model identity when available.
- Reported input, output, cache, reasoning, and other supported usage fields,
  with semantics sufficient to avoid counting overlapping categories twice.
- Activity IDs, tool identity, reported lifecycle/outcome, and supported
  operation/path metadata. Distinguish observed event timing from an exact
  execution duration if the runtime does not report the latter.
- Provenance and explicit unknown, partial, or unsupported measurements.

Normalization must avoid adding repeated cumulative snapshots, adding child
usage already included in parent totals, or treating absent values as zero.
Stable IDs must prevent report retries and final reconciliation from counting
the same observation again. A retried model request is a different event from
a retried delivery of the same accounting record.

Initial activity capture is metadata only. The code and reporting contract
should permit richer configurable capture later without requiring it now.
A reported shell invocation does not establish every file its process tree
accessed. Tool activity also does not establish an exact token cost per file
or shell command.

Gateway is a first-class accounting dimension. Retained observations must
remain interpretable if a Gateway is updated, deleted, or recreated. Record
what is known about configured and reported models without claiming that an
alias reveals every downstream routing decision. Future pricing metadata can
enrich Gateway-based reporting; current consumers apply their own pricing.

### OpenTelemetry and Delivery

Use an OTel SDK to report records continuously. The specific choice of logs,
spans, and derived metrics, and the versioned attribute contract, must be
validated before implementation. Where applicable, use OTel semantic
conventions and version extensions needed for accounting.

Accounting records must not be sampled away as diagnostic traces might be.
SDK queue limits, export failures, and partial rejection must be observable.
Prometheus-compatible aggregate metrics can describe consumption and pipeline
health but are not the authoritative per-run history.

Before an orderly harness exit, collect remaining observations and perform a
bounded final delivery/reconciliation attempt. This is a completion step,
not a barrier after every ACP prompt or model call. Mark delivery successful
only when collected records have been durably accepted into the accounting
store or a service-owned persistent queue with a defined recovery path.
Customer destinations need not have received their copies at that point.

An ordinary successful SDK flush or OTLP response does not by itself establish
this end-to-end guarantee. Validate the selected pipeline's acknowledgement
behavior and define any required final receipt/manifest check. A terminal
marker alone cannot prove earlier SDK records were not dropped.

In best-effort mode, unsuccessful delivery does not fail successful agent work
or block subsequent workflow stages. Represent accounting delivery separately
from execution outcome, including disabled, pending, delivered, and failed or
incomplete cases. Exact Kubernetes condition mapping and how later delivery
updates retained status remain to be designed. Delivery success is distinct
from completeness of provider billing coverage.

Runtime session stores may support recovery. Goose's usage ledger can recover
observations missed by live delivery, but it cannot recover usage the runtime
never received. A sidecar or final cleanup handler cannot guarantee execution
after node loss or forced termination. Initial collection must work with the
existing Sandbox deployment; adding sidecars is not an assumed prerequisite.

### Retention and External Export

Retain usage and activity records for 30 days by default, configurable by the
operator. Initially apply one retention policy rather than introducing
separate long-lived aggregate retention. Retention is independent of pod,
AgentRun, and workflow CR cleanup.

The service stores records and asynchronously sends copies to configured
destinations. Destination outages must not prevent local ingestion or queries.
Retry limits, storage bounds, export health, and treatment of export backlog
at retention expiry must be specified. Disabling accounting must have an
explicit, documented policy for retained data rather than implicitly deleting
its volume. External copies are subject to the destination's own policies.

### API and UI Access

The API supports run/workflow detail, activity history, and consumption over
time grouped by workflow, model, or Gateway. Include filtering, pagination,
and documented measurement semantics; exact endpoints remain open. Queries
must remain meaningful after execution resources are deleted.

Store API credentials in namespace Secrets, with Kubernetes RBAC controlling
access to those Secrets. The accounting service separately authenticates API
requests and enforces credential scope. Secret access is not automatic
authorization for arbitrary API queries.

Konveyor UI requests go through Hub. Hub keeps accounting credentials
server-side and applies Konveyor user authorization. The accounting API remains
available to independent authenticated clients without requiring Hub.
Separate ingestion and read credentials are recommended; exact scopes,
rotation, revocation, and trusted attribution require design review.

The 0.12 UI has two views:

- Run/workflow detail: token breakdowns by model/Gateway and stage, delivery
  status, and a runtime-reported activity timeline.
- Usage overview: historical comparisons across runs, with drill-down links.

Implement run/workflow detail first. Retained detail can be available through
the API before every field has a dedicated UI presentation. Do not display
missing usage as zero or missing history as evidence that no work occurred.

### Runaway-Run Containment in 0.12

The MVP use case is containing one runaway AgentRun. The harness enforces a
configured supported token-consumption limit using local observations,
independently of OTel export and accounting-service availability. Reserve
headroom for wind-down/handoff, stop based on observed consumption, and report
a clear limit-reached outcome while preserving work through the harness's
normal git lifecycle.

Usage may arrive after a model request has consumed tokens, and requests may
already be in flight. Record observed overshoot; do not promise a strict
billing ceiling. Supported limit definitions, adapter normalization, and
measurement-conformance tests are implementation responsibilities. Preserve
granular observations, but do not introduce arbitrary category controls
without a demonstrated use case.

Determine the MVP limit definition, configuration precedence, headroom policy,
and runtime stop escalation before implementation. Resolve missing-measurement
behavior through that design rather than silently treating unknown usage as
zero. Budget enforcement does not make a dishonest or non-reporting runtime
an authoritative source of provider billing.

### Relationship to Existing Enhancements

The agentic platform proposal describes git as the persistence layer for
workspaces and handoff information. This enhancement introduces a separate
durable category: queryable usage/activity observations. It does not move
workspace persistence out of git or require full session transcripts there.

The LLM proxy proposal excludes initial metering. This enhancement observes
supported runtime reports and coexists with configured/customer gateways; it
does not require routing every inference request through a new proxy.

OpenShell adoption must preserve the reporting contract, but no specific
`inference.local` topology or SDK billing capability is assumed. Released and
development OpenShell routing designs differ and must be validated against
the version actually selected for integration.

### Security, Risks, and Mitigations

- Bind accepted execution identity to authenticated producers; do not trust
  arbitrary namespace/run attributes for authorization or cross-tenant writes.
- Scope query access, protect service credentials, and avoid distributing a
  shared read credential to browsers or agent runtimes.
- Allowlist metadata. File paths and free-form error/title fields can still
  contain customer information; review extraction/redaction and export rules.
- Use authenticated, encrypted transport and Secret references for export
  credentials. Review credential rotation and revocation semantics.
- Bound ingestion, buffers, queries, and retained storage. Expose overload and
  delivery gaps instead of claiming complete accounting after data loss.
- Preserve ownership/authorization metadata through CR cleanup so deletion
  does not change who can query retained observations.
- Obtain security and UX review from the relevant component maintainers;
  reviewers are not assigned yet.

## Open Questions

1. Select a backend and deployment topology, including the initial availability
   target, volume sizing, backup/recovery, and supported disconnected images.
2. Define the OTel signal/schema contract, stable record identity, cumulative
   snapshot handling, and final durable-receipt verification.
3. Define ingestion/read credential scopes, provisioning/rotation, and retained
   authorization context, including the Hub integration.
4. Specify accounting conditions and ownership of late status updates without
   making controller reconciliation depend on a live telemetry stream.
5. Define bounded export/retry policy, shutdown timeout, overload behavior,
   replay storage, and interactions with retention and feature disablement.
6. Define the 0.12 supported budget limit, precedence, headroom, and escalation;
   determine whether a controller-managed stop backstop is necessary.
7. Validate the Goose/provider support matrix, including streaming, compaction,
   subagents, errors, and the OpenShell version actually deployed.
8. Confirm the complete 0.11 backport footprint and release-owner acceptance.

## Design Details

### Test Plan

- Use captured, sanitized fixtures from the pinned Goose/runtime and supported
  provider adapters; compare recorded consumption with provider response
  metadata. Do not rely solely on synthetic event shapes.
- Cover cached input, output/reasoning overlap, repeated prompts, compaction,
  subagents, retries, missing usage, and cumulative snapshot replacement.
- Exercise duplicate delivery, replay, out-of-order events, partial OTLP
  rejection, SDK overflow, collector/service restart, and acknowledgement loss.
  Assert no duplicate accounting and honest delivery status.
- Inject storage outages and full disks. Test durable-queue recovery and prove
  what a successful acknowledgement guarantees in the selected deployment.
- Test final flush, runtime errors, cancellation, forced pod termination, and
  resource cleanup. Do not expect cleanup hooks to run after every failure.
- Test retention, Gateway changes/deletion, AgentRun deletion, and pagination
  without losing interpretation or authorization of retained records.
- Test authenticated ingest/query separation, forged attribution, scope
  isolation, credential rotation, and Hub user authorization.
- Verify metadata-only payloads exclude raw tool contents and secrets, including
  when exporting to a configured destination.
- In 0.12, test wind-down, handoff accounting, cancellation, observed overshoot,
  and containment with accounting disabled or unreachable.
- Verify UI history without a live pod, accounting failure alongside successful
  execution, aggregate/detail consistency, and unknown-versus-zero rendering.
- Validate enabled/disabled installs, upgrade/downgrade, data preservation,
  disconnected operation, and representative event-volume performance.

### Upgrade / Downgrade Strategy

The proposed 0.11 backport is disabled by default and must preserve current
execution behavior when disabled. Enabling it provisions the complete required
accounting deployment and storage. For 0.12, default enablement adds storage and
resource requirements that must be documented before release.

Version record and API schemas. Define supported mixed-version behavior for
harnesses, controller, Hub, and accounting components. An older harness without
accounting support must be identified as unsupported/unreported, not as zero
consumption. Define database migration and rollback constraints for the chosen
backend. Do not silently remove retained data on disablement or downgrade.

### Implementation History

- 2026-09-23: Investigated runtime usage, persistence, routing visibility,
  OpenTelemetry delivery, and accounting/UI requirements.
- 2026-09-24: Created this provisional proposal from the design discussion;
  included release boundaries, metadata scope, retention, local containment,
  and Hub-mediated access. No feature implementation or release commitment.

## Drawbacks

Default-enabled accounting adds storage, operational ownership, and schema
maintenance. Activity metadata can be much larger than token totals. A
standards-based telemetry pipeline still needs accounting-specific identity,
deduplication, interpretation, and delivery verification. Token containment
depends on runtime measurement coverage and cannot remove all overshoot.

## Alternatives

### Standard Collector and Telemetry Backend

Reuse an OTel Collector and an existing telemetry database, with a thin
accounting API. This provides ecosystem integration and query capabilities,
but adds packaging/operational dependencies and still requires accounting
deduplication and receipt semantics. Validate an actual supported combination
before selecting it; Konveyor currently has no general telemetry history store
established by this investigation.

### OTLP-Compatible Service and Relational Storage

Implement OTLP ingestion in an accounting service using a relational database,
with transactional uniqueness and asynchronous export. This can reduce the
deployment footprint and make durable acceptance explicit, but increases our
ownership of ingestion, retention, export, and query implementation. Compare
this against ecosystem reuse rather than assuming a custom store is required.

### Accounting Inside Hub or the Controller

Hub-only persistence reduces Konveyor-specific deployment overhead but does
not satisfy independent consumers without additional extraction. Hosting
telemetry/storage in the controller couples reconciliation to a new workload;
local harness containment does not require that coupling. Keep the accounting
service independently deployable.

### Prometheus or Termination Records Alone

Prometheus summaries help with operational monitoring but do not provide the
required detailed event history and deduplication contract. AgentRun termination
records are bounded and disappear with CR cleanup. Neither alone meets this
proposal's retention and investigation requirements.

## Infrastructure Needed

Accounting deployment/storage packaging, supported images, API/OTLP fixtures,
provider integration testing, and operator upgrade/recovery tests. Repository
ownership and any new image publication requirements remain to be assigned.

## References

- [OpenTelemetry overview](https://opentelemetry.io/docs/what-is-opentelemetry/):
  standard instrumentation/transport; storage and visualization are separate.
- [OTLP specification](https://opentelemetry.io/docs/specs/otlp/):
  acknowledgements are hop-by-hop, not an end-to-end persistence contract.
- [Collector resiliency](https://opentelemetry.io/docs/collector/resiliency/):
  persistent queues, retries, and remaining loss conditions.
- [Goose v1.45.0 session accounting](https://github.com/aaif-goose/goose/blob/v1.45.0/crates/goose/src/session/session_manager.rs):
  runtime ledger and cumulative session measurements.
- [Goose v1.45.0 tool events](https://github.com/aaif-goose/goose/blob/v1.45.0/crates/goose/src/acp/server/tool_calls/conversion.rs):
  available runtime activity metadata.
- [Vertex response usage](https://cloud.google.com/vertex-ai/generative-ai/docs/reference/rest/v1/GenerateContentResponse):
  provider-reported token categories.
- [Kubernetes sidecar lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/):
  termination order is not a guarantee that final collection always runs.
