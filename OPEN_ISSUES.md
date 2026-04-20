# Open Issues — unresolved constitutional debt

Known failure modes in the published spec that have been identified but not yet
resolved. Each entry is a **debt note**, not a bug report: it names a place where
the spec is currently *wrong*, *incomplete*, or *lying*, and commits to a future
resolution path.

This file is the constitutional-debt register. If a witness implementation or a
consumer encounters one of these while working, check this file before assuming
the spec is self-consistent. Do not silently route around the issue in code —
flag the relevant entry in the implementation's own scar notes or commit
message, and update this file when the debt is paid.

Entries are numbered for stable reference. New entries append; entries are
closed by being moved into a `RESOLVED` section below their original number,
with a link to the commit or spec version that resolved them.

---

## 2. `zfs_vdev` profile fields vs coverage-tag granularity

**Severity:** profile-design — coverage tags are coarser than per-observation
field requirements, so a witness can declare honest coverage and still omit
required fields inside an observation without tripping any contract check.

**Location:** `profiles/zfs.md` §`zfs_vdev` required fields (`vdev_path`,
`vdev_guid`, `vdev_type`) vs §Required coverage tags (`vdev_state`,
`vdev_error_counters`). A witness can legitimately claim `vdev_state` coverage
while omitting `vdev_path` / `vdev_guid` / `vdev_type` — those require `zdb`
(root-only on most distros) or udev-path resolution the helper may not do.

**Forcing case:** the reference ZFS witness on lil-nas-x emits `zfs_vdev`
observations without `vdev_guid`, `vdev_path`, or `vdev_type`, yet its
coverage declarations are honest. Nothing in the contract catches the gap.

**Why not fixed yet:** three plausible resolutions; the right one depends on
what NQ detectors actually key by once the collector lands.

**Proposed resolution options:**

1. Split required-field sets into finer coverage tags (`vdev_identity` tag
   covers guid/path/type; `vdev_state` covers state; `vdev_error_counters`
   covers counters). Finer-grained, more moving parts.
2. Accept nullable required fields and add a profile-level conformance test
   that reports which required fields are populated per observation.
3. Keep the profile coarse but require a sidecar field on the observation
   declaring "partial observation; fields X, Y, Z absent."

**How to apply:** revisit once the NQ-side ZFS collector exists and real
detectors exercise the observations. If detectors key by `subject` and
`pool` without caring about `vdev_guid` / `vdev_path`, option 2 is fine.
If detectors need GUIDs for cross-generation identity, option 1 is forced.

**Tracking:** referenced from NQ project memory
`project_zfs_witness_mvp_scars.md` (scar #2).

---

## RESOLVED

### 1. `collection_mode` enum missing the unprivileged-subprocess case

**Resolved in:** clarify-pass commit adding `subprocess` to the
`collection_mode` enum in `SPEC.md` §Section requirements 2.

**Resolution:** the enum now reads
`{embedded, exporter, textfile, subprocess, sudo_helper}`. `subprocess` is
defined as "one-shot invocation without privilege elevation — caller spawns
the witness, witness emits a report and exits." Distinct from `sudo_helper`,
which is specifically sudo-mediated invocation. Orthogonal to
`privilege_model`, which captures the privilege dimension independently.

The reference ZFS witness at `examples/nq-zfs-witness` now emits
`collection_mode: "subprocess"` + `privilege_model: "unprivileged"` by
default on deployments where `/dev/zfs` permissions allow unprivileged
reads (the lil-nas-x case that forced this fix). Deployments using a
sudo-NOPASSWD wrapper emit the previous `sudo_helper` + `nopasswd_fixed_helper`
pair, triggered via `NQ_COLLECTION_MODE` / `NQ_PRIVILEGE_MODEL` env vars.
