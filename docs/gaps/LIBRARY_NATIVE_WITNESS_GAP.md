# Gap: Library-Native Witness Construction — typed mint, shared validator

**Status:** candidate gap — name the surface, do not build yet. **No crate creation, no schema migration, no `mint` block addition is ratified by this filing.** Future forcing cases promote.
**Depends on:** none (orthogonal — names a construction discipline, not a build slice)
**Build phase:** doctrine — adds a producer-side construction discipline and a shared consumer-side validator, both library-mediated
**Blocks:** any future native-language witness implementation (driftwatch, embedded Python services, third-profile work) that would otherwise hand-roll canonical JSON; any future tightening of the witness wire boundary that requires construction-side enforcement
**Last updated:** 2026-05-07

## Keepers

> **Schema possession is not testimonial standing. The mint is not a badge printer.**

> **A witness report enters NQ as untrusted JSON. Validation is the boundary where shape becomes admissible testimony.**

The first names the producer-side discipline; the second names the consumer-side discipline. Both ends of the same wall.

Operational form:

> **The mint can make valid construction cheap. Only the consumer validator can make shape admissible.**

## The Problem

`nq-witness` is spec-first. SPEC.md and the per-profile docs (`profiles/zfs.md`, `profiles/smart.md`) define what a valid witness report looks like; reference implementations (`examples/nq-zfs-witness`, `examples/nq-smart-witness`) hand-roll canonical JSON in bash. Two profiles, two reference impls, all hand-rolled.

The chalk outline for the missing piece is already in the spec:

- SPEC.md states the witness contract is **semantic, not transport-specific** — NQ consumes one canonical witness report regardless of origin.
- SPEC.md explicitly refuses to define programming language or transport.
- `collection_mode: embedded` is already a valid mode in the enum.

Native libraries wearing a fake mustache are basically permitted by the existing spec. What's missing is the library that makes hand-rolling JSON harder than constructing reports through profile-bound builders.

The dangerous default, today and for every future profile:

```text
Application
  → hand-rolls JSON
    → NQ trusts it because shape good
```

The intended path:

```text
Application
  → nq-witness-mint library
    → typed observations + coverage declarations
      → profile-validated canonical report
        → stdout/file/socket/whatever
          → NQ consumer
```

This gap names the library before a third profile or first native application forces it.

## Why this is being named now

Retrofit cost, not active pain. Two bash reference impls hand-rolling JSON is fine today. The library becomes load-bearing the first time:

- a native application (driftwatch, an embedded Python service, anything that doesn't have an operator copy-pasting from `~/git/nq-witness/examples/`) wants to emit testimony, or
- a third profile lands and the shared report-construction code that doesn't exist becomes hand-duplicated across N implementations, or
- NQ's own consumer wants to share a validator with producers so "valid witness report" means the same thing on both sides of the wire.

Per `~/.claude/CLAUDE.md` YAGNI doctrine: name the candidate, ratify lazily. Wait-until-forced is a brake, not a ceiling — naming load-bearing surfaces is justified by retrofit cost, not only by forcing case. Two profiles + two reference impls + the SPEC's `collection_mode: embedded` mode → the body has been discovered. Filing now is justified; building is a separate decision.

## Cross-system relationship

This gap is the **construction-side sibling** of NQ's `TESTIMONY_OBSERVABLE_NOT_CONSTRUCTIBLE` gap (`~/git/nq/docs/gaps/TESTIMONY_OBSERVABLE_NOT_CONSTRUCTIBLE_GAP.md`, filed 2026-05-07). TONC names the wire-boundary problem ("anyone with the shape can mint testimony"); this gap is the production-side answer for one specific class of producer (witness emitters under nq-witness profiles).

Cross-system pattern (preserve the family resemblance — these converged independently):

| AG | NQ | nq-witness |
|----|----|-----------|
| `AuthorizationVerdict` (defined, no minter) | `Finding` (type-sealed, no Deserialize) | `WitnessReport` (today: hand-rolled JSON; future: typed mint) |
| Authority observable, not constructible | Testimony observable, not constructible | Schema possession is not testimonial standing |
| Validator-only-factory | Type-system seal | Library-mediated mint + consumer validator |

The keeper translates across all three; the implementation differs because the substrate differs (Lean kernel; Rust type system; cross-language library). Filing as a sibling preserves the independent-derivation signal and lets each substrate evolve its own close-out.

This gap **does not absorb** TONC and is **not absorbed by** TONC:

- TONC names the wire boundary. Even with a library-mediated mint, anyone hand-rolling JSON can produce a shape-conformant report — the library is not a seal at the wire layer.
- This gap names the construction discipline. Even with a perfect wire-boundary seal (signing, attestation, whatever future work lands), the producer-side construction story would still be "hand-roll vs library" without this discipline.

Both are needed. Neither replaces the other.

## What Already Exists

Falsification grep at filing time:

| Component | Location | Construction discipline today |
|-----------|----------|------------------------------|
| Witness contract | `SPEC.md` (this repo) | Semantic, transport-agnostic, language-agnostic by design. `collection_mode: embedded` already declared as a valid mode. |
| ZFS profile | `profiles/zfs.md` | Schema definition for `nq.witness.zfs.v0`. No library; reference impl is bash. |
| SMART profile | `profiles/smart.md` | Schema definition for `nq.witness.smart.v0`. No library; reference impl is bash. |
| ZFS reference implementation | `examples/nq-zfs-witness` | Hand-rolled JSON in bash. |
| SMART reference implementation | `examples/nq-smart-witness` | Hand-rolled JSON in bash. |
| NQ consumer (ZFS) | `~/git/nq/crates/nq/src/collect/zfs.rs` | Local shape-trust parsing — receives JSON, validates `schema` and `witness.profile_version` strings, then deserializes into the consumer's own structs. No shared validator with the producer side. |
| NQ consumer (SMART) | `~/git/nq/crates/nq/src/collect/smart.rs` | Same shape as ZFS consumer — local parsing, no shared validator. |

The grep evidence is the cheapest available falsification. If a future audit finds a typed report builder shared between producer and consumer paths, this gap's "library does not yet exist" claim is wrong and the spec should be rewritten or retired.

## Laundering Vector

State carefully, mirroring TONC's framing:

- **Hand-rolled JSON is correct as-is for one-off operator tools and reference implementations.** The bash reference impls are fine; their job is to demonstrate the contract, not to be the substrate every native application builds on.
- **Profile schemas being shape-checked at the consumer is correct as-is for V1.** NQ's `schema` and `witness.profile_version` verification at ingestion time is the right floor.
- **The laundering vector lives at the construction discipline boundary**, in two distinct shapes:

### Vector A — Native application authors hand-roll JSON

A Python or Rust application embeds witness emission. Without a library, the author copies the bash reference impl's JSON shape into their language's serializer. Mistakes (missing required fields, wrong coverage tag spelling, profile-version drift, observations with required-field gaps) ship as runtime data and are caught at the consumer if at all. The wire boundary catches structural failures; semantic profile failures (declaring `vdev_state` coverage but emitting observations missing `vdev_path` — see OPEN_ISSUES #2) cross the boundary as valid-shaped JSON.

### Vector B — Consumer-side validator drift

Today NQ's witness consumers parse JSON and trust shape-plus-version-string. Tomorrow, a richer validator (per-profile field requirements, coverage-vocabulary enforcement, `authoritative_for` matching `can_testify`, partial-vs-failed invariants) lands somewhere — either in NQ, in nq-witness, or as duplicated logic in both. Without a single shared validator crate, the meaning of "valid witness report" drifts between repos. Six months later, NQ accepts reports the canonical validator would reject, and nq-witness's reference impls produce reports NQ rejects. The contract is the JSON shape; the *interpretation* of that contract drifts.

### The vocabulary collapse

| Question | Answering surface today |
|----------|------------------------|
| Does this JSON match the witness schema? | Consumer-side schema string check |
| Does this JSON match its declared profile version? | Consumer-side profile-version string check |
| Were the observations constructed under a profile-bound builder? | **No production answering surface.** Hand-rolled JSON is structurally indistinguishable from library-built JSON. |
| Does the consumer apply the same validation rules the canonical mint would have applied? | **No production answering surface.** Each consumer reimplements its own subset. |

The third and fourth rows are where consumers transition from "received some bytes that match the contract by shape" to "trust this as conforming testimony." Today that transition rests on author discipline and per-consumer parsing implementations.

## Doctrine (proposed; not yet ratified)

Both keepers from §"Keepers" above, plus:

> **Library-first, transport-later.** The mint is a construction library. Transport (stdout, HTTP, socket, file) is the consumer's problem. The same canonical report serializes across all transports.

> **The same library/crate serves both producer-side construction and consumer-side validation.** NQ ingestion uses the validator path of the same crate that mints reports on the producer side. "Valid witness report" means the same thing in nq-witness and NQ by construction.

> **`mint` block metadata is admission-side, not construction discipline.** Anyone hand-rolling JSON can write `"mint": {...}` and lie. Consumers MAY treat missing `mint` metadata as a policy tripwire; consumers MUST NOT treat present `mint` metadata as proof of honest construction. Schema possession is not testimonial standing — and neither is mint-metadata possession.

The guardrails (load-bearing — do not lose):

> **The fix is not to seal hand-rolled JSON out of the wire boundary.** Reference implementations may continue to be bash. Operators may continue to hand-craft witness reports for one-off purposes. The library exists to make valid construction cheap, not to make hand-rolling impossible.

> **The fix is not to ratify a particular signing/attestation scheme by this filing.** Signing/attestation is the wire-boundary half of the discipline; this gap is the construction-side. Both are deferred until forcing cases promote.

> **The fix is not to fix `mint.attestation` as a v0 schema field.** Reserve the slot conceptually; do not write it into the schema until something concrete needs to land there.

## Acceptance Criteria

This gap is closed when a doctrine record exists that:

1. States the construction-discipline rule: native applications emitting NQ witness testimony should construct reports through a profile-bound library, not by hand-rolling canonical JSON. Hand-rolling remains permitted for reference impls and one-off tools; the library exists to make valid construction cheap.
2. States the dual-role rule: the same crate/library serves both producer-side construction (mint) and consumer-side validation (NQ ingestion). "Valid witness report" means the same thing in nq-witness and NQ by sharing the validator implementation, not by independent reimplementation.
3. Names the two laundering vectors: (a) native authors hand-roll JSON without library mediation; (b) consumer-side validator drifts away from canonical mint validation.
4. Explicitly preserves the guardrails: hand-rolled JSON is permitted; no signing/attestation is ratified; `mint` metadata is admission-side not seal.
5. Identifies the future work that would close the gap mechanically (deferred, not ratified by this filing): see §"V1 Candidate Slice" below.
6. Records that no crate creation, no schema migration, and no consumer-side enforcement is ratified by the doctrine record itself.
7. Identifies forcing cases that would justify promotion to implementation: see §"Forcing Cases" below.

## V1 Candidate Slice (deferred)

Sketch only. Implementation requires a forcing case beyond the audit-witness discovery. Candidate ratification path if forced:

**Crate / package.** Extract a Rust crate, candidate name `nq-witness-mint` (alternatives: `nq-witness-validator`, `nq-witness-kernel`, `nq-witness-contract`). The name does doctrinal work; the gap doc itself is where any rename argument should land if a better candidate appears. Open question — see below.

**Crate responsibilities (producer side):**

- Schema constants (`schema = "nq.witness.v0"`, profile version registry).
- Profile-bound report builder: typed `ReportBuilder` that requires profile selection, coverage declaration, status, witness identity. Cannot construct a `WitnessReport` value through a public struct literal — only through `ReportBuilder::finish()`.
- Coverage vocabulary validation: `can_testify` tags must be drawn from the selected profile's declared vocabulary; `authoritative_for` claims must have matching coverage; `inadmissible_for` always includes `authorization` and `remediation`.
- Standing validation: `partial` reports must carry errors; `failed` coverage must not appear in `can_testify`; observations whose required coverage was demoted must not be emitted.
- Canonical JSON serialization (deterministic field order, RFC 3339 timestamps, integer durations).

**Crate responsibilities (consumer side):**

- Untrusted JSON parse path: `UntrustedWitnessReport::from_str(&raw_json)` returns a parsed-but-not-validated value. Distinct type from `WitnessReport`.
- Validation entry point: `untrusted.validate_against(profile) -> Result<WitnessReport, ValidationError>`. Same validation rules the producer side enforces — single source of truth.
- Exposed for NQ ingestion: `~/git/nq/crates/nq/src/collect/zfs.rs` and `smart.rs` parse via `UntrustedWitnessReport`, then call `validate_against`. Local shape-trust parsing goes away.

**Profile crates (separate from core):**

- `nq-witness-smart` — SMART profile observation builders, coverage tag constants, validation rules specific to the SMART profile.
- `nq-witness-zfs` — same for ZFS.
- These depend on the core crate; the core crate does not depend on them.

**Mint metadata (V1 schema):**

V1 ships with **only** `mint.origin` and `mint.version`:

```json
"mint": {
  "origin": "nq-witness-mint/rust",
  "version": "0.1.0"
}
```

Reserved-but-not-shaped (open questions, not v0 fields): `mint.profile_validator`, `mint.validation_mode`, `mint.attestation`. Adding these now would be admission-block bloat; ship the minimum and let forcing cases shape the rest. Per the recurring NQ doctrine: "five-state status enums that need to be hand-maintained also rot." Two-field admission block, expanded only on forcing case.

**Consumer rules (policy, not structural):**

- Consumers MAY reject reports lacking `mint.origin` (policy tripwire).
- Consumers MUST NOT treat `mint.origin` as proof of honest construction.
- Validation always runs against `witness.profile_version`, not against any mint-side claim about which validator was used.

**Reference implementations (later, not V1):**

Wrapping or rewriting the existing bash reference impls is **not** required by V1. The existing scripts continue to produce conforming reports without library mediation. If a future slice rewrites them in a native language, they become exemplars of library-mediated construction; until then, they are operator-readable bash.

**Python parallel (open question, not V1):**

Python authors writing witnesses on infrastructure NQ trusts as testimony is exactly the laundering surface. Python cannot seal the construction discipline (frozen ≠ sealed; consenting-adults language semantics). Two postures, both deferred to a Python-forcing case:

- Make laundering visible: frozen dataclasses, ReportBuilder with private constructors, loud-named escape hatches (`UnsafeWitnessReport.from_mapping_without_validation`). Greppable sin.
- Runtime-detectable laundering at the consumer boundary: open question about whether the producer can carry a runtime flag the consumer can read.

V1 ships Rust only. Python lands when forced.

## Cross-Repo Coordination

V1 implementation involves three coordinated changes across two repos:

1. Extract `nq-witness-mint` crate (this repo, `rust/nq-witness-mint/` or top-level `nq-witness-mint/`).
2. Update `~/git/nq/crates/nq/src/collect/{zfs,smart}.rs` to consume the validator. NQ depends on `nq-witness-mint` via path / git URL / published crate (open question).
3. Optionally extract profile crates (`nq-witness-smart`, `nq-witness-zfs`); optionally update bash reference impls (later, not V1).

The "narrow first slice" framing is correct in scope, not in coordination cost. V1 is two-repo work even when narrowly scoped.

## Repo Layout (proposed; not yet ratified)

If V1 ships, this repo gains:

```
SPEC.md
profiles/
  zfs.md
  smart.md
fixtures/                    # canonical happy/partial/failed reports
  zfs/
  smart/
rust/
  nq-witness-mint/           # core crate
  nq-witness-smart/          # SMART profile crate
  nq-witness-zfs/            # ZFS profile crate
examples/                    # bash reference impls (unchanged)
  nq-zfs-witness
  nq-smart-witness
docs/
  gaps/
    LIBRARY_NATIVE_WITNESS_GAP.md  # this file
```

Python packages (`python/nq_witness/`, `python/nq_witness_smart/`, etc.) land separately when Python becomes a forcing case.

The bash reference impls become **reference frontends**, not the conceptual center. The conceptual center moves to the typed mint.

## Non-Goals

Non-goals are load-bearing here:

- **Not implementing a signing scheme.** Signing/attestation is the wire-boundary half of the discipline (see TONC). This gap is the construction-side; signing is deferred.
- **Not making the existing bash reference impls library-only.** Hand-rolled JSON remains permitted. The library is for native applications and shared validation, not for replacing operator-readable scripts.
- **Not a daemon, HTTP listener, or sidecar.** The library is library-first, transport-later. Transport is the caller's choice; the library produces canonical JSON.
- **Not a plugin ecosystem.** No dynamic profile loading, no third-party profile registries, no "ecosystem" anything. Profile crates are first-party; new profiles ship as new crates in this repo.
- **Not a generic fact-ingestion library.** The library produces and validates witness reports under declared profiles. Anything outside that shape is out of scope.
- **Not a Python parallel in V1.** Python lands on a Python-forcing case.
- **Not `mint.attestation` as a V1 schema field.** Reserved conceptually; not in the V1 wire shape.
- **Not `mint.profile_validator` as a V1 schema field.** Consumers always run their own validator against `witness.profile_version`. If divergence becomes a forcing case (mint claims validator vX, consumer at vY), promote then.
- **Not trust in producer-claimed `validation_mode` strings.** Anyone hand-rolling can write `"validation_mode": "strict"`. The string is admission, not seal.
- **Not arbitrary observation dicts as first-class testimony.** Observations are profile-typed; ad-hoc fields are not load-bearing testimony.
- **Not a replacement for TESTIMONY_OBSERVABLE_NOT_CONSTRUCTIBLE.** Sibling, not absorbing.

## Forcing Cases

Promotion to V1 implementation justified when:

1. **A native application wants to emit witness testimony.** First Python or Rust application that needs to be a witness without copying the bash reference impl's JSON shape forces the library.
2. **A third profile lands.** ZFS + SMART + (procfs / sqlite_health / log_silence / whatever next) → the shared report-construction code that doesn't exist becomes hand-duplicated. Library extracts cleanly.
3. **NQ's consumer-side validator grows past local shape-trust.** First time NQ wants to enforce profile-level invariants (coverage-tag-vocabulary checks, `authoritative_for` matching `can_testify`, etc.) at ingestion, the validator wants to live in a shared crate rather than a NQ-internal module.
4. **Validator drift between nq-witness and NQ becomes operationally visible.** A report nq-witness fixtures consider valid is rejected by NQ ingestion, or vice versa. Drift detected → shared crate forces a fix.
5. **A laundered-witness incident.** Postmortem traces back to a hand-rolled-JSON producer that nobody noticed was emitting subtly-malformed reports for weeks.

None of these have fired. None should be assumed-imminent. This is preemptive naming — name the surface, watch for the case.

## Open Questions

1. **Crate name.** `nq-witness-mint` makes the doctrinal role legible at the import line ("the mint mints testimony, not truth"). Alternatives: `nq-witness-validator` (consumer-skewed), `nq-witness-kernel` (more sober, possibly grand), `nq-witness-contract` (accurate, dull). The doc could argue for a better name during V1 implementation.
2. **Repo placement.** Crate inside `nq-witness/rust/nq-witness-mint/` or as a separate repo? Inside-this-repo simplifies cross-profile work; separate repo simplifies external dependencies. Lean: inside this repo for V1.
3. **NQ dependency style.** Path dependency (developer setup), git URL dependency (works without crates.io), or published crate (cleanest but requires publishing). Lean: path or git for V1, published when stable.
4. **Python parallel.** When does Python become a forcing case? Lean: not until a Python application actually needs to be a witness. Until then, Rust-only.
5. **`mint.origin` versioning.** When the mint version bumps and the wire format does not change, do reports still need to advertise the new mint version? Lean: yes, but consumers should not branch on it. Open until forced.
6. **Bash reference impl future.** Stay as bash forever, or eventually rewrite as native exemplars of library-mediated construction? Lean: stay as bash until a native rewrite has its own forcing case.
7. **Profile crate boundaries.** SMART and ZFS each get their own crate, or do they share `nq-witness-profiles`? Lean: separate crates per profile — different domain expertise, different versioning cadence.
8. **What does "validation mode" mean operationally?** chatty's earlier sketches had `validation_mode: "strict" | "permissive"` on the mint block. V1 deliberately omits this — strict-by-default is the only shape the library supports. If permissive mode is a real need, it'll appear as a forcing case; for now, no permissive escape hatch in the library.

## Provenance

Filed 2026-05-07 as a sidebar to a NQ session that closed eight legacy gap-status ratifications and shipped the REGIME_FEATURES V1.6 observability slice. The operator returned from a separate language-audit conversation (`chatty`/ChatGPT) where the cross-language sealing-discipline question had just been worked through (Ada, Erlang, Rust, Python comparison). The library-native witness idea fell out of that audit's recognition that:

- nq-witness already says the contract is semantic, not transport-specific (SPEC.md);
- two reference impls now exist (ZFS, SMART) and both hand-roll JSON;
- `collection_mode: embedded` is already a valid mode in the spec;
- the `nq-witness` repo is one profile away from forcing the shared-construction-code question;
- TESTIMONY_OBSERVABLE_NOT_CONSTRUCTIBLE filed earlier in the same NQ session names the consumer-side / wire-boundary version of the same construction discipline.

Filing as a sibling preserves the cross-system convergence (AG → NQ TONC → nq-witness library mint) without conflating the wire-boundary and construction-side versions of the discipline.

The keeper, lifted from the chatty conversation:

> **The mint can make valid construction cheap. Only the consumer validator can make shape admissible.**
