# piccolo-perf fleet install

Ansible project to roll out [piccolo-perf](https://github.com/retecolo/piccolo-perf)
exporters fleet-wide and bring every host's Prometheus to a hardened baseline:
mesh-interface-only binding, native TLS, and a centrally-issued Let's Encrypt
wildcard certificate.

Full design: `docs/superpowers/specs/2026-07-16-piccolo-perf-fleet-install-design.md`

## Prerequisites

- A mesh VPN with a **consistent interface name** across the fleet, providing
  IPv6 addressing and (ideally) internal DNS resolving `<hostname>.<mesh_domain>`.
- Inventory hostnames matching the mesh VPN's hostnames (see `inventory/hosts.example.ini`).
- A DNS provider supported by a certbot DNS-01 plugin, for the wildcard cert.
- Ansible Vault secrets (create before first run):
  - `cert_issuer_dns_credentials_content` — DNS provider API credentials file contents
  - `prometheus_basic_auth_password_hash` — bcrypt hash (`htpasswd -nbBC 10 admin '<pw>'`)

## Bootstrap order

1. Copy `inventory/hosts.example.ini` to `inventory/hosts.ini` and edit it, and
   `inventory/group_vars/all.yml` to match your fleet's topology.
2. Set the two Vault secrets above.
3. `ansible-playbook -i inventory/hosts.ini playbooks/cert-issue.yml` — issues the
   wildcard cert and distributes it to every host currently in inventory (the
   distribution is triggered automatically by certbot's `--deploy-hook`).
4. `ansible-playbook -i inventory/hosts.ini playbooks/site.yml` — deploys
   piccolo-perf, the config server, and hardened Prometheus in one run.
5. `ansible-playbook -i inventory/hosts.ini playbooks/verify-fleet.yml` — smoke-tests
   every deployed service over HTTPS via the mesh interface.

`playbooks/verify-fleet.yml` needs the **plaintext** Prometheus basic-auth password
(to make an authenticated HTTPS request), separate from the bcrypt hash consumed by
`web.config.yml`. Set `prometheus_basic_auth_password` via Vault alongside
`prometheus_basic_auth_password_hash` if you intend to run this playbook.

## Role testing

Each role has a Molecule scenario:

```sh
cd roles/piccolo_perf && molecule test
cd roles/config_server && molecule test
cd roles/prometheus_hardened && molecule test
```

`cert_issuer` has no Molecule scenario — DNS-01 issuance can't run in CI. It's
covered by `ansible-lint` and `ansible-playbook --syntax-check` only; test it
manually against a real DNS zone before relying on it.

**`cert_issuer` requires manual verification.** Before relying on it in production,
run `playbooks/cert-issue.yml` once against a real DNS zone with a real (even if
short-lived, e.g. via Let's Encrypt's staging environment) certbot DNS plugin
credential, and confirm `/etc/letsencrypt/live/<mesh_domain>/fullchain.pem` is
issued and `playbooks/cert-distribute.yml` fires via the deploy hook.
