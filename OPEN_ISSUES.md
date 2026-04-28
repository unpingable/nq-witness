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

## 3. First-class witness position

**Severity:** latent schema pressure — canonical reports identify `witness.type`
and profile but do not declare witness position explicitly. Position is currently
derivable from type because every shipped witness occupies the same position
(substrate). This stops being sufficient the moment a non-substrate witness
exists or NQ needs to correlate testimony across positions.

**Location:** `SPEC.md` §Section requirements 2 (`witness` block fields). No
field today carries the position dimension. `witness.type` carries domain
(`zfs`, `device_health`, etc.) but conflates "what the witness observes" with
"what register it observes from."

**Forcing case:** none yet. NQ-side doctrine (`docs/SCOPE_AND_WITNESS_MODEL.md`
in the notquery repo, codified 2026-04-28) names five witness positions:
`substrate`, `application_internal`, `application_external`, `platform_internal`,
`platform_external`. Existing shipped witnesses (ZFS, SMART) are all
`substrate`. The doctrine bites when: (a) the first non-substrate witness
profile is proposed (e.g. an HTTP-probe witness = `application_external`, a
systemd-unit witness = `platform_internal`), or (b) NQ needs to render or
correlate disagreement across witness positions for the same subject.

**Why not fixed yet:** today the field would be 100% derivable from
`witness.type` for every conforming witness. Requiring it now is schema theater
— consumers cannot use it because there is no cross-position case to consume.
Filing the debt avoids the alternative failure mode where the first
non-substrate witness ships and downstream reverse-engineers position from
type, locking in a local convention rather than a canonical declaration.

**Proposed resolution options:**

1. Add `witness.position` as a required field in the next schema version.
   Profiles enumerate valid positions for the domain (most profiles will
   declare exactly one). Schema bump (`v0` → `v1`) and consumer migration.
2. Add `witness.position` as an optional field in `v0` with a profile-level
   "MUST be declared starting from profile vN" rule. Lets adoption happen
   per-profile without a generic schema bump.
3. Defer entirely until a second position appears in the wild. Risks the
   reverse-engineer-from-type failure mode above; the upside is no schema
   churn for a field with one possible value.

**Constitutional line (applies under all three options):**

> Witness position is not authority. It locates testimony; it does not license
> action.

This prevents the later failure mode where some position (typically
`application_external`) gets treated as schema-level "higher truth" rather than
as a witness location whose evidentiary weight is dispute-specific.

**Candidate JSON shape (illustrative, not normative):**

```json
{
  "witness": {
    "id": "smart.local.lil-nas-x",
    "type": "device_health",
    "position": "substrate",
    "profile_version": "nq.witness.smart.v0"
  }
}
```

**How to apply:** revisit when the first non-substrate witness profile is
proposed (forces option 1 or 2) or when NQ-side detector code needs to read
`witness.position` to render cross-position disagreement (also forces option 1
or 2). Until then, `witness.type` carrying position implicitly is acceptable
local convention but should not be relied on as canonical testimony.

**Tracking:** referenced from NQ project memory `project_scope_axes.md`
(pointer to `notquery/docs/SCOPE_AND_WITNESS_MODEL.md`). Doctrine pinned
2026-04-28; spec change deliberately deferred per "doctrine now, schema on
forcing case" pattern.

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
