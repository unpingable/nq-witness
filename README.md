# nq-witness

NQ-native witness reports for structured operational evidence.

Prometheus exporters emit samples. NQ witnesses emit bounded reports: what was observed, what the witness can testify about, what it cannot see, and how fresh the evidence is.

Witnesses do not make decisions. They provide evidence for NQ detectors.

## Role

| Layer | Role |
|---|---|
| **Witness** | observes domain facts (ZFS, SQLite, /proc, SMART, etc.) |
| **NQ** | classifies findings and regimes |
| **Night Shift** | prepares review packets |
| **Governor** | authorizes force |
| **Human** | owns standing |

## Why this exists

Off-the-shelf Prometheus exporters provide visibility. They do not automatically provide standing.

A concrete example: `pdf/zfs_exporter` exposes pool health, capacity, fragmentation, and dataset statistics. It does not expose per-vdev state, per-vdev error counts, scrub completion, or spare activation. You can tell a pool is degraded. You cannot tell whether the degradation is stable or worsening.

For chronic-condition monitoring — a known-failed drive in a pool with spares, for instance — that's the difference between coarse liveness and operationally useful evidence.

NQ witnesses close the gap by emitting structured reports that declare their own coverage, standing, and freshness. The canonical output is JSON. Prometheus is an optional projection, not the source of truth.

## Core distinctions

- **Visibility vs standing** — an exporter can report a fact without being the authoritative source for that fact. Witnesses declare which is which.
- **Evidence vs decision** — witnesses provide admissible evidence; detectors and downstream systems (NQ, Night Shift, Governor) decide what to do with it.
- **Privilege for observation vs privilege for action** — witnesses may run with elevated read access, but they never mutate state. Privilege increases visibility; it does not increase authority.

## Repository layout

```
SPEC.md              # witness contract (schema, coverage, standing, freshness, failure behavior)
profiles/
  zfs.md             # ZFS witness profile — first concrete profile
examples/            # synthetic fixtures and reference outputs
```

No SDK yet. The contract comes first. Implementations will follow when a second profile forces common code to exist.

## Status

Spec-first. Initial profile: ZFS. No implementation in this repo.

The first ZFS witness implementation may live as a helper script on the monitored host (see `profiles/zfs.md` for the required fields). A Python or Rust library emerges later if multiple witness profiles need shared scaffolding.

## License

Apache-2.0.
