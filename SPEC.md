# Witness Exporter Specification (v0)

**Status:** draft — first concrete profile (ZFS) forcing the contract shape
**Last updated:** 2026-04-17

> **A backend integration may supply observations, but only a conforming witness may supply testimony.**

This is the constitutional sentence of the spec. Everything below is a consequence.

## Purpose

Define what makes an exporter or helper **acceptable as an NQ witness**, not merely as a Prometheus endpoint — or a Nagios plugin, a Zabbix item, a Datadog API response, or any other monitoring backend NQ may grow to consume.

Prometheus exporter:

> emits samples

NQ witness:

> emits admissible operational evidence with declared coverage, freshness, and limits

The witness contract is a **semantic contract, not a transport shape**. Backends are disposable; the evidence object NQ consumes is not. Prom, Nagios, Zabbix, Datadog, and whatever arrives next may all be legitimate upstream *sources* or *substrates*, but NQ ingests one canonical witness report shape regardless of origin. Any adapter that cannot emit a conforming report is a source, not a witness — see the *Source conformance tiers* section for the formal lane.

This spec is the thing an implementation conforms to. Profiles (`profiles/*.md`) specify which fields each domain must expose to qualify as a witness for that domain.

## Core invariants

1. **Exporters provide visibility. Witnesses declare standing.**
2. **A witness must state what it can prove, what it can only suggest, and what it cannot see.**
3. **Privilege may increase visibility. It must not increase authority.**
4. **Witnesses produce evidence for detectors. They do not produce decisions.**
5. **Fail closed.** Silent witness is a finding. Missing data is not "all clear."
6. **Canonical report over backend-native shape.** Consumers ingest one witness document shape. Backend-native payloads (Prom scrapes, Nagios plugin output, Zabbix item values, Datadog metrics) are normalized *into* the canonical report by adapters; they are never treated as alternative first-class dialects.
7. **Coverage is declared, never inferred.** Consumers do not guess what a source "probably" covers. Absence of a `can_testify` tag is absence of coverage, full stop. Inferred capability is the failure mode this contract exists to prevent.

## Canonical report shape

The witness emits a structured report per collection cycle. Canonical format is JSON. Prometheus metrics, if emitted, are a projection of the same state — not the source of truth.

```json
{
  "schema": "nq.witness.v0",
  "witness": {
    "id": "zfs.local.lil-nas-x",
    "type": "zfs",
    "host": "lil-nas-x",
    "profile_version": "nq.witness.zfs.v0",
    "collection_mode": "sudo_helper",
    "privilege_model": "nopasswd_fixed_helper",
    "collected_at": "2026-04-16T22:30:00Z",
    "duration_ms": 143,
    "status": "ok"
  },
  "coverage": {
    "can_testify": [
      "pool_state",
      "vdev_state",
      "vdev_error_counters",
      "scrub_state"
    ],
    "cannot_testify": [
      "smart_drive_health",
      "enclosure_slot_mapping"
    ]
  },
  "standing": {
    "authoritative_for": [
      "current_pool_state",
      "current_vdev_error_counts"
    ],
    "advisory_for": [
      "likely_failure_regime"
    ],
    "inadmissible_for": [
      "authorization",
      "remediation"
    ]
  },
  "observations": [
    {
      "kind": "zfs_pool",
      "subject": "tank",
      "state": "DEGRADED"
    }
  ],
  "errors": []
}
```

## Section requirements

### 1. `schema`

Top-level schema identifier. Format: `nq.witness.v{N}` where `{N}` is the generic spec version. Consumers reject reports with an unrecognized schema.

### 2. `witness`

Identity and collection metadata.

**Required fields:**

| Field | Meaning |
|---|---|
| `id` | Stable witness identifier. Format: `{type}.{scope}.{host}` (e.g. `zfs.local.lil-nas-x`). Consumers key evidence by this ID across generations. |
| `type` | Short domain name (e.g. `zfs`, `sqlite`, `desktop_process`). Maps to a profile. |
| `host` | **Runner / emitter identity.** The host on which the witness process is executing and emitting the report. This is provenance about the witness itself, not about what it observes. |
| `profile_version` | Profile schema version (e.g. `nq.witness.zfs.v0`). |
| `collection_mode` | How the witness gathers data: `embedded`, `exporter`, `textfile`, `subprocess`, `sudo_helper`. `subprocess` is a one-shot invocation without privilege elevation (caller spawns the witness, witness emits a report and exits). `sudo_helper` is specifically sudo-mediated invocation (`sudo NOPASSWD` of a fixed helper path). The two are distinguished by privilege_model, which is orthogonal. |
| `privilege_model` | How privilege is structured: `unprivileged`, `root_exporter_localhost`, `nopasswd_fixed_helper`, `capability_bounded`. |
| `collected_at` | ISO-8601 UTC timestamp of the collection cycle start. |
| `duration_ms` | Collection cycle duration in milliseconds. |
| `status` | `ok`, `partial`, `failed`. |

**Conditionally required field:**

| Field | Meaning |
|---|---|
| `observed_subject` | Identifier of the subject the observations are about, when that subject is distinct from the runner host. |

**Normative rule (no silent aliasing):**

> A witness reporting on a subject other than its runner MUST include `witness.observed_subject`. Consumers MUST reject reports where the observations imply a different subject than `witness.host` but `witness.observed_subject` is absent.

Schema semantics (non-normative): when `observed_subject` is omitted, it is interpreted as equal to `witness.host`. Single-host deployments — a witness running locally on the machine it observes — satisfy this by omission. Federated and remote-observation cases must populate the field explicitly; no default-masking.

**Optional field (`backend` provenance block):**

| Field | Meaning |
|---|---|
| `backend.type` | Upstream backend this witness normalizes from (e.g. `prometheus`, `nagios`, `zabbix`, `datadog`, `native`). `native` means the witness reads the observed state directly without an intermediate backend. |
| `backend.adapter_version` | Version of the adapter that produced this canonical report from the backend's native shape. |
| `backend.source_ref` | Optional opaque reference to the upstream source query/check/item identifier, for audit and trace. |

Provenance is metadata, not authority. **Backend origin does not grant standing.** A report that claims `backend.type: "datadog"` does not thereby earn authoritative coverage of anything; standing comes only from `coverage.can_testify` + `standing` fields, regardless of backend.

### 3. `coverage`

Explicit declaration of what the witness can and cannot testify about. Required:

| Field | Meaning |
|---|---|
| `can_testify` | Array of profile-defined coverage tags the witness exposes with confidence. |
| `cannot_testify` | Array of coverage tags the witness does NOT expose (and would otherwise be expected in the profile). Explicit gaps matter more than implicit silence. |

Coverage tags are profile-defined strings. The ZFS profile defines `pool_state`, `vdev_state`, `vdev_error_counters`, `scrub_state`, `spare_state`, `dataset_capacity`, etc.

**Honest rule:** if a profile says `vdev_error_counters` is expected and the witness doesn't expose them, `vdev_error_counters` MUST appear in `cannot_testify`. Silence equals admission is the failure mode this section prevents.

### 4. `standing`

How the evidence may be used downstream. Required:

| Field | Meaning |
|---|---|
| `authoritative_for` | The witness is the canonical source of truth for these facts. |
| `advisory_for` | The witness suggests but does not prove these facts; consumers should reconcile with other sources. |
| `inadmissible_for` | The witness explicitly does NOT provide standing for these uses. `authorization` and `remediation` are always on this list — evidence is never authority. |

### 5. `observations`

Array of structured facts. Each observation is profile-defined but shares a common shape:

```json
{
  "kind": "<profile-defined-kind>",
  "subject": "<profile-defined-subject-identifier>",
  "...": "profile-defined fields"
}
```

Observations are evidence records, not alerts. They carry no severity, no action recommendation, no remediation hint. Those belong to NQ detectors consuming the observations.

### 6. `errors`

Array of collection errors. Non-empty when `status` is `partial` or `failed`. Each error has:

```json
{
  "kind": "<error-category>",
  "detail": "<human-readable short reason>",
  "observed_at": "<ISO-8601 UTC>"
}
```

A witness that fails to collect a specific coverage tag reports it via `errors` AND moves that tag from `can_testify` to `cannot_testify` for that cycle.

## Freshness and liveness

Every witness exposes its own health in the report. Consumers monitor witness freshness as a first-class signal:

- `witness.collected_at` — when this cycle ran
- `witness.duration_ms` — how long it took
- `witness.status` — ok / partial / failed
- `errors[]` — specific collection failures

A consumer treats a witness as **stale** when `collected_at` is older than a configurable threshold (profile-recommended; default 5 minutes for high-frequency profiles, longer for expensive collectors). A stale witness emits an `nq_witness_silent` finding on the consumer side.

## Failure behavior

All witness implementations must:

1. **Fail closed.** A witness that cannot collect must report `failed` with errors, not return stale or fabricated data.
2. **Bound timeouts.** No unbounded shell commands, no unbounded file reads, no unbounded network calls.
3. **Bound output.** Observations are capped at profile-defined maxima (e.g. top-N processes, max-N vdevs per pool). Unbounded enumeration is forbidden.
4. **Read-only.** No writes to observed state. No mutation commands. No side effects.
5. **No shell interpolation.** Fixed command sets. No operator-controlled or metric-controlled input reaching shell invocation.
6. **No remote exposure by default.** Localhost binding for exporter-mode; no network listeners for helper-mode.

**Partial collection is first-class.** A report with `status: "partial"` MUST satisfy all of:

- `errors[]` contains at least one entry naming the specific collection failure.
- Every coverage tag that failed to collect this cycle is demoted from `can_testify` to `cannot_testify` **in the same report**. Sticky coverage across cycles is forbidden — standing is recomputed per cycle from the current report's declarations.
- Observations whose coverage tag has been demoted are absent from `observations[]`. A partial report does not include half-observations or observations without their required coverage tag.
- `standing.authoritative_for` and `standing.advisory_for` are trimmed to reflect the reduced coverage. A witness cannot claim authoritative standing for a fact whose supporting coverage tag just moved to `cannot_testify`.

This rule prevents the ambiguous-half-success failure mode common to third-party monitoring backends (credential weirdness, scope mismatches, truncated payloads). Partial truth is representable; silent partial truth is not.

## Privilege model

Witnesses may run with elevated privilege when the observed state requires it (e.g. ZFS pool state is root-readable). The privilege grant must be:

- **Narrow** — authorized for exactly the commands or paths the witness needs, nothing broader.
- **Reviewable** — the operator can inspect the grant (sudoers line, systemd capability set, etc.).
- **Read-only** — no write commands within the grant.
- **Bounded** — not a general-purpose root shell.

Three acceptable privilege models for v0:

1. **`unprivileged`** — witness runs as an ordinary user and observes only state that user can read.
2. **`root_exporter_localhost`** — witness is a long-running daemon with necessary privileges, bound to localhost, exposing read-only metrics or JSON over HTTP.
3. **`nopasswd_fixed_helper`** — witness is a fixed-path script/binary invokable via `sudo NOPASSWD` with no arguments. Consumer shells to it, parses output, exits.

The invariant across all three:

> **Privilege is confined to observation. The consumer (NQ) remains unprivileged and never gains direct access to the privileged scope.**

## Output formats

### Canonical: JSON

The primary output is the canonical JSON report above. Consumers ingest it directly or via stored files, logs, or HTTP.

### Optional: Prometheus projection

Witnesses MAY additionally expose a Prometheus-format `/metrics` endpoint. When they do, the metrics are a projection of the canonical JSON state. Rules:

- Every exported metric corresponds to observations or collection metadata already present in the canonical report.
- The Prometheus output must not claim coverage not declared in `coverage.can_testify`.
- Metric names are profile-recommended but not profile-mandated; consumers should not depend on specific metric names when the JSON shape is available.
- `nq_witness_collection_duration_seconds`, `nq_witness_last_success_timestamp_seconds`, `nq_witness_status{status="ok|partial|failed"}`, and `nq_witness_schema_version` are standard across all witnesses.

**Prom is not the ontology.** A witness that emits only Prometheus metrics without a canonical JSON equivalent is a legacy exporter, not a witness. It may still be useful; it does not satisfy this spec.

## Profiles

This spec defines the generic contract. Concrete profiles live in `profiles/*.md` and specify:

- Which `coverage` tags are expected for the domain
- Which `observations.kind` values are required / optional
- Fields within each observation kind
- Severity/freshness defaults
- Example reports for happy-path, chronic-degraded, and failed-collection cases

**Profile rules (normative):**

1. **Controlled coverage-tag vocabulary.** Every profile MUST enumerate the complete set of coverage tags valid in `can_testify` / `cannot_testify` for that profile. Witnesses MUST NOT emit tags absent from the profile's vocabulary. Consumers MAY operate in *strict mode* (reject reports containing unknown tags) or *permissive mode* (log unknown tags and continue); production deployments SHOULD use strict mode. The default for new consumers is strict.

2. **Profiles describe evidence needs, not vendor mappings.** A profile specifies what observations exist, what coverage tags exist, and what facts may be testified to. It does not specify "Prom metric X means coverage tag Y" or "Datadog facet Z is observation kind W." Backend-specific recipes belong in adapter documentation (per-implementation), not in profile law. Profile spec is constitution; adapter docs are bylaws.

Currently shipped:

- `profiles/zfs.md` — ZFS pools, vdevs, scrubs, spares
- `profiles/smart.md` — drive-level SMART health complementing ZFS
- `profiles/fs_inode.md` — per-mount inode usage with explicit scope declaration

Future (not yet written):

- Desktop process / memory pressure profile — supports DESKTOP_FORENSICS_GAP in NQ
- SQLite / WAL witness profile — detects external-DB-reader pinning

## Source conformance tiers

NQ encounters three classes of upstream signal. The spec names them explicitly so the consumer's gating rules cannot equivocate.

### Tier 1: Conforming witness

A source that emits the canonical report shape from this spec, conforming to a published profile, with `coverage` and `standing` declared. **Domain-specific detectors fire only from Tier 1 sources.** Any detector that depends on the profile's coverage tags must verify them in the report's `can_testify` array before firing.

A Tier 1 source is the only thing that supplies *testimony* in the constitutional-sentence sense.

### Tier 2: Non-conforming observational source

A source that exposes domain-relevant signal in some structured form (Prometheus metrics, Nagios plugin output, Zabbix item values, Datadog API responses) but does not emit a canonical witness report or declare `coverage` / `standing`. NQ may consume Tier 2 sources through their native protocols, and observations from them flow into NQ's general-purpose stores (e.g. `metrics_history` for Prom scrapes). Tier 2 sources may feed *generic* detectors (threshold crossings, transition detection, presence/absence checks) but **may not feed profile-specific detectors** that require declared coverage.

A bare Prometheus exporter exposing `zfs_pool_health` without `coverage.can_testify` is the canonical Tier 2 example. Useful, ingested, but not testimony.

### Tier 3: Generic metrics source

A source providing arbitrary numeric / textual signal with no domain semantics whatsoever (a hand-rolled `node_exporter` textfile, a Pushgateway entry, a custom scraper). Same gating as Tier 2: feeds generic detectors only, never profile-specific ones. The distinction from Tier 2 is editorial — Tier 3 sources don't even pretend to expose domain shape.

### Tier promotion

A source can be promoted from Tier 2 or Tier 3 to Tier 1 by being wrapped in a witness adapter that:

- Reads the upstream signal in its native shape
- Normalizes into the canonical report
- Declares `coverage` honestly (only what the upstream actually provides, not what the profile asks for)
- Declares `backend.type` / `backend.adapter_version` / `backend.source_ref` so origin is auditable
- Submits to all the same fail-closed / bounded / read-only / no-shell-interpolation rules as a native witness

Wrapping a non-conforming source is legitimate work. It is the responsibility of the operator who chose that source — not the responsibility of NQ or the witness spec — to produce the wrapper. The spec defines what conformance looks like; it does not catalog backend-specific recipes for getting there.

## Relationship to NQ

NQ consumes witness reports via:

1. **Direct JSON ingestion** — NQ CLI or collector reads the report file or subprocess output.
2. **Prometheus scrape** — NQ's existing `prometheus_targets` configuration, when the witness exposes a Prom projection. Less rich than (1) because coverage/standing declarations don't survive the projection cleanly.

NQ's existing gaps consume this spec rather than defining their own exporter shapes:

- `ZFS_COLLECTOR_GAP` (in the NQ repo) references this spec for the ZFS witness contract. The gap specifies what NQ detectors do with the evidence; this spec specifies what the evidence must look like.
- Future `DESKTOP_FORENSICS` witness, SQLite witness, etc. use profile files in this repo.

## What this spec does NOT define

- The programming language of witness implementations
- The transport (subprocess stdout, file, HTTP, Unix socket are all fine)
- Specific metric names for Prometheus projections (profile-recommended, not spec-mandated)
- Authentication or signing of witness reports (out of scope for v0; federation/signing is a future concern)
- Retention or storage of historical reports (the consumer's job, not the witness's)
- Rate-limiting policy (profile-recommended defaults only)

## Versioning

- Generic spec: `nq.witness.v{N}`. v0 is this document.
- Profiles: `nq.witness.{type}.v{M}` where `{type}` is the profile domain. ZFS v0 is `nq.witness.zfs.v0`.
- Witness implementations declare both versions in `witness.profile_version` and the top-level `schema`.
- Consumers may require minimum versions; version negotiation is out of scope for v0 (pin versions in config for now).

## Failure-to-conform examples

### Example 1: bare Prometheus exporter (Tier 2 misclassified as witness)

A generic Prometheus ZFS exporter that exposes `zfs_pool_health` and capacity metrics but does NOT expose per-vdev error counts is **not a witness under this spec** because:

1. It does not declare coverage (`can_testify` / `cannot_testify` absent).
2. It does not declare standing (consumers cannot tell what the metrics may be used for).
3. It silently omits fields the ZFS profile requires, without admitting the gap.
4. It has no canonical JSON representation.

It can still be used as a Prometheus source by NQ (Tier 2) — just not as an authoritative ZFS witness. The ZFS profile's acceptance criteria are not satisfied by its output.

### Example 2: witness claiming inferred coverage

A witness that emits the canonical report shape, declares `coverage.can_testify: ["pool_state", "vdev_state", "vdev_error_counters"]`, but in fact only ran `zpool list` (no per-vdev inspection) is **not a conforming witness** because:

1. It claims coverage tags it cannot actually source. The vdev fields in `observations[]` would be fabricated, omitted, or guessed.
2. The structural compliance (correct JSON shape, correct schema string, correct profile_version) does not satisfy the substantive rule that coverage is *declared, never inferred*.
3. The witness is lying about what it observed, even if every individual JSON field is well-formed.

This is the second-most-dangerous failure mode after silent omission. A consumer that trusts coverage declarations cannot defend against a witness that fabricates them. The spec cannot enforce honesty at runtime, but it can name dishonesty as non-conformance — and conformance test suites for each profile MUST include checks that exercise the actual underlying commands the witness claims to run.

This is the intended distinction. Some exporters are witnesses. Most are not. Some witnesses are also not. The spec lets consumers know which.
