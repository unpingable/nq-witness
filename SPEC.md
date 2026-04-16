# Witness Exporter Specification (v0)

**Status:** draft — first concrete profile (ZFS) forcing the contract shape
**Last updated:** 2026-04-16

## Purpose

Define what makes an exporter or helper **acceptable as an NQ witness**, not merely as a Prometheus endpoint.

Prometheus exporter:

> emits samples

NQ witness:

> emits admissible operational evidence with declared coverage, freshness, and limits

This spec is the thing an implementation conforms to. Profiles (`profiles/*.md`) specify which fields each domain must expose to qualify as a witness for that domain.

## Core invariants

1. **Exporters provide visibility. Witnesses declare standing.**
2. **A witness must state what it can prove, what it can only suggest, and what it cannot see.**
3. **Privilege may increase visibility. It must not increase authority.**
4. **Witnesses produce evidence for detectors. They do not produce decisions.**
5. **Fail closed.** Silent witness is a finding. Missing data is not "all clear."

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

Identity and collection metadata. Required fields:

| Field | Meaning |
|---|---|
| `id` | Stable witness identifier. Format: `{type}.{scope}.{host}` (e.g. `zfs.local.lil-nas-x`). Consumers key evidence by this ID across generations. |
| `type` | Short domain name (e.g. `zfs`, `sqlite`, `desktop_process`). Maps to a profile. |
| `host` | Hostname the witness is running on / observing. |
| `profile_version` | Profile schema version (e.g. `nq.witness.zfs.v0`). |
| `collection_mode` | How the witness gathers data: `embedded`, `exporter`, `textfile`, `sudo_helper`. |
| `privilege_model` | How privilege is structured: `unprivileged`, `root_exporter_localhost`, `nopasswd_fixed_helper`, `capability_bounded`. |
| `collected_at` | ISO-8601 UTC timestamp of the collection cycle start. |
| `duration_ms` | Collection cycle duration in milliseconds. |
| `status` | `ok`, `partial`, `failed`. |

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

Currently shipped:

- `profiles/zfs.md` — ZFS pools, vdevs, scrubs, spares

Future (not yet written):

- Desktop process / memory pressure profile — supports DESKTOP_FORENSICS_GAP in NQ
- SQLite / WAL witness profile — detects external-DB-reader pinning
- SMART profile — drive-level health complementing ZFS

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

A generic Prometheus ZFS exporter that exposes `zfs_pool_health` and capacity metrics but does NOT expose per-vdev error counts is **not a witness under this spec** because:

1. It does not declare coverage (`can_testify` / `cannot_testify` absent).
2. It does not declare standing (consumers cannot tell what the metrics may be used for).
3. It silently omits fields the ZFS profile requires, without admitting the gap.
4. It has no canonical JSON representation.

It can still be used as a Prometheus source by NQ — just not as an authoritative ZFS witness. The ZFS profile's acceptance criteria are not satisfied by its output.

This is the intended distinction. Some exporters are witnesses. Most are not. The spec lets consumers know which.
