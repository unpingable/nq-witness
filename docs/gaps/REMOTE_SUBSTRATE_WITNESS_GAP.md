# Gap: Remote Substrate Witness — producer-family pattern for online corpora

**Status:** candidate gap — name the surface, do not build yet. **No crate creation, no concrete witness family (page-drift, DNS, package-registry, CT log, etc.) is ratified by this filing.** Future forcing cases promote individual witness families one at a time.
**Depends on:** NQ's `DURABLE_ARTIFACT_SUBSTRATE_GAP` (V1 shipped 2026-05-12 — admits corpus-bound, extraction-derived testimony as a witnessable substrate class; remote/online corpora are explicitly in scope per its §"Remote / online corpora are in scope" note).
**Build phase:** doctrine — names a producer-family pattern, not a build slice.
**Blocks:** any future remote-substrate witness implementation (page drift, package registry, DNS, certificate transparency, RSS feed, docket, API resource, repository metadata, etc.) that would otherwise hand-roll its own ad-hoc testimony shape and ignore the existing inbound contract.
**Last updated:** 2026-05-12

## Keepers

> **A scraper tells you what it saw. A witness tells you what its seeing is worth.**

> **Remote witness is not a new NQ surface. It is a new way to produce admissible testimony for an existing surface.**

The first names the producer-side discipline; the second pins the constitutional boundary. Both reinforce SPEC.md's load-bearing sentence:

> *A backend integration may supply observations, but only a conforming witness may supply testimony.*

## The Problem

Operationally-interesting online state — page drift, package release churn, DNS record changes, certificate transparency entries, public dockets, RSS feeds, API resources, repository metadata — is **corpus-bound** (subject identity is the URL / package / record / endpoint, not a host) and **extraction-derived** (state is observed when the witness fetches, not self-clocked by the substrate). That substrate class is already admitted upstream: NQ's `DURABLE_ARTIFACT_SUBSTRATE_GAP` shipped V1 on 2026-05-12 and explicitly names remote/online corpora as in-scope.

What's not yet named is the **producer-family pattern**: how an `nq-witness`-style library testifies about a remote corpus without becoming the scraper, the parser, the rate-limit accountant, or the inference engine. Without a named pattern, each future remote-witness implementation (page drift first, others later) risks reinventing:

- ad-hoc snapshot conventions
- inconsistent `cannot_testify` fence vocabulary
- producer-specific dependency-health reporting
- one-off retry / rate-limit / cache semantics that smuggle silence-vs-failure conflation past NQ's wire boundary
- witness-side decisions about "what the change means" that should be downstream of the witness

The dangerous default if this is not named:

```text
Application / cron
  → curl + jq + grep
    → emits JSON that happens to match nq.finding_import.v1
      → NQ trusts it because shape good
```

The intended path:

```text
Application / cron
  → conforming remote-substrate witness (this pattern)
    → typed observation + coverage declaration + cannot_testify fence
      → canonical nq.finding_import.v1 manifest
        → NQ inbound contract
          → admissibility surface decides standing
```

The constitutional boundary is the existing one. This gap names the producer-family discipline that lets concrete remote-witness implementations conform to it.

## Design Stance

### Producer family, not a new substrate class

The substrate-class admission already exists upstream. This gap does **not** propose a new NQ-side testimony class. Remote-substrate witnesses emit testimony through `nq.finding_import.v1` (NQ's existing inbound contract from DURABLE_ARTIFACT_SUBSTRATE V1). The wire shape is producer-contract; semantic consumption is downstream. Concrete witness families plug into this pattern.

### Required testimony boundaries

A conforming remote-substrate witness makes explicit:

- **target identity** — URL, DID, package name + version, repo + ref, API endpoint, docket ID, CT log entry ID, etc. Whatever names the corpus.
- **access path** — fetch method, transport, authentication posture, archive vs live, whether a redirect chain was followed.
- **extraction time** — producer-clock RFC3339 UTC; populates `origin.producer_extraction_time` on the NQ wire.
- **snapshot basis** — content hash, normalized representation, retrieved bytes, headers, status. Whatever lets a second witness reproduce or refute.
- **normalization method** — how raw bytes became the compared/diffed representation. Different normalizers produce different "changes."
- **dependency health** — what other observations were required for this testimony to exist (DNS, TLS, HTTP, parser version, rate-limit quota, archive availability). Composes onto NQ's existing testimony-dependency primitives.
- **coverage claims** — what was actually observed (`http_fetch`, `status_code`, `content_hash`, `normalized_text_diff`, `header_set`, etc.).
- **anti-claims (`cannot_testify`)** — what the witness explicitly does not bear standing on (author intent, business reason, whether all users see the same content, whether a change is malicious, whether silence is access failure vs absence-of-change).
- **reproducibility limits** — what would falsify the testimony; under what conditions another witness might observe differently.

The `cannot_testify` fence is the load-bearing piece. It's what keeps a page-drift witness from accidentally becoming a motive-inference engine, a DNS witness from claiming "this propagation looks coordinated," or a package-registry witness from asserting "this maintainer transfer was hostile." Each witness family's fence is narrow on purpose.

### Constitutional boundary

NQ stays the admissibility layer over witnesses. The witness owns:

- HTML / API / DNS / CT / registry normalization
- snapshotting
- rate-limit accounting
- retry / cache / archive semantics
- detection-of-distinguishability-between-silence-and-access-failure
- "did Cloudflare just sneer at us" tactical defensiveness

NQ owns:

- "Is this testimony fresh?"
- "Does the witness's coverage claim hold?"
- "Has the basis been retired?"
- "Is there contradictory testimony from a peer witness?"
- "Is the witness itself silent?"
- "Does this finding compose admissibly with native NQ findings?"

The keeper that pre-existed this filing — *a backend integration may supply observations, but only a conforming witness may supply testimony* — extends naturally: a scraper can supply observations; only a conforming remote-substrate witness can supply admissible testimony about remote corpora.

### One witness family per forcing case

Concrete remote-substrate witness implementations land **one family at a time, each behind its own forcing case**. The candidate list (page drift, package registry, DNS, certificate transparency, RSS feed, docket, API resource, repository metadata) is enumerated here for shape-naming, not as a roadmap. Filing this gap does not authorize any of them.

When a family promotes from candidate to built, it gets its own `profiles/` entry in this repo (matching the existing `profiles/zfs.md`, `profiles/smart.md` pattern) and goes through the same SPEC.md conformance discipline as native-substrate witnesses.

## Forcing cases for promotion (per witness family)

- **Page drift witness** — first concrete need for "policy page changed" / "docs page removed a guarantee" / "API deprecation notice appeared" testimony with admissibility standing past arbitrary curl output.
- **Package registry witness** — first concrete need for "package remains installable but authority/maintenance basis changed" testimony (license change, maintainer transfer, yank, archive/unarchive).
- **DNS / CT / BGP witnesses** — first concrete need for infrastructure-adjacent change testimony where NQ's existing host-bound substrate axes can't carry the claim.
- **Repository-metadata witness** — first concrete need for repo-health testimony that isn't covered by GENERATION_LINEAGE / FLEET_INDEX.
- **Docket / regulatory / institutional witnesses** — first concrete need for public-record change testimony where the substrate is an authoritative external source whose interface drift matters.

Each family stays as a candidate-named member of this gap until its forcing case fires. The gap doc does not pre-implement.

## Non-goals

- **No new NQ-core testimony class.** Remote-substrate witnesses use the existing DURABLE_ARTIFACT inbound contract. If a future witness needs a wire-shape extension, that's a NQ-side gap conversation, not a witness-side one.
- **No claim to monitor "the internet."** The witness pattern is narrow per family. The pattern doesn't authorize a "watch all the things" surface.
- **No scraping framework / fetch library / archive infrastructure.** Each witness family solves its own substrate ugliness. Shared fetch/snapshot/normalization primitives may emerge later if multiple families converge — premature shared-infrastructure is the [premature portability](https://example.invalid) trap.
- **No `remote-witness` repo.** That repo earns existence only when shared fetch/snapshot primitives are genuinely needed across families, or when non-NQ consumers want the library. Today, the candidate witness work lives in `nq-witness` under this gap and its (future) sibling `profiles/` entries.
- **No intent inference, motive attribution, or coordination claim.** Each witness family's `cannot_testify` fence excludes these explicitly. NQ's admissibility surface does not promote behavioral observations into intent claims.
- **No new authority/signing scheme.** Remote-witness testimony inherits the same trust posture as other inbound producers (V1 accepts well-formed manifests as truthful; real-producer multi-host signing is downstream of DURABLE_ARTIFACT V2+).

## References

- NQ: [`docs/gaps/DURABLE_ARTIFACT_SUBSTRATE_GAP.md`](../../../nq/docs/gaps/DURABLE_ARTIFACT_SUBSTRATE_GAP.md) — substrate-class admission this gap is a producer-family for; see §"Remote / online corpora are in scope" for the upstream scope note.
- NQ: [`docs/CONSUMER_DRYRUN.md`](../../../nq/docs/CONSUMER_DRYRUN.md) — consumer-side handoff pattern that mirrors this producer-side filing; same hardened-seam discipline.
- nq-witness: `SPEC.md` — constitutional sentence ("backend integration may supply observations, only a conforming witness may supply testimony") is the discipline this gap extends.
- nq-witness: [`docs/gaps/LIBRARY_NATIVE_WITNESS_GAP.md`](LIBRARY_NATIVE_WITNESS_GAP.md) — sibling producer-side construction-discipline filing (typed mint, shared validator). Composes: a future remote-substrate witness implementation would use the library-native mint pattern to construct conforming reports.
- Doctrine framing 2026-05-12 (James + ChatGPT conversation, NQ-side): *bad velocity collapses boundaries to reduce friction; good velocity hardens boundaries so crossing them becomes cheap.* This gap is the producer-family-side hardening for remote substrates.
