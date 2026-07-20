# piccolo-perf fleet install

Ansible project that idempotently rolls [piccolo-perf](https://github.com/retecolo/piccolo-perf)
out across a fleet of Unix/Linux hosts and brings every host's Prometheus to a hardened
baseline: bound to the mesh VPN interface only, TLS-terminated with a centrally-issued
Let's Encrypt wildcard certificate, and firewalled as defense-in-depth.

## Architecture

```
Control node (cert_issuer_hosts)
  ├── certbot DNS-01  →  *.mesh.example.net wildcard cert
  └── cert-distribute.yml  →  pushes cert to fleet on renewal

Fleet hosts
  ├── piccolo-perf-exporter   [mesh_ipv6]:9862   HTTPS/TLS
  ├── nginx config server     [mesh_ipv6]:8443   HTTPS/TLS
  └── prometheus (selected)   [mesh_ipv6]:9090   HTTPS/TLS + basic auth

nftables on every host: 9862, 8443, 9090 blocked on all interfaces except the mesh VPN
```

All network services bind exclusively to each host's mesh-VPN IPv6 address, never to
`0.0.0.0` or `::`. The wildcard cert is issued once centrally and distributed to every
host that needs it, so no host carries DNS credentials.

## Repository layout

```
piccolo-perf-install/
├── ansible.cfg
├── inventory/
│   ├── hosts.example.ini          # copy to hosts.ini and edit
│   └── group_vars/
│       ├── all.yml                # mesh topology & fleet config
│       ├── prometheus_hosts.yml   # prometheus version & auth vars
│       └── config_server_hosts.yml
├── roles/
│   ├── piccolo_perf/              # install + harden piccolo-perf exporter
│   ├── config_server/             # nginx serving topology JSON over HTTPS
│   ├── prometheus_hardened/       # install or adopt-and-harden Prometheus
│   └── cert_issuer/               # DNS-01 ACME issuance + renewal timer
└── playbooks/
    ├── site.yml                   # deploys everything in one run
    ├── piccolo-perf.yml
    ├── config-server.yml
    ├── prometheus.yml
    ├── cert-issue.yml             # run once to issue the wildcard cert
    ├── cert-distribute.yml        # push renewed cert to fleet
    └── verify-fleet.yml           # HTTPS smoke-test across all services
```

## Requirements

**Control node**

- Ansible ≥ 2.15
- Python ≥ 3.9
- A DNS provider supported by a certbot DNS-01 plugin (Cloudflare, Route53, etc.)
- The project checked out somewhere the certbot deploy hook can reach:
  `cert_issuer_playbook_dir` (default `/opt/piccolo-fleet`)

**Fleet hosts**

- Debian/Ubuntu family (uses `apt`). Other families are a documented follow-up.
- A mesh VPN (WireGuard or similar) with a **consistent interface name** across the
  entire fleet, providing IPv6 ULA addressing.
- SSH access from the control node via the mesh VPN.
- Python 3 (Ansible dependency).

**Ports used**

| Service | Port | Protocol | Bound to |
|---|---|---|---|
| piccolo-perf exporter | 9862 | TCP | mesh IPv6 only |
| config server (nginx) | 8443 | TCP | mesh IPv6 only |
| Prometheus | 9090 | TCP | mesh IPv6 only |

---

## Initial setup

### 1. Clone and configure

```bash
git clone <this-repo> /opt/piccolo-fleet
cd /opt/piccolo-fleet
```

Copy the example inventory and edit it to match your fleet:

```bash
cp inventory/hosts.example.ini inventory/hosts.ini
$EDITOR inventory/hosts.ini
```

The inventory uses short hostnames as Ansible host names; `ansible_host` carries the
mesh-VPN FQDN used for SSH:

```ini
[piccolo_perf]
probe-a ansible_host=probe-a.mesh.example.net
probe-b ansible_host=probe-b.mesh.example.net

[prometheus_hosts]
probe-a          # hosts that should run (or already run) Prometheus

[config_server_hosts]
probe-a          # exactly one host serves the topology JSON

[cert_issuer_hosts]
localhost ansible_connection=local   # the control node issues certs
```

### 2. Edit fleet topology

Open `inventory/group_vars/all.yml` and fill in your actual values:

```yaml
mesh_iface: wg0                    # WireGuard interface name, same on every host
mesh_domain: mesh.example.net      # internal DNS domain for the fleet
mesh_vpn_service: "wg-quick@wg0.service"  # systemd unit for the mesh VPN

piccolo_perf_hosts:
  - { name: probe-a, address: "fd00:dead:beef::1", site: site-a }
  - { name: probe-b, address: "fd00:dead:beef::2", site: site-b }

piccolo_perf_measurements:
  - { type: twamp, interval: 60s, targets: all, burst_size: 5,
      burst_interval: 200ms, packet_timeout: 5s }
  - { type: dns, interval: 120s,
      resolvers: ["2620:fe::fe", "9.9.9.9"],
      names: ["example.com"] }
```

The `piccolo_perf_hosts` list is the single source of truth for fleet topology. The
`config_server` role serves it as JSON over HTTPS; the `prometheus_hardened` role uses it
to build the Prometheus scrape config. Add or remove hosts here and re-run `site.yml`
to propagate changes everywhere.

### 3. Configure secrets with Ansible Vault

Two group_vars files hold secrets that must be vault-encrypted before committing.

#### Cert issuer credentials

`inventory/group_vars/cert_issuer_hosts.yml` is created for you with a placeholder.
Edit it to add your real DNS API token, then encrypt:

```bash
$EDITOR inventory/group_vars/cert_issuer_hosts.yml
ansible-vault encrypt inventory/group_vars/cert_issuer_hosts.yml
```

The file should contain:

```yaml
cert_issuer_acme_email: "admin@mesh.example.net"

# For Cloudflare (python3-certbot-dns-cloudflare):
cert_issuer_dns_credentials_content: |
  dns_cloudflare_api_token = <your-token-here>
```

The Cloudflare token needs **Zone:Zone:Read** and **Zone:DNS:Edit** permissions
for the zone that contains your `mesh_domain`.

> **DNS plugin selection:** The default plugin is `cloudflare`. To use a different
> provider, override `cert_issuer_dns_plugin_name` in `cert_issuer_hosts.yml`:
>
> ```yaml
> cert_issuer_dns_plugin_name: route53
> # cert_issuer_certbot_dns_package and cert_issuer_certbot_authenticator
> # are derived automatically from cert_issuer_dns_plugin_name.
> ```

#### Prometheus credentials

Edit `inventory/group_vars/prometheus_hosts.yml` to add the password variables,
then encrypt:

```bash
# Generate a bcrypt hash
htpasswd -nbBC 10 admin '<password>'

$EDITOR inventory/group_vars/prometheus_hosts.yml
# Add:
#   prometheus_basic_auth_password_hash: "$2y$10$..."
#   prometheus_basic_auth_password: "<your-password-here>"

ansible-vault encrypt inventory/group_vars/prometheus_hosts.yml
```

Pass `--ask-vault-pass` or `--vault-password-file ~/.vault_pass` on every
`ansible-playbook` invocation that touches encrypted files.

---

## First deployment

Run these four commands in order. Each is idempotent; re-running is always safe.

### Step 1: Issue the wildcard certificate

This runs on the control node (`cert_issuer_hosts`) and issues
`*.mesh.example.net` via DNS-01 ACME. On success, certbot's deploy hook
automatically calls `cert-distribute.yml` to push the cert to every host in
inventory.

```bash
ansible-playbook -i inventory/hosts.ini playbooks/cert-issue.yml \
  --ask-vault-pass
```

Expected outcome: `/etc/letsencrypt/live/mesh.example.net/fullchain.pem` and
`privkey.pem` exist on the control node, and the cert files appear at
`/etc/piccolo-perf/tls/` and `/etc/prometheus/tls/` on every fleet host.

### Step 2: Deploy the full stack

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --ask-vault-pass
```

This is equivalent to running `piccolo-perf.yml`, `config-server.yml`, and
`prometheus.yml` in sequence. On first run it will:

- Download and install piccolo-perf v1.0.7 on every `piccolo_perf` host.
- Deploy the `piccolo-perf-exporter` systemd unit bound to the mesh IPv6 address.
- Install nginx on `config_server_hosts` and serve the topology JSON over HTTPS on port 8443.
- Install or upgrade Prometheus to v3.13.1 on every `prometheus_hosts` host, applying
  TLS + basic auth and rebinding from any existing `0.0.0.0:9090` listener to the mesh
  interface only.
- Apply nftables rules on every host restricting the relevant ports to the mesh interface.

### Step 3: Verify

```bash
ansible-playbook -i inventory/hosts.ini playbooks/verify-fleet.yml \
  --ask-vault-pass
```

This hits every deployed service over HTTPS through the mesh interface:

- `piccolo-perf` `/metrics` on each `piccolo_perf` host — expects HTTP 200.
- Config server `/piccolo-config.json` — expects HTTP 200.
- Prometheus `/-/healthy` with basic auth — expects HTTP 200.
- Prometheus API query `up{job="piccolo_perf"}` — asserts every target reports `1`.

---

## Day-2 operations

### Re-run after inventory changes

Adding a host to `[piccolo_perf]` and re-running `site.yml` is sufficient. The roles
are fully idempotent — hosts already at the correct state see zero changes.

### Certificate renewal

Certbot's renewal timer (`piccolo-cert-renew.timer`) fires daily on the control node.
When a cert is actually renewed (within 30 days of expiry), the deploy hook
automatically runs `cert-distribute.yml`, which copies the new cert to every host and
restarts the affected services only if the cert content changed.

To force an immediate renewal check:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/cert-issue.yml \
  --ask-vault-pass
```

To distribute a cert that was renewed outside Ansible (e.g., manually via certbot):

```bash
ansible-playbook -i inventory/hosts.ini playbooks/cert-distribute.yml \
  --ask-vault-pass
```

### Expanding the fleet

1. Add the new host to the appropriate groups in `inventory/hosts.ini`.
2. Add it to `piccolo_perf_hosts` in `inventory/group_vars/all.yml`.
3. Run `cert-distribute.yml` to push the existing cert to the new host.
4. Run `site.yml` to deploy the stack.

### Updating piccolo-perf or Prometheus

Change `piccolo_perf_version` in `roles/piccolo_perf/defaults/main.yml` or
`prometheus_version` in `roles/prometheus_hardened/defaults/main.yml` and re-run
`site.yml`. The install tasks check the currently installed version and only download
and replace the binary if the version doesn't match.

### Rotating the Prometheus password

1. Generate a new bcrypt hash: `htpasswd -nbBC 10 admin '<new-password>'`
2. Update `prometheus_basic_auth_password_hash` and `prometheus_basic_auth_password`
   in the vault.
3. Run `ansible-playbook -i inventory/hosts.ini playbooks/prometheus.yml --ask-vault-pass`.
   The `web.config.yml` template will be redeployed and Prometheus restarted.

---

## Configuration reference

### `inventory/group_vars/all.yml` — fleet-wide

| Variable | Default | Description |
|---|---|---|
| `mesh_iface` | `wg0` | Mesh VPN interface name (must be consistent across fleet) |
| `mesh_domain` | `mesh.example.net` | Internal DNS domain |
| `mesh_vpn_service` | `wg-quick@wg0.service` | systemd unit for the mesh VPN |
| `piccolo_perf_hosts` | see file | List of `{name, address, site}` dicts describing the fleet |
| `piccolo_perf_measurements` | see file | List of measurement configs served to piccolo-perf |

### `roles/piccolo_perf/defaults/main.yml`

| Variable | Default | Description |
|---|---|---|
| `piccolo_perf_version` | `1.0.7` | Release tag to install |
| `piccolo_perf_binary_path` | `/usr/local/bin/piccolo-perf` | Install path |
| `piccolo_perf_metrics_port` | `9862` | Port the exporter listens on |
| `piccolo_perf_twamp_port` | `862` | Port for TWAMP measurements |
| `piccolo_perf_probe_mode` | `background` | piccolo-perf probe mode |
| `piccolo_perf_tls_dir` | `/etc/piccolo-perf/tls` | Where the cert and key live |
| `piccolo_perf_cert_group` | `piccolo-perf-cert` | Group with read access to TLS files |
| `piccolo_perf_service_name` | `piccolo-perf-exporter` | systemd unit name |
| `piccolo_perf_config_url` | `https://config.<mesh_domain>:8443/piccolo-config.json` | URL piccolo-perf fetches its config from |

### `roles/prometheus_hardened/defaults/main.yml`

| Variable | Default | Description |
|---|---|---|
| `prometheus_version` | `3.13.1` | Release to install |
| `prometheus_binary_path` | `/usr/local/bin/prometheus` | Install path |
| `prometheus_config_dir` | `/etc/prometheus` | Config directory |
| `prometheus_data_dir` | `/var/lib/prometheus` | TSDB storage directory |
| `tls_cert_dir_prometheus` | `/etc/prometheus/tls` | Where the cert and key live |
| `prometheus_web_port` | `9090` | Port Prometheus listens on |

Set in `inventory/group_vars/prometheus_hosts.yml`:

| Variable | Description |
|---|---|
| `prometheus_basic_auth_username` | Username for basic auth (default `admin`) |
| `prometheus_basic_auth_password_hash` | bcrypt hash — **must be in Vault** |

### `roles/config_server/defaults/main.yml`

| Variable | Default | Description |
|---|---|---|
| `config_server_port` | `8443` | HTTPS port nginx listens on |
| `config_server_www_dir` | `/etc/piccolo-fleet/www` | Document root |
| `config_server_tls_dir` | `/etc/piccolo-fleet/tls` | TLS cert location |
| `config_server_remove_default_site` | `false` | Remove `/etc/nginx/sites-enabled/default`; set `true` only on hosts where no other service (e.g. Grafana) uses that vhost |

**Co-locating with Grafana or other nginx vhosts:** the role deploys a single named vhost (`piccolo-fleet-config-server.conf`) on port 8443 bound exclusively to the mesh IPv6 address. It does not touch other vhosts. Set `config_server_remove_default_site: false` (the default) to leave any existing Grafana or default vhost in place. The systemd drop-in uses `Wants=` rather than `BindsTo=` for the mesh VPN service so that nginx stays up (and Grafana keeps serving) even if the mesh interface goes down.

**First-run ordering:** nginx is only started by the role when the TLS cert is already present at `config_server_tls_dir`. On a fresh host, run `cert-issue.yml` first; the deploy hook calls `cert-distribute.yml`, which copies the cert and reloads nginx automatically. If you run `site.yml` before issuing a cert, the role enables nginx but does not start it — re-run after cert distribution to bring it up.

### `roles/cert_issuer/defaults/main.yml`

| Variable | Default | Description |
|---|---|---|
| `cert_issuer_acme_email` | `admin@<mesh_domain>` | Email for Let's Encrypt registration |
| `cert_issuer_dns_plugin_name` | `cloudflare` | certbot DNS plugin name |
| `cert_issuer_dns_credentials_path` | `/etc/letsencrypt/dns-credentials.ini` | Where credentials file is written |
| `cert_issuer_dns_credentials_content` | `""` | **Must be in Vault** — credentials file contents |
| `cert_issuer_dns_propagation_seconds` | `60` | Seconds to wait for DNS TXT record propagation before ACME validation |
| `cert_issuer_deploy_hook_path` | `/usr/local/sbin/piccolo-cert-deploy-hook.sh` | Script certbot calls on renewal |
| `cert_issuer_playbook_dir` | `/opt/piccolo-fleet` | Project root on the control node |
| `cert_issuer_inventory_path` | `inventory/hosts.ini` | Inventory path used by the deploy hook |

---

## Security design

**Bind address as the primary control.** Every service binds to a specific IPv6 address
(`mesh_ipv6`, resolved from `ansible_facts[mesh_iface]`), not to `0.0.0.0` or `::`.
If the mesh interface goes down, services fail to bind rather than silently falling back
to a public address.

**nftables as defense-in-depth.** Each role deploys a named nftables table that drops
inbound traffic to its port(s) from any interface other than `mesh_iface`. This protects
against a bind-address misconfiguration surviving a restart and against other processes
opening the same ports on unintended interfaces. Loopback (`lo`) is always allowed for
local health checks.

**Single-credential ACME issuance.** The DNS provider API token lives on the control
node only, in Ansible Vault. Fleet hosts never hold credentials capable of issuing
certificates. The cert is pushed to hosts after issuance; `cert-distribute.yml` restarts
services only when the cert content actually changes (idempotent).

**No root for piccolo-perf.** The exporter unit uses `DynamicUser=yes` and
`NoNewPrivileges=true`, acquiring only the capabilities it needs
(`CAP_NET_BIND_SERVICE`, `CAP_NET_RAW`) via `AmbientCapabilities`.

**Prometheus locked down on adopt.** The `prometheus_hardened` role handles the
"adopt an existing exposed install" case: if Prometheus is already running bound to
`0.0.0.0:9090` with no TLS, the role replaces its systemd unit and restarts it. The
Molecule scenario exercises exactly this path — `prepare.yml` intentionally installs a
stale, insecure Prometheus, and the verify assertions confirm it is remediated.

---

## Role testing

Each role ships a Molecule scenario using the Docker driver with a systemd-capable
Debian 12 image. Tests include prepare → converge → idempotency check → verify.

```bash
# Run a single role's full suite
cd roles/piccolo_perf && molecule test
cd roles/config_server && molecule test
cd roles/prometheus_hardened && molecule test

# Or run all three in sequence from the project root
for r in piccolo_perf config_server prometheus_hardened; do
  (cd roles/$r && molecule test) || exit 1
done
```

The `cert_issuer` role has no Molecule scenario — DNS-01 issuance requires live DNS
infrastructure and cannot run in CI containers. It is covered by `ansible-lint` and
`--syntax-check` only. Test it manually against a real DNS zone (Let's Encrypt staging
is a good choice) before relying on it in production.

### Local environment notes

If Docker is managed by [Colima](https://github.com/abiosoft/colima) on macOS, set
`DOCKER_HOST` before running Molecule:

```bash
export DOCKER_HOST="unix://${HOME}/.colima/default/docker.sock"
molecule test
```

---

## Linting

```bash
ansible-lint roles playbooks
```

Zero failures are expected. Warnings on handler naming and conditional tasks are
acknowledged and suppressed in `.ansible-lint`.

---

## Manual fallback

The upstream `install.sh` one-liner remains the fallback for any device outside
Ansible's reach (OpenWrt routers, embedded hosts without Python):

```bash
curl -sSL https://raw.githubusercontent.com/retecolo/piccolo-perf/main/install.sh | sh
```

This installs the binary only. TLS, interface binding, and nftables hardening for
such devices require manual follow-up and are outside the scope of this project.

---

## Full design document

`docs/superpowers/specs/2026-07-16-piccolo-perf-fleet-install-design.md`
