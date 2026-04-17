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

## 1. `collection_mode` enum is missing the unprivileged-subprocess case

**Severity:** structural — the MVP reference witness is currently *fabricating* a value.

**Location:** `SPEC.md` §Section requirements 2 (witness). The enum is
`{embedded, exporter, textfile, sudo_helper}`. None of these fit a helper that
runs as an ordinary user via subprocess invocation (no sudo, no long-running
daemon, no file drop).

**Forcing case:** `lil-nas-x` (2026-04-17) has `/dev/zfs` at permissions `0666`,
so the reference ZFS witness at `examples/nq-zfs-witness` can run as the
unprivileged `claude` user — no sudoers entry needed. This is a legitimate
deployment shape; `/dev/zfs` permissions are a distribution / kernel-configuration
decision, and an unprivileged witness there is the principled option
(strictly less privilege than a `sudo_helper`).

**Current lie:** the MVP witness emits `collection_mode: "sudo_helper"` with
`privilege_model: "unprivileged"`. `sudo_helper` specifically means
"invokable via `sudo NOPASSWD`" per its spec definition. The MVP does not use
sudo. The two fields disagreeing is the tell.

**Why not fixed yet:** the fix is small (one new enum value) but it is an
enum-level spec change, not an MVP fix. It should be made deliberately in the
Python "clarify" pass of the witness implementation (per the
`exist → clarify → harden` progression) so that other first-contact scars can
be annealed in the same commit rather than in isolation.

**Proposed resolution:** add `subprocess` as a fifth value to the enum, defined
as "one-shot invocation without privilege elevation — caller spawns the
witness, witness emits a report and exits. Contrasts with `sudo_helper` which
is specifically sudo-mediated." Orthogonal to `privilege_model`, which already
captures the privilege dimension independently.

**Anti-social-normalization reminder:** every cycle the MVP runs on lil-nas-x,
its emitted report contains the `sudo_helper` lie. That output is the constant
reminder that this is not resolved. Do not add a workaround in the consumer
that papers over the mismatch; the consumer's behavior in the presence of
mismatched `collection_mode` / `privilege_model` should be to log a warning,
not to infer around it.

**Tracking:** referenced from
- `notquery/docs/gaps/ZFS_COLLECTOR_GAP.md` §Detectors (scar-tissue note)
- NQ project memory: `project_zfs_witness_mvp_scars.md` (scar #1)

---

## RESOLVED

*(empty — no resolved entries yet)*
