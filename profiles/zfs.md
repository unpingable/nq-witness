# ZFS Witness Profile (v0)

**Profile version:** `nq.witness.zfs.v0`
**Applies to:** OpenZFS (Linux/ZoL, FreeBSD). Illumos is compatible in principle; not tested.
**Forcing case:** chronic-degraded pool — one faulted drive with spares assigned, pool otherwise stable. The witness must make stable-vs-worsening distinguishable.

## Required `coverage.can_testify` tags

A conforming ZFS witness MUST testify about:

- `pool_state` — current state string for each pool
- `pool_capacity` — bytes allocated / free / total per pool
- `vdev_state` — state string per vdev within each pool
- `vdev_error_counters` — read / write / checksum error counts per vdev
- `scrub_state` — current scan state (scrub, resilver, none) per pool
- `scrub_completion` — last scrub timestamp and result per pool
- `spare_state` — spare assignment and activation per pool

A witness missing any of these MUST list them in `cannot_testify` rather than emitting partial data or silence.

## Optional `coverage.can_testify` tags

- `dataset_capacity` — per-dataset used/available/quota/referenced
- `dataset_properties` — compression, dedup, encryption status per dataset
- `scrub_progress` — in-flight scrub percentage and bytes examined
- `fragmentation` — pool fragmentation ratio
- `readonly_flag` — pool-level readonly status

## Required observation kinds

### `zfs_pool`

One observation per pool. Required fields:

```json
{
  "kind": "zfs_pool",
  "subject": "tank",
  "state": "DEGRADED",
  "health_numeric": 6,
  "size_bytes": 79989470920704,
  "alloc_bytes": 8277407145984,
  "free_bytes": 71712063774720,
  "readonly": false,
  "fragmentation_ratio": 0.0
}
```

- `subject` — pool name, stable across generations
- `state` — one of `ONLINE`, `DEGRADED`, `FAULTED`, `OFFLINE`, `UNAVAIL`, `REMOVED`, `SUSPENDED`
- `health_numeric` — integer form if the underlying source exposes one (for compatibility with sparse exporters that emit only `zfs_pool_health` as an integer); consumers MAY derive `state` from this if the string isn't available, but the witness SHOULD provide both
- `size_bytes` / `alloc_bytes` / `free_bytes` — pool capacity in bytes
- `readonly` — boolean
- `fragmentation_ratio` — 0.0 to 1.0, optional if not available

### `zfs_vdev`

One observation per vdev (physical device or mirror/raidz group leaf). Required fields:

```json
{
  "kind": "zfs_vdev",
  "subject": "tank/raidz2-0/ata-WDC-WD100EMAZ-0...",
  "pool": "tank",
  "vdev_path": "/dev/disk/by-id/ata-WDC-WD100EMAZ-0...",
  "vdev_guid": "1234567890123456",
  "vdev_type": "disk",
  "state": "FAULTED",
  "read_errors": 0,
  "write_errors": 0,
  "checksum_errors": 12,
  "is_spare": false,
  "is_replacing": false
}
```

- `subject` — stable vdev identifier. Prefer `{pool}/{parent}/{device}` format for human readability; `vdev_guid` is the ZFS-authoritative identifier.
- `pool` — containing pool name
- `vdev_path` — filesystem path to the device (when applicable)
- `vdev_guid` — ZFS internal vdev GUID
- `vdev_type` — `disk`, `mirror`, `raidz`, `raidz2`, `raidz3`, `spare`, `log`, `cache`
- `state` — one of `ONLINE`, `DEGRADED`, `FAULTED`, `OFFLINE`, `UNAVAIL`, `REMOVED`
- `read_errors` / `write_errors` / `checksum_errors` — cumulative error counters since last `zpool clear`
- `is_spare` — this vdev is a configured spare
- `is_replacing` — this vdev is actively replacing a failed device

**This is the section pdf/zfs_exporter and similar basic exporters fail to provide.** A witness without vdev-level detail cannot satisfy the chronic-degraded forensic case this profile exists for.

### `zfs_scan`

One observation per pool reporting current or last scan (scrub / resilver) state. Required fields:

```json
{
  "kind": "zfs_scan",
  "subject": "tank",
  "pool": "tank",
  "scan_type": "scrub",
  "scan_state": "completed",
  "last_completed_at": "2026-04-12T03:14:00Z",
  "errors_found": 0,
  "repaired_bytes": 0,
  "examined_bytes": 42000000000000,
  "total_bytes": 52000000000000,
  "progress_pct": 100.0
}
```

- `scan_type` — `scrub`, `resilver`, or `none` (if no scan has ever been run or no history available)
- `scan_state` — `in_progress`, `completed`, `canceled`, `none`
- `last_completed_at` — ISO-8601 UTC of most recent completed scan, or null if none
- `errors_found` — errors discovered during last completed scan
- `repaired_bytes` — bytes repaired during last scan
- `examined_bytes` / `total_bytes` — progress metrics (both populated during in-progress scans)
- `progress_pct` — 0.0 to 100.0

### `zfs_spare`

One observation per configured spare. Required fields:

```json
{
  "kind": "zfs_spare",
  "subject": "tank/spare/ata-WDC-WD100EMAZ-1...",
  "pool": "tank",
  "spare_path": "/dev/disk/by-id/ata-WDC-WD100EMAZ-1...",
  "spare_guid": "2345678901234567",
  "state": "AVAIL",
  "is_active": false,
  "replacing_vdev_guid": null
}
```

- `state` — `AVAIL`, `INUSE`, `UNAVAIL`
- `is_active` — spare has been assigned to replace a failed vdev
- `replacing_vdev_guid` — GUID of the vdev this spare is replacing, when active

## Standing defaults for ZFS witnesses

```json
{
  "authoritative_for": [
    "current_pool_state",
    "current_vdev_state",
    "current_vdev_error_counts",
    "last_scrub_completion",
    "spare_assignment"
  ],
  "advisory_for": [
    "chronic_vs_worsening_regime_classification"
  ],
  "inadmissible_for": [
    "drive_smart_health",
    "authorization",
    "remediation"
  ]
}
```

The `chronic_vs_worsening_regime_classification` being advisory (not authoritative) matters: the witness reports facts. The regime classification is NQ's job, using multiple generations of witness evidence plus its own stability/trajectory/recovery features. A single witness snapshot can't testify that a condition is "stable" — that requires cross-generation analysis the witness doesn't do.

## Freshness defaults

- **Expected collection cadence:** every 60 seconds for a typical NQ deployment. ZFS state doesn't change fast; faster collection wastes cycles.
- **Stale threshold:** 5 minutes without a fresh report. Beyond this, the consumer emits `nq_witness_silent` for the ZFS witness.
- **Collection timeout:** 5 seconds. `zpool status` can briefly block on slow devices; a wider window increases the chance of a stuck helper.

## Privilege model recommendations

ZFS state reads (`zpool status`, `zpool list`, `zfs list`) are root-restricted on most Linux distributions. Acceptable models:

1. **`nopasswd_fixed_helper`** (recommended for minimal privilege surface) — a root-owned helper at a known path, invokable via `sudo` with `NOPASSWD` for exactly that path, no arguments. Emits the canonical JSON report to stdout.

2. **`root_exporter_localhost`** — a long-running exporter daemon with sufficient privileges, bound to `127.0.0.1`, exposing the canonical JSON at `/report` or equivalent. May also expose Prometheus projection at `/metrics`.

3. **`unprivileged`** — acceptable only if the deployment has specifically granted non-root access to `zpool`/`zfs` binaries (rare and typically requires custom setup). Must still declare coverage honestly.

## Example reports

### Happy path — healthy pool, recent scrub

```json
{
  "schema": "nq.witness.v0",
  "witness": {
    "id": "zfs.local.example-host",
    "type": "zfs",
    "host": "example-host",
    "profile_version": "nq.witness.zfs.v0",
    "collection_mode": "sudo_helper",
    "privilege_model": "nopasswd_fixed_helper",
    "collected_at": "2026-04-16T22:30:00Z",
    "duration_ms": 112,
    "status": "ok"
  },
  "coverage": {
    "can_testify": ["pool_state", "pool_capacity", "vdev_state", "vdev_error_counters", "scrub_state", "scrub_completion", "spare_state"],
    "cannot_testify": ["smart_drive_health", "enclosure_slot_mapping"]
  },
  "standing": {
    "authoritative_for": ["current_pool_state", "current_vdev_state", "current_vdev_error_counts", "last_scrub_completion", "spare_assignment"],
    "advisory_for": ["chronic_vs_worsening_regime_classification"],
    "inadmissible_for": ["drive_smart_health", "authorization", "remediation"]
  },
  "observations": [
    {"kind": "zfs_pool", "subject": "tank", "state": "ONLINE", "size_bytes": 10000000000000, "alloc_bytes": 2000000000000, "free_bytes": 8000000000000, "readonly": false, "fragmentation_ratio": 0.05},
    {"kind": "zfs_vdev", "subject": "tank/raidz2-0/disk-a", "pool": "tank", "vdev_type": "disk", "state": "ONLINE", "read_errors": 0, "write_errors": 0, "checksum_errors": 0, "is_spare": false, "is_replacing": false},
    {"kind": "zfs_scan", "subject": "tank", "pool": "tank", "scan_type": "scrub", "scan_state": "completed", "last_completed_at": "2026-04-12T03:14:00Z", "errors_found": 0, "repaired_bytes": 0}
  ],
  "errors": []
}
```

### Chronic-degraded — one faulted drive, spare assigned, pool stable

```json
{
  "schema": "nq.witness.v0",
  "witness": {
    "id": "zfs.local.example-host",
    "type": "zfs",
    "host": "example-host",
    "profile_version": "nq.witness.zfs.v0",
    "collection_mode": "sudo_helper",
    "privilege_model": "nopasswd_fixed_helper",
    "collected_at": "2026-04-16T22:30:00Z",
    "duration_ms": 118,
    "status": "ok"
  },
  "coverage": {
    "can_testify": ["pool_state", "pool_capacity", "vdev_state", "vdev_error_counters", "scrub_state", "scrub_completion", "spare_state"],
    "cannot_testify": ["smart_drive_health", "enclosure_slot_mapping"]
  },
  "standing": {
    "authoritative_for": ["current_pool_state", "current_vdev_state", "current_vdev_error_counts", "last_scrub_completion", "spare_assignment"],
    "advisory_for": ["chronic_vs_worsening_regime_classification"],
    "inadmissible_for": ["drive_smart_health", "authorization", "remediation"]
  },
  "observations": [
    {"kind": "zfs_pool", "subject": "tank", "state": "DEGRADED", "size_bytes": 10000000000000, "alloc_bytes": 2000000000000, "free_bytes": 8000000000000, "readonly": false, "fragmentation_ratio": 0.05},
    {"kind": "zfs_vdev", "subject": "tank/raidz2-0/disk-a", "pool": "tank", "vdev_type": "disk", "state": "ONLINE", "read_errors": 0, "write_errors": 0, "checksum_errors": 0, "is_spare": false, "is_replacing": false},
    {"kind": "zfs_vdev", "subject": "tank/raidz2-0/disk-b", "pool": "tank", "vdev_type": "disk", "state": "FAULTED", "read_errors": 3, "write_errors": 0, "checksum_errors": 47, "is_spare": false, "is_replacing": false},
    {"kind": "zfs_spare", "subject": "tank/spare-0", "pool": "tank", "state": "INUSE", "is_active": true, "replacing_vdev_guid": "1234567890123456"},
    {"kind": "zfs_scan", "subject": "tank", "pool": "tank", "scan_type": "scrub", "scan_state": "completed", "last_completed_at": "2026-04-12T03:14:00Z", "errors_found": 0, "repaired_bytes": 0}
  ],
  "errors": []
}
```

NQ's regime features over multiple generations of this report detect: pool DEGRADED for N generations, disk-b error counts flat (no rise), scrub clean, spare assigned and holding. That's the evidence set that supports `zfs_degraded_stable` classification.

If `disk-b.checksum_errors` rises across generations, OR a second vdev transitions to DEGRADED/FAULTED, OR a scrub reports new errors — the regime re-classifies as `zfs_degraded_worsening`. The witness doesn't make that classification; it just reports the facts that let the classifier detect the transition.

### Partial collection failure

```json
{
  "schema": "nq.witness.v0",
  "witness": {
    "id": "zfs.local.example-host",
    "type": "zfs",
    "host": "example-host",
    "profile_version": "nq.witness.zfs.v0",
    "collection_mode": "sudo_helper",
    "privilege_model": "nopasswd_fixed_helper",
    "collected_at": "2026-04-16T22:30:00Z",
    "duration_ms": 5010,
    "status": "partial"
  },
  "coverage": {
    "can_testify": ["pool_state", "pool_capacity"],
    "cannot_testify": ["vdev_state", "vdev_error_counters", "scrub_state", "scrub_completion", "spare_state", "smart_drive_health"]
  },
  "standing": {
    "authoritative_for": ["current_pool_state"],
    "advisory_for": [],
    "inadmissible_for": ["chronic_vs_worsening_regime_classification", "authorization", "remediation"]
  },
  "observations": [
    {"kind": "zfs_pool", "subject": "tank", "state": "DEGRADED", "size_bytes": 10000000000000, "alloc_bytes": 2000000000000, "free_bytes": 8000000000000, "readonly": false, "fragmentation_ratio": 0.05}
  ],
  "errors": [
    {"kind": "zpool_status_timeout", "detail": "zpool status exceeded 5s timeout; no vdev/scan/spare data available this cycle", "observed_at": "2026-04-16T22:30:05Z"}
  ]
}
```

This is the honest failure: partial collection, coverage demoted, status=partial, explicit error. The consumer gets pool-level liveness but correctly cannot classify regime.

## Not required in v0

- SMART drive health (separate profile, `profiles/smart.md` when written)
- ZFS encryption state or key status
- Dataset snapshot inventories
- Send/receive progress
- ZFS event daemon (ZED) event history
- Per-pool property dump (`zpool get all`)
- Pool history (`zpool history`)

Add these in profile v1+ if a concrete use case justifies them.

## Conformance

A ZFS witness conforms to this profile when:

1. It emits the canonical JSON report shape from SPEC.md with `profile_version: "nq.witness.zfs.v0"`.
2. It provides all required `coverage.can_testify` tags, or honestly lists missing ones in `cannot_testify`.
3. It emits `zfs_pool`, `zfs_vdev`, `zfs_scan`, and (when spares exist) `zfs_spare` observations with the required fields.
4. It respects the freshness and failure-behavior rules from SPEC.md.
5. It does not claim coverage for data it cannot reliably source.

A Prometheus-only exporter that exposes a subset of these fields but does not emit canonical JSON and does not declare coverage/standing is NOT a conforming witness under this profile. It may still be useful as a coarse Prometheus source; it just doesn't satisfy the forensic-standing requirement.
