# Kea DHCP Witness Profile (v0)

**Profile version:** `nq.witness.kea_dhcp.v0`
**Applies to:** ISC Kea DHCPv4 / DHCPv6 (`kea-dhcp4`, `kea-dhcp6`), observed via the memfile lease store, the control socket / control-agent API (`lease4-get`, `lease4-get-all`, `stat-lease4-get`, `config-get`, `version-get`), or an equivalent backend. Field shapes captured from a real `kea-dhcp4` 2.2.0 instance (memfile + `lease_cmds` hook).
**Forcing case:** Kea sits at the intersection of address assignment, client identifier, subnet/pool scope, and DNS-update *intent*. A lease is identity-heavy testimony — a Prometheus lease-count gauge cannot carry the lease/reservation/DDNS-intent identity or the refusal boundary. The witness must keep DHCP from laundering DNS, and DNS from laundering host identity.

## Required `coverage.can_testify` tags

A conforming Kea witness MUST testify about whichever of these its backend can source, and MUST list the rest under `cannot_testify` (never omit, never silence):

- `daemon_state` — control/API reachability + per-daemon (v4/v6) up/responding state
- `lease_identity` — per-lease address + client identifier (hwaddr / client-id / DUID) + hostname
- `lease_lifecycle` — observed lease state (`default` / `declined` / `expired-reclaimed`) + valid-lifetime + absolute expiry
- `subnet_pool_identity` — subnet-id and pool-id scope for observed leases/reservations

## Optional `coverage.can_testify` tags

- `reservation_identity` — static host reservations (address / identifier / hostname)
- `pool_utilization` — assigned vs declared addresses per subnet/pool (`stat-lease4-get`)
- `ddns_intent` — per-lease forward/reverse DDNS *intent* flags (`fqdn_fwd` / `fqdn_rev`) the DHCP server would hand to D2
- `d2_state` — kea-dhcp-ddns (D2) daemon reachability/state, where exposed
- `ha_peer_state` — High-Availability peer/role state, where configured

## Required observation kinds

### `kea_daemon`

One observation per Kea daemon (or the control agent fronting them).

```json
{
  "kind": "kea_daemon",
  "subject": "kea-dhcp4@example-host",
  "daemon": "dhcp4",
  "control_channel": "unix:/run/kea/kea4-ctrl-socket",
  "reachable": true,
  "version": "2.2.0",
  "observed_at": "2026-06-29T12:00:00Z"
}
```

- `daemon` — `dhcp4`, `dhcp6`, `d2`, or `ctrl-agent`
- `reachable` — the control channel / API answered this cycle (`version-get` / socket connect). `reachable: false` is honest testimony, not an outage claim.
- `version` — daemon version when sourced (`version-get`), else null

### `kea_lease`

One observation per current lease (memfile: the LAST row per address; API: `lease4-get-all`). Required fields:

```json
{
  "kind": "kea_lease",
  "subject": "10.99.0.100",
  "family": 4,
  "subnet_id": 1,
  "hw_address": "08:00:27:aa:bb:01",
  "client_id": null,
  "duid": null,
  "hostname": "lab-host-active",
  "lease_state": "default",
  "valid_lifetime_s": 3600,
  "expire_at": "2026-07-01T12:22:34Z",
  "fqdn_fwd": false,
  "fqdn_rev": false
}
```

- `subject` — leased address (stable lease identity key with `subnet_id`)
- `family` — `4` or `6`
- `hw_address` / `client_id` / `duid` — whichever identifiers the backend exposes (any may be null)
- `lease_state` — `default`, `declined`, or `expired-reclaimed` (the Kea lease-machine state; **independent of expiry** — a `default` lease may already be lapsed by `expire_at`, which the consumer evaluates against its own clock)
- `valid_lifetime_s` — lease valid lifetime in seconds
- `expire_at` — absolute UTC expiry (memfile `expire`; API `cltt + valid-lft`)
- `fqdn_fwd` / `fqdn_rev` — DDNS *intent* flags carried on the lease (NOT confirmation that DNS contains a record)

**This is the section a Prometheus exporter cannot honestly provide:** lease *identity* (address ↔ identifier ↔ hostname ↔ subnet) plus the declined/expired-reclaimed state distinction. A lease-count gauge flattens all of it.

### `kea_subnet` (required when `subnet_pool_identity` is claimed)

One observation per observed subnet/pool.

```json
{
  "kind": "kea_subnet",
  "subject": "subnet:1",
  "family": 4,
  "subnet_id": 1,
  "pool_id": 0,
  "prefix": "10.99.0.0/24",
  "addresses_declared": 101,
  "addresses_assigned": 3
}
```

- `addresses_declared` / `addresses_assigned` — from `stat-lease4-get` when available; both null if not sourced (then `pool_utilization` belongs in `cannot_testify`)

### `kea_reservation` (when `reservation_identity` is claimed)

```json
{
  "kind": "kea_reservation",
  "subject": "resv:1:08:00:27:aa:bb:09",
  "subnet_id": 1,
  "identifier_type": "hw-address",
  "identifier": "08:00:27:aa:bb:09",
  "reserved_address": "10.99.0.50",
  "hostname": "printer-1"
}
```

## Standing defaults for Kea witnesses

```json
{
  "authoritative_for": [
    "current_lease_state",
    "lease_identity",
    "observed_lease_expiry",
    "subnet_pool_identity",
    "reservation_identity",
    "daemon_reachable_state",
    "ddns_intent_flags"
  ],
  "advisory_for": [
    "pool_utilization_pressure"
  ],
  "inadmissible_for": [
    "client_connectivity",
    "client_identity_ownership",
    "host_authorization",
    "address_reachability",
    "dns_correctness",
    "reverse_dns_correctness",
    "renewal_prediction",
    "network_health",
    "remediation",
    "failover_or_page"
  ]
}
```

`pool_utilization_pressure` is advisory, not authoritative: a single snapshot reports assigned/declared counts; whether that constitutes "pressure" is NQ's cross-generation trajectory job, not the witness's.

## What a Kea witness MUST refuse (`cannot_testify` is not optional here)

A lease is an assignment record, not a fact about the world. The Kea witness refuses, always:

- **client has working connectivity** — a lease is not a ping
- **client is who the lease says** — `hostname` / identifier are client-supplied or operator-configured, not verified identity
- **the address is reachable** — assignment ≠ reachability
- **DNS contains a matching A/AAAA record** — `fqdn_fwd` is *intent*; DNS testimony belongs to `dns_state`
- **DNS contains a matching PTR record** — `fqdn_rev` is *intent*; reverse-DNS is `dns_state`'s, never inferred from a lease
- **the host is authorized** — DHCP assigns; it does not authorize
- **renewal will succeed** — future tense is never witnessed
- **the network is healthy** — out of jurisdiction
- **remediation / failover / page** — consequence is the claim/operator layer's

A memfile-only backend additionally lists under `cannot_testify`: `daemon_state` (no live API), `pool_utilization` (no `stat-*`), `reservation_identity` (memfile holds leases, not reservations), `d2_state`, `ha_peer_state` — it must not pretend coverage it cannot source.

## Future composite (named, NOT implemented in v0)

`dhcp_dns_identity_consistency` — a *composite* claim that may later say only: "Kea lease/reservation testimony and DNS response testimony agree for tuple X (hostname H ↔ address A, forward and PTR) from vantage V at times T1/T2." It still refuses host identity, reachability, global DNS correctness, and authorization. It is the *only* sanctioned place DHCP and DNS testimony meet — and it composes two witnesses, it never lets DHCP mint DNS or DNS mint identity. Recorded here so a future session builds the fence, not the laundromat.

## Freshness defaults

- **Expected collection cadence:** 60 seconds. Lease state changes on client activity, not microseconds.
- **Stale threshold:** 5 minutes without a fresh report → consumer emits `nq_witness_silent` for the Kea witness.
- **Collection timeout:** 5 seconds for the control-socket path; memfile reads are near-instant.

## Privilege model recommendations

1. **`nopasswd_fixed_helper`** — root-owned helper that reads the control socket (`/run/kea/*-ctrl-socket`) or the memfile and emits canonical JSON. Recommended.
2. **`root_exporter_localhost`** — a long-running collector with control-socket access bound to `127.0.0.1`, exposing canonical JSON (Prometheus projection optional, not source of truth).
3. **`unprivileged`** — acceptable only where the memfile / control socket is group-readable to the witness user; must still declare coverage honestly.

## Example reports

### Control-socket backend — full coverage

```json
{
  "schema": "nq.witness.v0",
  "witness": {
    "id": "kea_dhcp.local.example-host",
    "type": "kea_dhcp",
    "host": "example-host",
    "profile_version": "nq.witness.kea_dhcp.v0",
    "collection_mode": "sudo_helper",
    "privilege_model": "nopasswd_fixed_helper",
    "collected_at": "2026-06-29T12:00:00Z",
    "duration_ms": 41,
    "status": "ok"
  },
  "coverage": {
    "can_testify": ["daemon_state", "lease_identity", "lease_lifecycle", "subnet_pool_identity", "pool_utilization", "ddns_intent"],
    "cannot_testify": ["reservation_identity", "d2_state", "ha_peer_state"]
  },
  "standing": {
    "authoritative_for": ["current_lease_state", "lease_identity", "observed_lease_expiry", "subnet_pool_identity", "daemon_reachable_state", "ddns_intent_flags"],
    "advisory_for": ["pool_utilization_pressure"],
    "inadmissible_for": ["client_connectivity", "client_identity_ownership", "host_authorization", "address_reachability", "dns_correctness", "reverse_dns_correctness", "renewal_prediction", "network_health", "remediation", "failover_or_page"]
  },
  "observations": [
    {"kind": "kea_daemon", "subject": "kea-dhcp4@example-host", "daemon": "dhcp4", "control_channel": "unix:/run/kea/kea4-ctrl-socket", "reachable": true, "version": "2.2.0"},
    {"kind": "kea_lease", "subject": "10.99.0.100", "family": 4, "subnet_id": 1, "hw_address": "08:00:27:aa:bb:01", "client_id": null, "duid": null, "hostname": "lab-host-active", "lease_state": "default", "valid_lifetime_s": 3600, "expire_at": "2026-07-01T12:22:34Z", "fqdn_fwd": false, "fqdn_rev": false},
    {"kind": "kea_lease", "subject": "10.99.0.102", "family": 4, "subnet_id": 1, "hw_address": "08:00:27:aa:bb:03", "client_id": null, "duid": null, "hostname": "lab-host-declined", "lease_state": "declined", "valid_lifetime_s": 3600, "expire_at": "2026-07-01T12:22:34Z", "fqdn_fwd": false, "fqdn_rev": false},
    {"kind": "kea_subnet", "subject": "subnet:1", "family": 4, "subnet_id": 1, "pool_id": 0, "prefix": "10.99.0.0/24", "addresses_declared": 101, "addresses_assigned": 3}
  ],
  "errors": []
}
```

### Memfile-only backend — partial coverage (honest demotion)

```json
{
  "schema": "nq.witness.v0",
  "witness": {
    "id": "kea_dhcp.local.example-host",
    "type": "kea_dhcp",
    "host": "example-host",
    "profile_version": "nq.witness.kea_dhcp.v0",
    "collection_mode": "subprocess",
    "privilege_model": "unprivileged",
    "collected_at": "2026-06-29T12:00:00Z",
    "duration_ms": 3,
    "status": "partial"
  },
  "coverage": {
    "can_testify": ["lease_identity", "lease_lifecycle"],
    "cannot_testify": ["daemon_state", "subnet_pool_identity", "pool_utilization", "reservation_identity", "ddns_intent", "d2_state", "ha_peer_state"]
  },
  "standing": {
    "authoritative_for": ["current_lease_state", "lease_identity", "observed_lease_expiry"],
    "advisory_for": [],
    "inadmissible_for": ["client_connectivity", "client_identity_ownership", "host_authorization", "address_reachability", "dns_correctness", "reverse_dns_correctness", "renewal_prediction", "network_health", "remediation", "failover_or_page"]
  },
  "observations": [
    {"kind": "kea_lease", "subject": "10.99.0.100", "family": 4, "subnet_id": 1, "hw_address": "08:00:27:aa:bb:01", "client_id": null, "duid": null, "hostname": "lab-host-active", "lease_state": "default", "valid_lifetime_s": 3600, "expire_at": "2026-07-01T12:22:34Z", "fqdn_fwd": false, "fqdn_rev": false}
  ],
  "errors": [
    {"kind": "control_socket_absent", "detail": "no control socket configured for this collector; memfile leases only — daemon/subnet/reservation/DDNS coverage demoted to cannot_testify", "observed_at": "2026-06-29T12:00:00Z"}
  ]
}
```

This is the memfile reader NQ already ships (`nq-monitor` `parse_kea_leases`, lab-backed compatibility): real lease identity + lifecycle, everything else honestly demoted. It is the first backend adapter for this profile; the control-socket backend extends coverage, it does not change the contract.

## Not required in v0

- DHCPv6 prefix delegation (PD) detail beyond lease identity
- HA state-machine transitions / heartbeat history
- Hook-library inventory
- Per-class / per-client-class statistics
- Config-diff testimony
- D2 NCR queue depth

Add in v1+ if a concrete claim forces them.

## Conformance

A Kea witness conforms when:

1. It emits the canonical `nq.witness.v0` report with `profile_version: "nq.witness.kea_dhcp.v0"`.
2. It provides the required `coverage.can_testify` tags its backend can source, and honestly lists the rest in `cannot_testify`.
3. It emits `kea_lease` (and `kea_daemon` when an API is reached) observations with the required fields; the `lease_state`-vs-`expire_at` independence is preserved.
4. Its `standing.inadmissible_for` carries the full refusal set above — connectivity, identity ownership, DNS/reverse-DNS correctness, authorization, renewal, network health, remediation.
5. It respects the freshness/failure rules from SPEC.md and never upgrades DDNS *intent* into a DNS-content claim.

A Prometheus exporter exposing lease counts / pool utilization is a useful coarse source; it is NOT a conforming Kea witness — it carries no lease identity, no declined/expired-reclaimed distinction, and no standing/refusal declaration.
