# SMART Witness Profile (v0) — Phase 1 raw evidence only

**Profile version:** `nq.witness.smart.v0`
**Status:** draft, Phase 1
**Applies to:** Linux hosts with `smartmontools >= 7.2` (for JSON output via `-j`). macOS and FreeBSD are compatible in principle with recent smartmontools; not tested here.
**Forcing case:** a drive that `smartctl` reports as `smart_status.passed: true` while its `scsi_error_counter_log.read.total_uncorrected_errors` is nonzero. SMART's overall status is aspirational; the evidence is in the detail fields. The witness must surface both without reconciling them — reconciliation is a detector's job, and there is no detector in Phase 1.

## North star

> **A SMART witness answers "what did the device say?" not "what do we think it means?"**

Phase 1 surfaces raw device-reported evidence. There are no verdicts, no synthetic health scores, no cross-device aggregations. Detector work is a later phase, on top of this substrate.

## Structural difference from other profiles

Unlike the ZFS profile, where one host has one pool and one witness, a SMART witness emits one report containing **many** per-device observations, each with its own capabilities. An NVMe drive cannot testify to `scsi_error_counters`; a SAS drive cannot testify to `nvme_health_log`; a USB-bridged drive may not be able to testify to anything beyond identity. Coverage therefore lives at two tiers:

- **Witness-level coverage** declares whether the witness could enumerate devices at all, and the privilege it actually achieved.
- **Per-observation coverage** declares which attribute groups each specific device could testify about.

Consumers must read both. A witness that declares `device_enumeration` in its top-level `can_testify` is not implicitly claiming per-device detail coverage — every device's own block is authoritative for that device.

**Normative consumer rule:** detector gating reads the per-observation `coverage.can_testify` block for the device the detector is evaluating. Witness-level coverage exists only for enumeration and privilege context, and feeds source-level completeness propagation. A consumer that gates a device-detail detector on witness-level coverage is violating the contract.

## Required witness-level `can_testify` tags

A conforming SMART witness MUST testify about:

- `device_enumeration` — it successfully listed SMART-capable devices on this host.

That is the only universal tag. Every other capability varies per device and belongs in the per-observation block.

## Optional witness-level `can_testify` tags

- `smartd_cache_readable` — the witness read cached output from `smartd` instead of (or in addition to) spawning `smartctl`. Privilege model implication: unprivileged read of a cache is different from privileged invocation. If a deployment relies on the smartd-cache path, declaring this tag makes the evidence path explicit.

## Witness-level enumeration behavior

`device_enumeration` coverage must reflect what actually happened at the witness level, not just "we intended to scan." Expected behaviors:

| Scenario                                         | `witness.status` | `device_enumeration` | Observations emitted |
|---                                               |---               |---                  |---                   |
| Scan completed, ≥1 SMART-capable device returned | `ok`             | in `can_testify`    | one per device        |
| Scan completed, zero devices returned            | `ok`             | in `can_testify`    | zero (legitimate for disk-less VMs) |
| Scan command failed (exec error, missing binary) | `failed`         | in `cannot_testify` | zero; witness-level `errors[]` populated |
| Scan succeeded but per-device opens all failed   | `partial`        | in `can_testify`    | one per device with `collection_outcome: permission_denied` or similar |
| Scan timed out                                   | `failed`         | in `cannot_testify` | zero; witness-level `errors[]` populated with `kind: "scan_timeout"` |
| Scan succeeded with partial privilege (bridge refusal, etc.) | `partial` | in `can_testify` | devices that opened succeed; others land as `permission_denied` / `error` |

A zero-device scan is a legitimate outcome, not a failure — disk-less VMs, network-booted hosts, and containers without block-device access all have zero SMART-capable devices. The honest answer is `witness.status: ok` + empty `observations[]`. The anti-pattern is emitting `failed` because "we expected to find drives and didn't," because that confuses deployment-expectation gaps with collection failures.

A witness-level `errors[]` entry for enumeration failure carries a `kind` from the bounded set: `scan_failed`, `scan_timeout`, `scan_binary_missing`, `scan_insufficient_privilege`.

## Required observation kind

### `smart_device`

One observation per device discovered by `smartctl --scan` or equivalent. Required fields:

```json
{
  "kind": "smart_device",
  "subject": "wwn:0x5000cca26adf4db8",
  "device_path": "/dev/disk/by-id/wwn-0x5000cca26adf4db8",
  "device_class": "scsi",
  "protocol": "SCSI",
  "model": "HGST HUH721010AL42C0",
  "serial_number": "2TKYU2KD",
  "firmware_version": "A38K",
  "capacity_bytes": 10000831348736,
  "logical_block_size": 4096,
  "smart_available": true,
  "smart_enabled": true,
  "smart_overall_passed": true,
  "temperature_c": 30,
  "power_on_hours": 4872,
  "uncorrected_read_errors": 88,
  "uncorrected_write_errors": 0,
  "uncorrected_verify_errors": 0,
  "media_errors": null,
  "nvme_percentage_used": null,
  "nvme_available_spare_pct": null,
  "nvme_critical_warning": null,
  "nvme_unsafe_shutdowns": null,
  "coverage": {
    "can_testify": [
      "device_identity",
      "smart_availability",
      "smart_overall_status",
      "temperature",
      "power_on_hours",
      "scsi_error_counters"
    ],
    "cannot_testify": [
      "nvme_health_log",
      "ata_smart_attributes"
    ]
  },
  "collection_outcome": "ok",
  "raw": { "...": "verbatim smartctl -j -a output, envelope stripped" }
}
```

### Subject identity (§Invariant — stable key)

The `subject` field is the identity under which the device's observations will be keyed across generations, reboots, and drive-bay shuffles. Three tiers, in priority order:

1. **Stable across hardware:** if the device reports a SCSI `logical_unit_id` or an ATA `wwn` → `wwn:<hex-lowercase-no-prefix>` (e.g. `wwn:0x5000cca26adf4db8`). Unique beyond host scope.
2. **Stable within host:** if `serial_number` is present and non-empty → `serial:<value>`. Unique only within host scope; two hosts might have drives with colliding serials (rare but not excluded by manufacturers historically). Consumers that federate across hosts should scope by host when keying on serial subjects.
3. **Last-resort unstable:** `path:<device_path>` (e.g. `path:/dev/sdh`). `device_path` flaps on reboot, on hotplug, and on HBA renumbering. Its use as a subject is explicitly a fallback, never a preference.

`device_path` is metadata even when used as subject — prefer `/dev/disk/by-id/...` forms over bare `/dev/sdX` when the bus exposes them.

**Normative rules:**

- A witness that falls back to tier 3 (`path:`) MUST set `collection_outcome: partial` AND list `device_identity` in this observation's `cannot_testify`. No silent tier-3 subjects.
- **Consumers MUST treat `path:` subjects as single-cycle records.** Cross-generation correlation on a `path:` subject is forbidden — the identity is not guaranteed to refer to the same physical device across reboots. Warning state, regime features, and basis lifecycle must all scope to `path:` findings within a single generation window.
- Subject prefix (`wwn:` / `serial:` / `path:`) is load-bearing. Consumers pattern-match on the prefix to classify stability tier; prefixes are never optional and never reordered.
- Federation (future) adds scoping to tier 2 — a federated consumer treating `serial:` as globally unique across sites is violating the contract. Tier 1 (`wwn:`) remains globally unique by construction.

### Device class normalization

Mapped from `smartctl` device/protocol fields to a bounded enum:

| smartctl reports                                           | `device_class` |
|---                                                         |---              |
| `protocol: NVMe` or `type: nvme`                           | `nvme`          |
| `protocol: SCSI` or `type: scsi` / `sat`                   | `scsi`          |
| `protocol: ATA` or `type: ata`                             | `ata`           |
| USB bridge (detected via `type` prefix `usb-…`)            | `usb_bridge`    |
| Anything else, or unable to classify                       | `unknown`       |

Witnesses that cannot classify confidently MUST use `unknown` rather than guess. Misclassification compounds downstream — a consumer that believes a device is NVMe will look for fields that don't exist.

**Caveat on `usb_bridge`:** this value is doing two jobs. The underlying device is usually still ATA-ish or SCSI-ish in what SMART content exists, but the bridge frequently mediates access badly (passes through a subset of ATA commands, blocks SMART-specific ones, or presents a vendor-proprietary "enclosure" identifier that masks the real drive). Treat `usb_bridge` as a **collection-path warning** that the evidence quality is bridge-mediated, not as a health-schema class. A USB-bridged SATA disk and a natively-attached SATA disk have different field availability despite being the same physical hardware class. Phase 2 detector gating should use per-observation coverage, not device class, to avoid building assumptions on top of this ambiguity.

### Per-observation `coverage` block

Each observation carries its own coverage block. Known tags:

| Tag                         | Meaning |
|---                          |---      |
| `device_identity`           | model + serial + firmware + stable subject key |
| `smart_availability`        | `smart_available` and `smart_enabled` booleans |
| `smart_overall_status`      | `smart_overall_passed` boolean (aspirational, see forcing case) |
| `temperature`               | `temperature_c` integer, current value |
| `power_on_hours`            | `power_on_hours` integer |
| `nvme_health_log`           | NVMe-specific: critical_warning, percentage_used, available_spare, media_errors, unsafe_shutdowns |
| `scsi_error_counters`       | SCSI-specific: uncorrected_{read,write,verify}_errors |
| `ata_smart_attributes`      | ATA-specific: the vendor attribute table surfaced raw inside `raw` |
| `nvme_error_information_log` | NVMe-specific: size/read/unread counts from the error log |

A field listed in the observation but whose coverage tag is NOT in `can_testify` is a protocol violation. Consumers may reject such observations.

Fields the witness cannot populate MUST be present in the observation but set to `null`, not omitted. Omission is a silent protocol break; explicit null carries "the witness tried and could not."

### `collection_outcome`

Per-device outcome string. Bounded enum:

| Value               | Meaning |
|---                  |---       |
| `ok`                | Every **class-applicable and successfully reachable** coverage tag for this device is either populated or explicitly moved to `cannot_testify`. `ok` is *not* "every nice field we hoped for is present" — it is "the witness finished without unexpected failures, and the cannot_testify list is a deliberate statement, not silence." |
| `partial`           | Device reachable, some expected tags populated, some unexpectedly moved to `cannot_testify`. Used when the witness ran but could not honor everything a well-formed device of this class would normally surface. Subject-identity fallback to `path:` (tier 3) always lands here. |
| `unsupported`       | `smart_support.available: false`. Device is reachable but cannot testify to SMART at all. Observation still carries identity (if available), path, and an honestly-empty `can_testify`. See §Unsupported devices. |
| `permission_denied` | `smartctl` returned permission error (exit 2 with "open device" in messages), distinct from generic `error` so consumers can render privilege-specific remediation rather than generic "collection broke." |
| `timeout`           | `smartctl` exceeded the witness's bounded timeout for this device. |
| `error`             | Any other `smartctl` failure. The `errors[]` array at witness level carries details. |

#### Unsupported devices

`unsupported` is not an omission path. An unsupported device MUST still be represented as an observation, with:

- `subject` populated per identity tier rules if any tier succeeds, otherwise a tier-3 path subject with the same partial-rules applied
- `device_path` present
- `smart_available: false` (explicit boolean, not null)
- `coverage.can_testify` containing at minimum `device_identity` if identity could be read; empty otherwise
- normalized fields populated where the device did surface them (some devices report capacity and model even with SMART disabled) or `null` where they did not
- `raw` carrying whatever smartctl returned, even if sparse

The rule: an `unsupported` device appearing as a populated observation is operator-legible ("this device exists, SMART is off"); an `unsupported` device silently omitted is operator-hostile ("where did my drive go?").

### `raw` subtree

Verbatim `smartctl -j -a` output with the outer `smartctl` envelope block stripped (it's redundant with witness-level metadata). Preserves everything structured that smartctl emitted, so future detectors or forensic queries can reach back to attribute-level evidence without the witness having to pre-normalize it.

**Bounded:** per-device raw payload is capped at 32 KiB. If the full output exceeds this, the witness truncates intelligently — preserves `nvme_smart_health_information_log`, `ata_smart_attributes.table`, `scsi_error_counter_log`, and discards verbose command chatter.

When truncation occurs, the observation MUST carry all of:

- `raw_full_fidelity` appended to `cannot_testify` (the coverage-tag form, for pipeline visibility)
- `raw_truncated: true` as a sibling of `raw` (explicit boolean, not inferred from coverage tags)
- `raw_original_bytes: <int>` and `raw_truncated_bytes: <int>` where `raw_truncated_bytes` is the size of the `raw` payload actually emitted after trimming

Inferring truncation from coverage-tag absence is expensive-to-debug later; the explicit boolean + byte counts pay that debt up front. Downstream debugging of a detector that behaves weirdly on a specific drive often starts from "did we see the full raw or a trimmed version?"

**Not a dumpster:** the witness does not concatenate multiple smartctl invocations into `raw`. It does not include `/proc`/`/sys` data. It does not include vendor-specific long-form diagnostic output.

## Deployment notes (non-normative)

Cadence and freshness are deployment concerns, not profile law. They're noted here because spinning disks behave differently from SSDs under frequent smartctl polling:

- 5 minutes is a reasonable default for mixed SSD/HDD hosts.
- Sub-minute cadence is cheap on NVMe/SSD but wasteful on spinning disks — smartctl access can wake a drive from standby, defeating `-n standby` intent elsewhere.
- Consumer-side stale threshold (for `nq_witness_silent` to fire) is a publisher/consumer setting, not a profile rule.

None of these values change what a conforming witness emits; they only shape how often it runs.

## Failure behavior (inherits SPEC §Failure behavior, plus)

- **Privilege missing ≠ drive healthy.** A device that smartctl cannot open because of permissions MUST land as `collection_outcome: permission_denied`. The witness MUST NOT omit the device. Silence on a permission error would let a misconfigured deploy render as "no SMART concerns."
- **`smart_status.passed` is not authoritative for health.** The profile surfaces it because smartctl does, but the forcing case proves it can be `true` on a drive with real uncorrected errors. Phase 1 surfaces both fields; reconciling them is detector work.
- **Never invent stable identity.** If neither WWN nor serial is available, `collection_outcome: partial` with `device_identity` in `cannot_testify`. Inventing an identity from path or position is worse than admitting uncertainty.

## Standing

Default standing block for a SMART witness:

```json
{
  "authoritative_for": [
    "smart_reported_uncorrected_error_counts",
    "smart_reported_power_on_hours",
    "smart_reported_temperature"
  ],
  "advisory_for": [
    "drive_health_overall"
  ],
  "inadmissible_for": [
    "drive_failure_prediction",
    "replacement_authorization",
    "vendor_rma_eligibility",
    "authorization",
    "remediation"
  ]
}
```

`drive_health_overall` is advisory, not authoritative, because the forcing case demonstrates that `smart_status.passed` can contradict raw error counters. A detector that wants to make a health call must reconcile the two; the witness refuses to pre-reconcile.

**Advisory is not discardable.** Evidence classified as `advisory_for` remains export-worthy and must propagate through finding exports, regime features, and downstream consumers. The word "advisory" in this contract means "not sufficient on its own for a health claim," not "consumers may ignore." A consumer that filters advisory evidence out of its rendering is violating the contract; what it may do is require a non-advisory source to corroborate before firing on advisory evidence alone.

## Phase 1 non-goals

- No detector logic. No `smart_predictive_failure`, no `smart_udma_errors_rising`, no `smart_temperature_high`. These arrive in Phase 2, after the raw substrate has been in production long enough to tune against real data.
- No cross-device aggregation. Phase 1 emits per-device observations; any host-level or array-level rollup is a consumer concern.
- No normalization of ATA vendor attributes into a common schema. ATA attribute IDs mean different things across vendors (e.g. attribute 5 — reallocated sectors — is common, but attribute 190 temperature has three different encodings across SSD vendors). The witness surfaces the raw attribute table; consumers and future detectors can normalize the subset they care about.
- No smartd notifications, no active scanning beyond `smartctl -a`, no self-test scheduling, no mutation of device state.
