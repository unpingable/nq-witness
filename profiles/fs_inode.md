# Filesystem Inode Witness Profile (v0)

**Profile version:** `nq.witness.fs_inode.v0`
**Applies to:** Linux mounts that expose inode counts via `statvfs(2)` / `statfs(2)`. Predominantly inode-based filesystems (ext4, xfs, btrfs in default config). Filesystems without fixed inode caps (ZFS, btrfs with `inode_cache=off` semantics) are observable but report `inode_model: "unbounded"`. Pseudo-filesystems (`/proc`, `/sys`, `/dev`, `/run`) are out of scope by default.
**Forcing case:** filesystems running out of inodes before they run out of bytes. Small-file workloads (mail spools, package caches, runaway fork bombs into a tempdir) exhaust the inode table while disk bytes remain available. Free inodes and free bytes are distinct failure modes; this profile makes the inode axis visible without conflating it with byte capacity.

## Required `coverage.can_testify` tags

A conforming `fs_inode` witness MUST testify about:

- `mount_enumeration` — the set of mounts the witness actually inspected this cycle.
- `mount_scope_declaration` — the set of mounts in scope by configuration, with explicit listing of mounts excluded or present-but-uninspected, so omissions are visible.
- `inode_usage` — per-mount used / free / total inode counts where the filesystem supports them.

A witness missing any of these MUST list them in `cannot_testify` rather than emitting partial data or silence.

## Optional `coverage.can_testify` tags

- `fs_type_identification` — the `fs_type` field on each `fs_inode_state` observation, sourced from `/proc/self/mountinfo` or platform equivalent.
- `inode_pressure_ratio` — the precomputed `inodes_used / inodes_total` ratio on each observation.

## Required observation kinds

### `fs_inode_state`

One observation per inspected mount. Required fields:

```json
{
  "kind": "fs_inode_state",
  "subject": "/var",
  "mount_point": "/var",
  "fs_type": "ext4",
  "inode_model": "supported",
  "inodes_total": 4194304,
  "inodes_used": 2048000,
  "inodes_free": 2146304,
  "inode_usage_ratio": 0.4883
}
```

- `subject` — stable mount identifier; the mount-point path by default.
- `mount_point` — Linux mountpoint path. Equal to `subject` in standard deployments.
- `fs_type` — filesystem type as reported by `/proc/self/mountinfo`.
- `inode_model` — one of:
  - `supported` — filesystem reports fixed inode counts via statvfs (ext4, xfs, etc.). `inodes_*` fields are populated.
  - `unbounded` — filesystem has no fixed inode cap (ZFS, some btrfs configurations). `inodes_*` fields are `null`.
  - `unsupported` — filesystem cannot report inode usage at all (squashfs, certain network mounts). `inodes_*` fields are `null`.
- `inodes_total`, `inodes_used`, `inodes_free` — integer counts; `null` when `inode_model != "supported"`.
- `inode_usage_ratio` — precomputed `inodes_used / inodes_total` in `[0.0, 1.0]`; `null` when `inode_model != "supported"`.

### `fs_mount_scope`

Exactly one observation per report, declaring scope explicitly:

```json
{
  "kind": "fs_mount_scope",
  "subject": "scope_declaration",
  "mounts_in_scope": ["/", "/var", "/home"],
  "mounts_excluded_by_config": ["/proc", "/sys", "/dev", "/run"],
  "mounts_present_but_not_inspected": []
}
```

- `mounts_in_scope` — mounts the witness actively inspected this cycle.
- `mounts_excluded_by_config` — mounts deliberately excluded (typically pseudo-filesystems).
- `mounts_present_but_not_inspected` — mounts found on the host but not covered by either of the above. Empty when scope is complete. Non-empty entries represent operator-visible omission, distinct from intentional exclusion.

The scope observation is what closes the `mount_scope_declaration` coverage tag honestly. A witness that inspects only `/` while filesystems `/var` and `/home` exist must list those in `mounts_present_but_not_inspected`, not silently drop them.

## Standing defaults for fs_inode witnesses

```json
{
  "authoritative_for": [
    "current_inode_usage_per_inspected_mount",
    "current_mount_scope"
  ],
  "advisory_for": [],
  "inadmissible_for": [
    "byte_capacity",
    "filesystem_health",
    "application_impact",
    "inode_pressure_cause",
    "inode_pressure_trend",
    "authorization",
    "remediation"
  ]
}
```

`byte_capacity` is explicitly inadmissible — bytes and inodes are distinct failure modes. A witness that conflates them is laundering coverage. `filesystem_health` is inadmissible because "healthy" is not an inode-observable property. `inode_pressure_cause` is inadmissible because the witness sees the count, not the cause (runaway process, unrotated log dir, misconfigured mail spool). `inode_pressure_trend` is inadmissible because a single snapshot cannot establish trend — that requires cross-generation analysis the consumer (NQ) performs over multiple reports.

## Freshness defaults

- **Expected collection cadence:** every 60 seconds for a typical NQ deployment. Inode counts change slowly under normal load; faster cadence wastes cycles.
- **Stale threshold:** 5 minutes. Beyond this the consumer emits `nq_witness_silent` for the fs_inode witness.
- **Collection timeout:** 2 seconds. `statvfs` is a sub-millisecond call; an exceeded budget signals a hung mount, typically a network filesystem.

## Privilege model recommendations

- **`unprivileged`** (recommended) — `statvfs(2)` is callable by any user for mounts the user can `stat`. Sufficient for most baseline coverage.
- **`nopasswd_fixed_helper`** — for restricted mount points (encrypted homes, certain bind mounts). The helper performs `statvfs` and emits the canonical report.

The `root_exporter_localhost` model is acceptable but provides no extra coverage for inode counts; prefer `unprivileged` unless a specific mount requires elevation.

## Example reports

### Happy path — three mounts, all comfortable

```json
{
  "schema": "nq.witness.v0",
  "witness": {
    "id": "fs_inode.local.sushi-k",
    "type": "fs_inode",
    "host": "sushi-k",
    "profile_version": "nq.witness.fs_inode.v0",
    "collection_mode": "subprocess",
    "privilege_model": "unprivileged",
    "collected_at": "2026-05-21T16:30:00Z",
    "duration_ms": 4,
    "status": "ok"
  },
  "coverage": {
    "can_testify": [
      "mount_enumeration",
      "mount_scope_declaration",
      "inode_usage",
      "fs_type_identification",
      "inode_pressure_ratio"
    ],
    "cannot_testify": []
  },
  "standing": {
    "authoritative_for": [
      "current_inode_usage_per_inspected_mount",
      "current_mount_scope"
    ],
    "advisory_for": [],
    "inadmissible_for": [
      "byte_capacity",
      "filesystem_health",
      "application_impact",
      "inode_pressure_cause",
      "inode_pressure_trend",
      "authorization",
      "remediation"
    ]
  },
  "observations": [
    {
      "kind": "fs_mount_scope",
      "subject": "scope_declaration",
      "mounts_in_scope": ["/", "/var", "/home"],
      "mounts_excluded_by_config": ["/proc", "/sys", "/dev", "/run", "/dev/shm"],
      "mounts_present_but_not_inspected": []
    },
    {
      "kind": "fs_inode_state",
      "subject": "/",
      "mount_point": "/",
      "fs_type": "ext4",
      "inode_model": "supported",
      "inodes_total": 6553600,
      "inodes_used": 412803,
      "inodes_free": 6140797,
      "inode_usage_ratio": 0.063
    },
    {
      "kind": "fs_inode_state",
      "subject": "/var",
      "mount_point": "/var",
      "fs_type": "ext4",
      "inode_model": "supported",
      "inodes_total": 4194304,
      "inodes_used": 198403,
      "inodes_free": 3995901,
      "inode_usage_ratio": 0.047
    },
    {
      "kind": "fs_inode_state",
      "subject": "/home",
      "mount_point": "/home",
      "fs_type": "ext4",
      "inode_model": "supported",
      "inodes_total": 30531584,
      "inodes_used": 1041927,
      "inodes_free": 29489657,
      "inode_usage_ratio": 0.034
    }
  ],
  "errors": []
}
```

### Forcing case — inode pressure rising on /var, ZFS mount honestly unbounded

```json
{
  "schema": "nq.witness.v0",
  "witness": {
    "id": "fs_inode.local.sushi-k",
    "type": "fs_inode",
    "host": "sushi-k",
    "profile_version": "nq.witness.fs_inode.v0",
    "collection_mode": "subprocess",
    "privilege_model": "unprivileged",
    "collected_at": "2026-05-21T16:30:00Z",
    "duration_ms": 5,
    "status": "ok"
  },
  "coverage": {
    "can_testify": [
      "mount_enumeration",
      "mount_scope_declaration",
      "inode_usage",
      "fs_type_identification",
      "inode_pressure_ratio"
    ],
    "cannot_testify": []
  },
  "standing": {
    "authoritative_for": [
      "current_inode_usage_per_inspected_mount",
      "current_mount_scope"
    ],
    "advisory_for": [],
    "inadmissible_for": [
      "byte_capacity",
      "filesystem_health",
      "application_impact",
      "inode_pressure_cause",
      "inode_pressure_trend",
      "authorization",
      "remediation"
    ]
  },
  "observations": [
    {
      "kind": "fs_mount_scope",
      "subject": "scope_declaration",
      "mounts_in_scope": ["/", "/var", "/home", "/tank/data"],
      "mounts_excluded_by_config": ["/proc", "/sys", "/dev", "/run"],
      "mounts_present_but_not_inspected": []
    },
    {
      "kind": "fs_inode_state",
      "subject": "/",
      "mount_point": "/",
      "fs_type": "ext4",
      "inode_model": "supported",
      "inodes_total": 6553600,
      "inodes_used": 415102,
      "inodes_free": 6138498,
      "inode_usage_ratio": 0.063
    },
    {
      "kind": "fs_inode_state",
      "subject": "/var",
      "mount_point": "/var",
      "fs_type": "ext4",
      "inode_model": "supported",
      "inodes_total": 4194304,
      "inodes_used": 4099201,
      "inodes_free": 95103,
      "inode_usage_ratio": 0.9773
    },
    {
      "kind": "fs_inode_state",
      "subject": "/home",
      "mount_point": "/home",
      "fs_type": "ext4",
      "inode_model": "supported",
      "inodes_total": 30531584,
      "inodes_used": 1042993,
      "inodes_free": 29488591,
      "inode_usage_ratio": 0.034
    },
    {
      "kind": "fs_inode_state",
      "subject": "/tank/data",
      "mount_point": "/tank/data",
      "fs_type": "zfs",
      "inode_model": "unbounded",
      "inodes_total": null,
      "inodes_used": null,
      "inodes_free": null,
      "inode_usage_ratio": null
    }
  ],
  "errors": []
}
```

NQ's regime features over multiple generations of this report detect: `/var` `inode_usage_ratio` elevated for N generations, slope of `inodes_used` rising — the regime classifier might name that `fs_inode_pressure_rising`. The witness does **not** make that classification; it reports the counts that let the classifier detect the transition. The ZFS mount is honestly reported as `inode_model: "unbounded"` rather than dropped or silently zeroed.

### Partial collection — hung NFS mount

```json
{
  "schema": "nq.witness.v0",
  "witness": {
    "id": "fs_inode.local.sushi-k",
    "type": "fs_inode",
    "host": "sushi-k",
    "profile_version": "nq.witness.fs_inode.v0",
    "collection_mode": "subprocess",
    "privilege_model": "unprivileged",
    "collected_at": "2026-05-21T16:30:00Z",
    "duration_ms": 2014,
    "status": "partial"
  },
  "coverage": {
    "can_testify": [
      "mount_enumeration",
      "mount_scope_declaration",
      "fs_type_identification"
    ],
    "cannot_testify": [
      "inode_usage",
      "inode_pressure_ratio"
    ]
  },
  "standing": {
    "authoritative_for": [
      "current_mount_scope"
    ],
    "advisory_for": [],
    "inadmissible_for": [
      "current_inode_usage_per_inspected_mount",
      "byte_capacity",
      "filesystem_health",
      "application_impact",
      "inode_pressure_cause",
      "inode_pressure_trend",
      "authorization",
      "remediation"
    ]
  },
  "observations": [
    {
      "kind": "fs_mount_scope",
      "subject": "scope_declaration",
      "mounts_in_scope": ["/", "/var", "/home", "/mnt/nfs"],
      "mounts_excluded_by_config": ["/proc", "/sys", "/dev", "/run"],
      "mounts_present_but_not_inspected": []
    }
  ],
  "errors": [
    {
      "kind": "statvfs_timeout",
      "detail": "statvfs(\"/mnt/nfs\") exceeded 2s budget; NFS mount likely hung. inode_usage demoted for all mounts this cycle to keep collection bounded.",
      "observed_at": "2026-05-21T16:30:02Z"
    }
  ]
}
```

A hung NFS mount can block `statvfs` on unrelated mounts when the collector serializes calls; the honest response is to demote `inode_usage` coverage for the cycle rather than emit half a report. The next cycle may reset if the NFS mount recovers or the operator removes it from scope.

## Not required in v0

- Per-mount byte capacity (`fs_capacity` is a separate profile when authored).
- File counts by directory.
- Inode allocation by user / group (per-uid quotas).
- Inode reservation tracking (xfs reserved-blocks-equivalent).
- Per-superblock journaling or metadata pressure.
- Trend or regime classification — that is the consumer's (NQ's) concern; the witness reports counts.

Add these in profile v1+ if a concrete use case justifies them.

## Conformance

A `fs_inode` witness conforms to this profile when:

1. It emits the canonical JSON report shape from `SPEC.md` with `profile_version: "nq.witness.fs_inode.v0"`.
2. It enumerates mounts via `/proc/self/mountinfo` (or platform equivalent) and reports both inspected and excluded sets in the `fs_mount_scope` observation.
3. It honestly reports `inode_model` per mount, including `unbounded` for ZFS and `unsupported` for filesystems that cannot report inode counts.
4. It declares `byte_capacity` as inadmissible standing — inode pressure is not byte capacity.
5. It respects the freshness and failure-behavior rules from `SPEC.md`.
6. It bounds `statvfs` calls with a per-mount or overall timeout; a hung mount must demote `inode_usage` coverage rather than block the report.

A baseline Prometheus exporter exposing `node_filesystem_files` / `node_filesystem_files_free` (node_exporter) without `coverage` / `standing` declarations is NOT a conforming witness under this profile. It can be ingested as a Tier 2 source per `SPEC.md`, but coverage- and standing-keyed detectors require a Tier 1 witness.

## Coverage audit cross-reference

This profile closes the `fs_inode` row of the NQ coverage audit (`notquery/docs/coverage/traditional-monitoring-coverage-audit.md`). The forcing case for naming this row was the traditional-monitoring observation that inode exhaustion is a distinct failure axis from byte capacity — a canonical "boring obvious" omission in monitoring stacks that conflate "disk is full" with "filesystem is full."
