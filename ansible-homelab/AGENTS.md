# AGENTS.md — Ansible Homelab

Agent-oriented reference for working in this repository.

## Running commands

All Ansible tooling is managed by **uv**. Prefix every `ansible` / `ansible-playbook` / `ansible-galaxy` / `ansible-vault` / `ansible-lint` invocation with `uv run`:

```bash
uv run ansible-playbook -i inventory/hosts.yml site.yml \
  --vault-password-file ~/.vault_pass
```

The Makefile wraps the common patterns — prefer it for routine operations:

```bash
make dry-run                        # --check --diff across all hosts
make dry-run LIMIT=mini-server      # dry run on one host
make deploy LIMIT=mini-server       # real run on one host
make ping                           # connectivity check
make install                        # install Galaxy collections
make lint                           # ansible-lint
```

### Common one-liners (uv-prefixed)

```bash
# Ping all hosts
uv run ansible all -i inventory/hosts.yml -m ping \
  --vault-password-file ~/.vault_pass

# Full site run
uv run ansible-playbook -i inventory/hosts.yml site.yml \
  --vault-password-file ~/.vault_pass

# Single host
uv run ansible-playbook -i inventory/hosts.yml site.yml \
  --vault-password-file ~/.vault_pass --limit mini-server

# Single role via tag
uv run ansible-playbook -i inventory/hosts.yml site.yml \
  --vault-password-file ~/.vault_pass --tags caddy

# Dry run + diff
uv run ansible-playbook -i inventory/hosts.yml site.yml \
  --vault-password-file ~/.vault_pass --check --diff

# Edit a vault file
uv run ansible-vault edit group_vars/all/vault.yml \
  --vault-password-file ~/.vault_pass
```

---

## Repository layout

```
ansible-homelab/
├── site.yml                        # Master playbook — runs all roles in order
├── inventory/hosts.yml             # All managed hosts + group assignments
├── ansible.cfg                     # Inventory path, SSH tuning, callback config
├── Makefile                        # uv-wrapped shortcuts
├── pyproject.toml                  # Python deps (ansible-core, ansible-lint) via uv
├── requirements.yml                # Ansible Galaxy collections
│
├── group_vars/all/
│   ├── vars.yml                    # Shared non-secret variables
│   ├── vault.yml                   # Encrypted secrets (ansible-vault)
│   └── vault_caddy.yml             # Encrypted AWS credentials for Caddy/Route53
│
├── host_vars/
│   ├── mini-server.yml             # mini-server-specific vars
│   └── desktop-tower.yml           # desktop-tower-specific vars
│
└── roles/
    ├── docker/                     # Installs Docker Engine + Compose plugin
    ├── monitoring-agents/          # node_exporter + cAdvisor (all hosts)
    ├── monitoring-server/          # Prometheus (mini-server)
    ├── portainer/                  # Portainer CE (mini-server)
    ├── portainer-agent/            # Portainer Agent (desktop-tower)
    ├── frigate-influx/             # Frigate NVR + InfluxDB (desktop-tower)
    ├── immich/                     # Immich photo management (mini-server)
    ├── homepage/                   # Homepage dashboard (mini-server)
    └── caddy/                      # Caddy reverse proxy (mini-server)
```

---

## Hosts

| Host | IP | Groups |
|---|---|---|
| `mini-server` | `192.168.4.239` | `monitored`, `monitoring_servers`, `portainer_hosts`, `photo_management`, `reverse_proxy`, `homepage_hosts` |
| `desktop-tower` | `192.168.4.10` | `monitored`, `portainer_agents`, `nvr` |

---

## Roles reference

### `docker`
Installs Docker Engine + Compose plugin from the official Docker apt repo. Adds the Ansible user to the `docker` group. Safe to re-run.

### `monitoring-agents`
Deploys node_exporter (systemd binary) and cAdvisor (Docker container) on every host in `monitored`. Versions pinned in `group_vars/all/vars.yml`.

### `monitoring-server`
Prometheus on `mini-server`. Scrape targets are auto-generated from `host_ip` + port variables in `host_vars/`.

### `portainer` / `portainer-agent`
Portainer CE on `mini-server` (ports 9000/9443). Portainer Agent on `desktop-tower`.

### `frigate-influx`
Frigate NVR + InfluxDB 1.x on `desktop-tower`. Device passthrough and paths configured in `host_vars/desktop-tower.yml`.

### `immich`
Immich on `mini-server`. Downloads the official `docker-compose.yml` from GitHub releases; templates a `.env` with vault secrets. NFS mount for photo storage.

### `homepage`
Homepage application dashboard (`gethomepage/homepage`) on `mini-server`. Preconfigured with dashboard links and live widgets for Frigate NVR (on `desktop-tower`), Immich, Portainer, and Prometheus.

### `caddy`
Custom Caddy build (with `caddy-dns/route53`) deployed via Docker Compose on `mini-server`. Issues a wildcard TLS cert for `*.internal.lrlilford.com` via Let's Encrypt DNS-01 challenge against Route53.

**Adding a new proxied service:** append an entry to `caddy_services` in `roles/caddy/defaults/main.yml` (or override in `host_vars/mini-server.yml`), then re-run the playbook. Only the Caddyfile changes, so Caddy hot-reloads with no downtime.

```yaml
caddy_services:
  - hostname: cameras.internal.lrlilford.com
    target_ip: 192.168.4.10
    target_port: 5000
  - hostname: homepage.internal.lrlilford.com
    target_ip: 192.168.4.239
    target_port: 3000
  - hostname: grafana.internal.lrlilford.com   # example new entry
    target_ip: 127.0.0.1
    target_port: 3000
```

---

## Secrets

All secrets live in encrypted vault files. **Never hardcode secret values in plain vars files.**

| Vault variable | Used by | Vault file |
|---|---|---|
| `vault_ansible_user` | SSH connection | `vault.yml` |
| `vault_tower_password` | SSH to desktop-tower | `vault.yml` |
| `vault_grafana_admin_password` | `monitoring-server` | `vault.yml` |
| `vault_influxdb_admin_password` | `frigate-influx` | `vault.yml` |
| `vault_influxdb_admin_token` | `frigate-influx` | `vault.yml` |
| `vault_frigate_camera_password` | `frigate-influx` | `vault.yml` |
| `vault_immich_db_password` | `immich` | `vault.yml` |
| `vault_caddy_aws_access_key_id` | `caddy` | `vault_caddy.yml` |
| `vault_caddy_aws_secret_access_key` | `caddy` | `vault_caddy.yml` |

The vault password file lives at `~/.vault_pass` (never committed). To bootstrap it:

```bash
echo "your-vault-password" > ~/.vault_pass
chmod 600 ~/.vault_pass
```

---

## Port reference

| Service | Host | Port |
|---|---|---|
| node_exporter | both | 9100 |
| cAdvisor | both | 8080 |
| Prometheus | mini-server | 9090 |
| Portainer HTTP | mini-server | 9000 |
| Portainer HTTPS | mini-server | 9443 |
| Immich | mini-server | 2283 |
| Homepage | mini-server | 3000 |
| Caddy HTTP | mini-server | 80 |
| Caddy HTTPS | mini-server | 443 |
| Frigate Web UI | desktop-tower | 5000 |
| Frigate RTSP | desktop-tower | 8554 |
| InfluxDB | desktop-tower | 8086 |
