# ansible-homelab

Ansible project to provision monitoring and container infrastructure across two Debian/Ubuntu machines.

> **Out of scope**: The Raspberry Pi running Home Assistant OS is intentionally **not** in this inventory and will never be touched.

---

## Machines

| Host | IP | Services |
|---|---|---|
| `desktop-tower` | `192.168.4.10` | Frigate NVR, InfluxDB, node_exporter, cAdvisor |
| `mini-server` | `192.168.4.239` *(update in host_vars)* | Immich, Portainer, Prometheus, node_exporter, cAdvisor |

---

## Directory Structure

```
ansible-homelab/
├── ansible.cfg
├── requirements.yml
├── site.yml                        # Master playbook
├── inventory/
│   └── hosts.yml                   # All hosts and groups
├── group_vars/
│   └── all/
│       ├── vars.yml                # Plain vars referencing vault vars
│       └── vault.yml               # 🔒 Encrypted secrets (ansible-vault)
├── host_vars/
│   ├── desktop-tower.yml           # Frigate/InfluxDB config + Coral USB vars
│   └── mini-server.yml             # Immich/Portainer/Prometheus/Grafana vars
└── roles/
    ├── docker/                     # Docker Engine via official apt repo
    ├── monitoring-agents/          # node_exporter (systemd) + cAdvisor (Docker)
    ├── monitoring-server/          # Prometheus + Grafana (Docker)
    ├── portainer/                  # Portainer CE (Docker)
    ├── frigate-influx/             # Frigate NVR + InfluxDB (Docker Compose)
    └── immich/                     # Immich via official Docker Compose
```

---

## Quick Start

### 1. Install dependencies with uv

```bash
# Install Ansible Galaxy collections via uv
uv run ansible-galaxy collection install -r requirements.yml
```

### 2. Set up Ansible Vault

Create your vault password file (**never commit this**):

```bash
echo "your-secure-vault-password" > ~/.vault_pass
chmod 600 ~/.vault_pass
```

Create the encrypted vault file with your secrets:

```bash
ansible-vault create group_vars/all/vault.yml --vault-password-file ~/.vault_pass
```

Paste these values (with real passwords):

```yaml
vault_grafana_admin_password: "your-grafana-password"
vault_influxdb_admin_password: "your-influxdb-password"
vault_influxdb_admin_token: "your-influxdb-token"
vault_immich_db_password: "your-immich-db-password"
vault_ansible_user: "ansible"
```

### 3. Update host IPs

Edit [`host_vars/desktop-tower.yml`](host_vars/desktop-tower.yml) and [`host_vars/mini-server.yml`](host_vars/mini-server.yml) — replace the `host_ip` placeholder values with your actual static IPs.

### 4. Configure Coral USB + Frigate

In [`host_vars/desktop-tower.yml`](host_vars/desktop-tower.yml):

```yaml
frigate_devices:
  - "/dev/bus/usb:/dev/bus/usb"   # Coral USB — already set
# - "/dev/dri:/dev/dri"           # Uncomment if you also have a GPU

frigate_media_path: "/mnt/frigate"   # update to your actual media mount
frigate_config_path: "/opt/frigate/config"
```

You still need to write `frigate.yml` (Frigate's own config) and place it at `frigate_config_path`. Frigate's config is camera/detector-specific and outside the scope of this Ansible project.

### 5. Run using Makefile

```bash
# Perform a dry run (check & diff) across all hosts
make dry-run

# Perform a dry run on ONLY desktop-tower
make dry-run LIMIT=desktop-tower

# Deploy ONLY to desktop-tower
make deploy LIMIT=desktop-tower

# Ping only desktop-tower
make ping LIMIT=desktop-tower
```

Or run directly via `ansible-playbook`:

```bash
# Full run
ansible-playbook site.yml --vault-password-file ~/.vault_pass

# Dry run first (recommended)
ansible-playbook site.yml --vault-password-file ~/.vault_pass --check --diff

# Only a specific role/tag
ansible-playbook site.yml --vault-password-file ~/.vault_pass --tags monitoring
```

---

## Roles Reference

### `docker`
- Installs Docker Engine + Compose plugin from the **official Docker apt repo** (not distro packages)
- Adds `ansible_user` to the `docker` group
- Enables `docker.service` systemd unit
- Idempotent — safe to re-run

### `monitoring-agents` *(applied to both hosts)*
- **node_exporter** `v1.8.2` — binary installed to `/usr/local/bin`, runs as dedicated system user with systemd hardening
- **cAdvisor** `v0.49.1` — Docker container with required host path mounts for container metrics
- Versions pinned in [`group_vars/all.yml`](group_vars/all.yml)

### `monitoring-server` *(mini-server only)*
- **Prometheus** — Docker container, config templated from [`prometheus.yml.j2`](roles/monitoring-server/templates/prometheus.yml.j2). Scrape targets auto-generated from inventory `host_ip` + port variables. Docker volume for data persistence.
- **Grafana** — Docker container. Prometheus pre-provisioned as default datasource via [`grafana_datasource.yml.j2`](roles/monitoring-server/templates/grafana_datasource.yml.j2). No manual UI wiring needed.

### `portainer` *(mini-server only)*
- Portainer CE on ports `9000` (HTTP) and `9443` (HTTPS)
- Docker socket bind-mounted for full container management
- Named Docker volume for data persistence

### `frigate-influx` *(desktop-tower only)*
- Renders a `docker-compose.yml` from template to `/opt/frigate-influx/`
- Coral USB passthrough via `frigate_devices` variable list
- InfluxDB 2.x initialized with org/bucket/token on first run
- All secrets injected from vault

### `immich` *(mini-server only)*
- Downloads Immich's **official** `docker-compose.yml` from their GitHub releases (pinned to `immich_version`)
- Templates a `.env` file with DB credentials (from vault) and storage paths
- Does **not** reinvent Immich's compose — stays compatible with upstream changes

---

## Secrets Management

| Secret | Vault Variable | Used By |
|---|---|---|
| Grafana admin password | `vault_grafana_admin_password` | `monitoring-server` role |
| InfluxDB admin password | `vault_influxdb_admin_password` | `frigate-influx` role |
| InfluxDB operator token | `vault_influxdb_admin_token` | `frigate-influx` role |
| Immich DB password | `vault_immich_db_password` | `immich` role |
| Ansible SSH user | `vault_ansible_user` | `group_vars/all.yml` |

**Workflow**: `vault.yml` is encrypted → `vars.yml` references vault vars by name → roles use the plain var names. Only `vault.yml` ever contains the actual secret values.

---

## Useful Ansible Commands

```bash
# Check connectivity to all hosts
ansible all -m ping --vault-password-file ~/.vault_pass

# Run only on one host
ansible-playbook site.yml --limit desktop-tower --vault-password-file ~/.vault_pass

# Rotate a secret: edit vault, then re-run
ansible-vault edit group_vars/all/vault.yml --vault-password-file ~/.vault_pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass

# View current inventory
ansible-inventory --list --vault-password-file ~/.vault_pass | python3 -m json.tool
```

---

## Port Reference

| Service | Host | Port |
|---|---|---|
| node_exporter | both | 9100 |
| cAdvisor | both | 8080 |
| Prometheus | mini-server | 9090 |
| Grafana | mini-server | 3000 |
| Portainer HTTP | mini-server | 9000 |
| Portainer HTTPS | mini-server | 9443 |
| Immich | mini-server | 2283 |
| Frigate Web UI | desktop-tower | 5000 |
| Frigate RTSP | desktop-tower | 8554 |
| InfluxDB | desktop-tower | 8086 |

---

## Pinned Versions

Update these in [`group_vars/all.yml`](group_vars/all.yml) or the relevant `host_vars` file, then re-run the playbook.

| Component | Pinned Version |
|---|---|
| node_exporter | `1.8.2` |
| cAdvisor | `v0.49.1` |
| Prometheus | `v2.53.0` |
| Grafana | `11.1.0` |
| Portainer CE | `2.21.5` |
| InfluxDB | `1.11` |
| Immich | `release` *(pin to a tag like `v1.106.4`)* |
| Frigate | `stable` |
