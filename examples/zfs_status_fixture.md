# ZFS status fixture — synthetic chronic-degraded pool

Synthetic reference data for writing and testing a ZFS witness implementation. Represents the chronic-degraded forcing case: one faulted drive, spare assigned, pool otherwise stable. Error counts populated to demonstrate the stable-vs-worsening distinction.

Not drawn from any real deployment. Hostnames, serials, and GUIDs are invented.

## Raw `zpool status` output

What a witness implementation parses. Format is operator-facing, not machine-facing; parsers accept the risk that ZFS may reformat.

```
  pool: tank
 state: DEGRADED
status: One or more devices could not be used because the label is missing or
        invalid.  Sufficient replicas exist for the pool to continue
        functioning in a degraded state.
action: Replace the device using 'zpool replace'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-4J
  scan: scrub repaired 0B in 04:12:17 with 0 errors on Sun Apr 12 07:26:33 2026
config:

        NAME                                              STATE     READ WRITE CKSUM
        tank                                              DEGRADED     0     0     0
          raidz2-0                                        DEGRADED     0     0     0
            ata-EXAMPLE-DISK-0000000001                   ONLINE       0     0     0
            spare-1                                       DEGRADED     0     0     0
              ata-EXAMPLE-DISK-0000000002                 FAULTED      3     0    47  too many errors
              ata-EXAMPLE-SPARE-0000000001                ONLINE       0     0     0
            ata-EXAMPLE-DISK-0000000003                   ONLINE       0     0     0
            ata-EXAMPLE-DISK-0000000004                   ONLINE       0     0     0
            ata-EXAMPLE-DISK-0000000005                   ONLINE       0     0     0
            ata-EXAMPLE-DISK-0000000006                   ONLINE       0     0     0
        spares
          ata-EXAMPLE-SPARE-0000000001                    INUSE     currently in use
          ata-EXAMPLE-SPARE-0000000002                    AVAIL

errors: No known data errors
```

## Parsed into canonical witness report

What the same state looks like after the witness has parsed and structured it. This is the authoritative representation — the Prometheus projection (if any) would derive from here.

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
    "can_testify": [
      "pool_state",
      "pool_capacity",
      "vdev_state",
      "vdev_error_counters",
      "scrub_state",
      "scrub_completion",
      "spare_state"
    ],
    "cannot_testify": [
      "smart_drive_health",
      "enclosure_slot_mapping",
      "dataset_properties"
    ]
  },
  "standing": {
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
  },
  "observations": [
    {
      "kind": "zfs_pool",
      "subject": "tank",
      "state": "DEGRADED",
      "health_numeric": 1,
      "size_bytes": 79989470920704,
      "alloc_bytes": 8277407145984,
      "free_bytes": 71712063774720,
      "readonly": false,
      "fragmentation_ratio": 0.0
    },
    {
      "kind": "zfs_vdev",
      "subject": "tank/raidz2-0/ata-EXAMPLE-DISK-0000000001",
      "pool": "tank",
      "vdev_path": "/dev/disk/by-id/ata-EXAMPLE-DISK-0000000001",
      "vdev_guid": "1000000000000000001",
      "vdev_type": "disk",
      "state": "ONLINE",
      "read_errors": 0,
      "write_errors": 0,
      "checksum_errors": 0,
      "is_spare": false,
      "is_replacing": false
    },
    {
      "kind": "zfs_vdev",
      "subject": "tank/raidz2-0/ata-EXAMPLE-DISK-0000000002",
      "pool": "tank",
      "vdev_path": "/dev/disk/by-id/ata-EXAMPLE-DISK-0000000002",
      "vdev_guid": "1000000000000000002",
      "vdev_type": "disk",
      "state": "FAULTED",
      "read_errors": 3,
      "write_errors": 0,
      "checksum_errors": 47,
      "is_spare": false,
      "is_replacing": true
    },
    {
      "kind": "zfs_vdev",
      "subject": "tank/raidz2-0/ata-EXAMPLE-SPARE-0000000001",
      "pool": "tank",
      "vdev_path": "/dev/disk/by-id/ata-EXAMPLE-SPARE-0000000001",
      "vdev_guid": "2000000000000000001",
      "vdev_type": "disk",
      "state": "ONLINE",
      "read_errors": 0,
      "write_errors": 0,
      "checksum_errors": 0,
      "is_spare": true,
      "is_replacing": true
    },
    {
      "kind": "zfs_scan",
      "subject": "tank",
      "pool": "tank",
      "scan_type": "scrub",
      "scan_state": "completed",
      "last_completed_at": "2026-04-12T07:26:33Z",
      "errors_found": 0,
      "repaired_bytes": 0
    },
    {
      "kind": "zfs_spare",
      "subject": "tank/spare/ata-EXAMPLE-SPARE-0000000001",
      "pool": "tank",
      "spare_path": "/dev/disk/by-id/ata-EXAMPLE-SPARE-0000000001",
      "spare_guid": "2000000000000000001",
      "state": "INUSE",
      "is_active": true,
      "replacing_vdev_guid": "1000000000000000002"
    },
    {
      "kind": "zfs_spare",
      "subject": "tank/spare/ata-EXAMPLE-SPARE-0000000002",
      "pool": "tank",
      "spare_path": "/dev/disk/by-id/ata-EXAMPLE-SPARE-0000000002",
      "spare_guid": "2000000000000000002",
      "state": "AVAIL",
      "is_active": false,
      "replacing_vdev_guid": null
    }
  ],
  "errors": []
}
```

## What NQ does with this

Across multiple generations of this report, NQ's regime features observe:

- `zfs_pool.state = DEGRADED` persistent for N generations → persistence class `persistent` or `entrenched` depending on duration
- `zfs_vdev[faulted_drive].checksum_errors` flat at 47 across generations → no worsening
- `zfs_scan.last_completed_at` stable → no new scrub events; `zfs_scrub_overdue` fires after configurable threshold
- `zfs_spare[spare_0001].is_active = true` persistent → spare assignment stable

The composite finding `zfs_pool_degraded` fires once, classifies as `persistent + stable`, and does not escalate severity across generations. Single finding per pool per persistence window, not one per generation.

If, in a future generation, any of these change:

- A second vdev state transitions to DEGRADED or FAULTED → `zfs_vdev_faulted` fires, severity escalates
- `checksum_errors` on any vdev rises from last generation → `zfs_error_count_increased` fires
- `scan_state` transitions to `in_progress` and `scan_type: resilver` → resilver event, notify
- `last_completed_at` older than policy → `zfs_scrub_overdue` fires

The regime classifier transitions the pool-level finding from `persistent + stable` to `degraded + worsening`, and severity escalates accordingly.

## Notes for witness implementers

1. `spare-1` in the `zpool status` output is a compound vdev representing the spare-replacement relationship. Parse it carefully: it wraps both the failed device and the spare currently in use. The canonical report represents this by setting `is_replacing: true` on the faulted vdev and `is_replacing: true` on the spare vdev, with `replacing_vdev_guid` linking them.

2. GUIDs in this fixture are invented placeholders. Real implementations must extract actual GUIDs from `zpool status -g` or `zpool list -Hv -o name,guid`.

3. The `status:` and `action:` lines from `zpool status` are human-operator-facing. Witness implementations MAY include them as an `advisory_message` field on the pool observation, but consumers should not treat them as authoritative machine-parseable content.

4. This fixture shows `errors: No known data errors`. A real-world degraded pool with scrub-discovered errors would have a different `errors:` section that witness implementations SHOULD parse and expose via the `zfs_scan.errors_found` field.
