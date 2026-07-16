# piccolo-perf Fleet Install & Prometheus Hardening — Design

Status: Approved for planning
Date: 2026-07-16
Owner: buraglio@forwardingplane.net

## Context

[piccolo-perf](https://github.com/retecolo/piccolo-perf) is a single static Go binary
(TWAMP, bandwidth, traceroute, MTU, DNS measurements) that already ships an `install.sh`
one-liner, systemd units (`deploy/piccolo-perf-agent.service`,
`deploy/piccolo-perf-exporter.service`) with reasonable hardening (`DynamicUser`,
`NoNewPrivileges`, `AmbientCapabilities` in place of root), a Dockerfile, and an OpenWrt
procd init script. The exporter mode serves Prometheus metrics on `:9862` and natively
supports TLS termination via `-metrics-tls-cert` / `-metrics-tls-key` — no reverse proxy
required for that leg.

We need a turnkey, repeatable way to roll piccolo-perf out across a fleet of Unix/Linux
hosts and to bring Prometheus scraping up to a consistent security baseline everywhere,
given:

- The fleet is managed manually today, except for one existing Ansible instance that
  bootstraps each host on install. That bootstrap is the natural extension point.
- A few hosts already run Prometheus; those installs are currently exposed (bound to all
  interfaces) and unencrypted. Most hosts have no Prometheus at all.
- Every host has a mesh VPN with a consistent interface name across the fleet, and all
  address binding should assume IPv6.
- Public ACME (Let's Encrypt) certificates are acceptable for TLS.

## Goals

- Idempotently install/upgrade piccolo-perf (agent or exporter mode) fleet-wide via the
  existing Ansible bootstrap.
- Idempotently bring every host's Prometheus to a hardened baseline, whether it's a fresh
  install or an existing exposed one: bound to the mesh interface only, TLS-terminated,
  basic-auth protected.
- Isolate both Prometheus's web UI/API and the piccolo-perf `/metrics` endpoint to the
  mesh VPN interface, with firewall rules as defense-in-depth against bind-address
  misconfiguration.
- Issue and distribute TLS certificates via one central DNS-01 ACME client, not per-host
  credentials.
- Preserve a manual fallback (`install.sh`) for any device outside Ansible's reach.

## Non-goals

- Building a new Prometheus-alternative dashboarding/alerting stack (Grafana dashboards
  already ship in `deploy/grafana/`; out of scope here).
- Mutual TLS / client-certificate auth between mesh peers and Prometheus. Native
  Prometheus TLS + basic auth is judged sufficient given the mesh interface is already a
  trusted-peer network; mTLS can be layered on later without changing this design's
  structure.
- Federated/multi-region Prometheus topology. Each host that already runs or should run
  Prometheus is hardened independently; cross-Prometheus federation is a separate effort.
- Migrating the mesh VPN itself, or managing its rollout — it's assumed already present
  and stable on every host, with a consistent interface name available as an Ansible fact.

## Architecture overview

```
┌─────────────────────────────────────────────────────────────────┐
│ Ansible control node                                             │
│                                                                   │
│  site.yml (existing bootstrap play)                              │
│    ├── role: piccolo_perf         (every fleet host)             │
│    ├── role: prometheus_hardened  (hosts in group prometheus_hosts)│
│    └── play: cert-distribute.yml  (triggered by cert renewal)    │
│                                                                   │
│  cert-issuer (runs on control node or a designated host)         │
│    acme.sh --issue --dns <provider> -d '*.mesh.example.net'      │
│    → renews at day 60/90, writes fullchain+key to a vault-        │
│      readable path, triggers cert-distribute.yml                 │
└─────────────────────┬─────────────────────────────────────────────┘
                       │ push (ansible-playbook over SSH)
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│ Each fleet host                                                  │
│                                                                   │
│  mesh VPN interface (consistent name, IPv6)                      │
│    ├── piccolo-perf exporter/agent  -metrics-addr [meshv6]:9862  │
│    │     TLS via -metrics-tls-cert/-key (wildcard cert)          │
│    └── prometheus (if in prometheus_hosts group)                 │
│          --web.listen-address=[meshv6]:9090                      │
│          web.config.yml: tls_server_config + basic_auth_users    │
│                                                                   │
│  nftables: meshv6:9862 and meshv6:9090 restricted to              │
│            iifname "<mesh_iface>" (defense-in-depth)             │
└─────────────────────────────────────────────────────────────────┘
```

## Components

### 1. `piccolo_perf` Ansible role

Applied to every host in the fleet inventory.

- **Binary install**: `ansible.builtin.get_url` against the pinned GitHub release asset
  matching `{{ ansible_system | lower }}_{{ arch_map[ansible_architecture] }}`, followed
  by checksum verification against the release's `checksums.txt` (mirrors `install.sh`'s
  logic without shelling out to a downloaded script). Idempotent: skipped if the
  installed binary's `-version` output already matches the pinned version.
- **Config**: the topology JSON (currently pulled live over HTTP from a config-server) is
  instead templated locally by Ansible from inventory/group_vars and redeployed on
  change. This removes an unauthenticated HTTP service from the fleet's attack surface;
  Ansible already reaches every host, so there's no live-reload need it was serving.
- **Units**: template `deploy/piccolo-perf-agent.service` / `-exporter.service`
  (unmodified hardening: `DynamicUser`, `NoNewPrivileges`, `AmbientCapabilities`) with
  two templated values: `-metrics-addr "[{{ mesh_ipv6 }}]:9862"` and
  `-metrics-tls-cert` / `-metrics-tls-key` pointing at the distributed wildcard cert.
- **Interface binding**: `mesh_ipv6` is resolved from
  `ansible_facts[mesh_iface]['ipv6']` (first non-link-local address), where
  `mesh_iface` is a group_var set once (consistent name across the fleet).
- **Firewall**: nftables task restricting inbound `:9862` to `iifname mesh_iface`.

### 2. `prometheus_hardened` Ansible role

Applied to hosts in a `prometheus_hosts` inventory group (today: the few hosts that
already run it; expand the group over time as needed — the role doesn't require
Prometheus to be new).

- **Detect-or-install**: check for an existing `prometheus.service` unit / binary.
  - If present → "adopt & remediate" path: rewrite the unit's flags in place, don't
    reinstall the binary unless the pinned version differs.
  - If absent → install from the pinned prometheus.io release tarball.
  - Both paths converge on the same handler-driven restart once config changes.
- **Interface binding**: `--web.listen-address="[{{ mesh_ipv6 }}]:9090"`. Unit gets
  `After=` and `BindsTo=` on the mesh VPN's systemd unit name (group_var), so Prometheus
  restarts and rebinds if the tunnel restarts or comes up late at boot — binding a
  literal address at service-start time is fragile if the interface isn't up yet.
- **TLS + basic auth**: template `web.config.yml` with `tls_server_config` (cert/key
  paths matching the distributed wildcard cert) and `basic_auth_users` (bcrypt hash
  generated once and stored in Ansible Vault, same for the whole fleet or per-host —
  per-host preferred to limit blast radius of a leaked hash).
- **Scrape config**: `prometheus.yml` scrape job targets piccolo-perf hosts by mesh-VPN
  hostname (mesh DNS), `scheme: https`. No custom `ca_file` needed — Let's Encrypt is in
  Prometheus's default trust store.
- **Firewall**: same nftables pattern as above, restricting `:9090` to `iifname
  mesh_iface`.

### 3. Central cert issuance (DNS-01) + distribution

- One host (the Ansible control node, or a small dedicated "cert-issuer" host) runs
  `acme.sh` (or `certbot` with a DNS plugin) against the mesh domain's authoritative DNS
  provider, issuing a single wildcard cert (`*.mesh.example.net`). The DNS provider API
  token lives in Ansible Vault on that one host only — this is the reason for
  centralizing issuance instead of per-host ACME clients: one credential, one blast
  radius, instead of one per fleet host.
- Renewal runs on a timer (systemd timer or cron) at day 60 of the 90-day lifetime,
  invoking a wrapper script that, on successful renewal, triggers
  `ansible-playbook cert-distribute.yml`.
- `cert-distribute.yml` copies `fullchain.pem` / `privkey.pem` to every host that needs
  them (`piccolo_perf` group ∪ `prometheus_hosts` group) into a fixed path (e.g.
  `/etc/piccolo-perf/tls/` and `/etc/prometheus/tls/`), with `notify` handlers that
  restart the relevant service only on actual content change (idempotent — no
  restart storms on a no-op run).

### 4. Manual/embedded fallback

The existing `install.sh` one-liner remains documented as the fallback for any device
outside Ansible's reach (e.g., embedded devices without Python). It installs the binary
only — TLS/interface-binding hardening for such a device is a manual follow-up, out of
scope for automated coverage in this design.

## Data flow / sequencing

1. Cert issuance (independent timer) → wildcard cert refreshed centrally →
   `cert-distribute.yml` pushes to fleet → services reload on change.
2. `site.yml` bootstrap (existing) gains two new role invocations:
   `piccolo_perf` (all hosts), `prometheus_hardened` (hosts in `prometheus_hosts`).
3. Both roles are safe to re-run at any time (full idempotency) — a normal
   `ansible-playbook site.yml` run against the whole fleet converges any drifted host
   back to the hardened baseline, including the currently-exposed Prometheus instances.

## Rollout plan

Canary both new roles against a small inventory group first — especially the "adopt &
remediate an already-exposed Prometheus" path, since it touches live infrastructure.
Verify: scrape success (`up{job="piccolo_perf"} == 1`), cert validity/expiry, and that
the Prometheus web UI is unreachable from outside the mesh interface. Then expand
`prometheus_hosts` and the general fleet group incrementally.

## Testing

- Molecule (or equivalent Ansible role testing) for `piccolo_perf` and
  `prometheus_hardened`, run against a container/VM target with a fake mesh interface,
  asserting: service binds only to the expected address, `web.config.yml` enables TLS,
  firewall rule exists and matches the interface name, re-running the role twice
  produces no changes on the second run (idempotency check).
- A smoke-test play (`verify-fleet.yml`) usable ad hoc or in CI: for every host in
  `prometheus_hosts`, curl `/metrics` and `/-/healthy` over HTTPS through the mesh
  interface and confirm a 200 with valid cert chain; confirm the same endpoints are
  unreachable when attempted from outside the mesh (run from a host not on the mesh, or
  assert firewall rule state directly).
- Cert distribution play tested for idempotency: running it twice with no underlying
  cert change should trigger zero service restarts.

## Open questions / follow-ups (not blocking this design)

- Whether `basic_auth_users` hashes should be per-host or fleet-wide — leaning per-host,
  final call deferred to implementation.
- Whether to expand `prometheus_hosts` to cover the whole fleet over time, or keep a
  smaller set of dedicated Prometheus servers scraping many piccolo-perf exporters — this
  design supports either; it's an inventory-grouping decision, not a structural one.
