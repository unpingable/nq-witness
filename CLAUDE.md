# CLAUDE.md — nq-witness

Repo-specific context for any Claude session working in this repo.

## What this is

A specification repo for NQ-native witness reports. Structured operational evidence with declared coverage, freshness, and limits. Prometheus exporters emit samples; witnesses emit bounded admissible reports.

Spec-first. No SDK yet. First concrete profile: ZFS.

## Relationship to other repos

- `~/git/notquery` (public: `unpingable/nq`) — the observatory that consumes witness reports. NQ's `ZFS_COLLECTOR_GAP` references this repo for the ZFS witness contract.
- `~/git/scheduler` (Night Shift) — downstream consumer via NQ's `FINDING_EXPORT` contract.
- `~/git/agent_gov` (Governor) — authority layer; separate concern.

Witness modules are independent of all three. They provide evidence. NQ classifies. Night Shift proposes. Governor authorizes.

## Writing discipline

**Provenance scrub is pre-commit, not post-commit.**

This repo is personal open source. The operator has a hard provenance guardrail: no employer names, internal tool names, internal jargon, or project codenames in committed text. Generic equivalents work for everything ("runbook-style", "mature ops tooling", "observatory pattern"). When ingesting prose from collaborators (AI agents, conversation dumps), treat it as unsanitized — strip identifying terms before incorporating.

Reason: 2026-04-15 incident in `~/git/notquery` had employer-name references propagate through collaborator prose into public commits. Required emergency history rewrite + repo recreate. Don't repeat that here.

Scan every draft before staging. The scrub pattern is maintained in the operator's auto-memory (`feedback_provenance.md`). Run it against each new file before `git add`. Empty output means clean. Extend the scan terms whenever the operator mentions a new prior employer or adjacent vendor.

## Push discipline

The operator pushes to public GitHub only after work hours. Claude does NOT push unless given explicit instruction that includes awareness of the timing.

Local commits are fine anytime.

## Design discipline

Match the NQ aesthetic:

- Brutalist, not liturgical. No ceremony.
- Name what the thing is and what it refuses to be.
- `cannot_testify` / non-goals sections carry as much weight as feature lists.
- No metaphor budget for "scars," "body horror," "control-theory cosplay."
- Greek-letter failure domains (Δ-codes) belong to the NQ cybernetic taxonomy; this repo is adjacent and doesn't coin its own.

## Core invariants (reference)

1. Privilege may increase visibility. It must not increase authority.
2. Witnesses produce evidence for detectors. They do not produce decisions.
3. Prometheus output is a projection, not the source of truth. A witness without canonical JSON is a legacy exporter, not a witness.
4. `cannot_testify` is a first-class field. A witness that doesn't say what it can't see is incomplete.

## When adding a new profile

1. Draft `profiles/<name>.md` following the ZFS profile shape.
2. Add the profile to the SPEC.md "Profiles" section.
3. Include a chronic-case + partial-failure example report.
4. Do not ship implementation code in this repo yet. Implementations live where they run (on monitored hosts, in deploy/, etc.) until a second profile forces common code to exist.
