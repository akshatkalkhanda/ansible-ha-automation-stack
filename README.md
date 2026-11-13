# ansible-ha-stack

A production-grade Ansible project for deploying a highly available web infrastructure with load balancing, automatic failover, and full observability.

## Architecture Overview

```
                         ┌────────────────────────────────────┐
                         │          Virtual IP (VIP)          │
                         │       192.168.10.10 (VRRP)         │
                         └──────────────┬─────────────────────┘
                                        │
              ┌─────────────────────────┴──────────────────────────┐
              │                                                    │
   ┌──────────▼──────────┐                           ┌──────────────▼──────────┐
   │    haproxy1          │  Keepalived VRRP         │    haproxy2             │
   │  192.168.10.11       │◄────────────────────────►│  192.168.10.12          │
   │  MASTER (prio 110)   │  heartbeat               │  BACKUP  (prio 100)     │
   └──────────┬──────────┘                           └──────────────┬──────────┘
              │                                                     │
              │        URL-based ACL routing                        │
              │  /api     → backend_api                             │
              │  /static  → backend_static                          │
              │  /admin   → backend_admin                           │
              │                                                     │
   ┌──────────┴─────────────────────────────────────────────────────┘
   │
   │
   ├──► backend_api      ──► app1:8080  app2:8080  app3:8080
   ├──► backend_static   ──► app1:8081  app2:8081
   └──► backend_admin    ──► app1:8082

   ┌──────────────────────────────────────────────────────────────────┐
   │                   Monitoring Host (monitor1)                      │
   │                      192.168.10.31                                │
   │                                                                   │
   │   Prometheus :9090  ◄── scrapes node_exporter :9100 on ALL hosts │
   │   Grafana    :3000  ◄── Prometheus datasource auto-provisioned    │
   └──────────────────────────────────────────────────────────────────┘

   node_exporter :9100 runs on ALL hosts (haproxy1, haproxy2, app1-3, monitor1)
```

## Features

- **HAProxy** with URL-based ACL routing to multiple backend pools
- **Keepalived** VRRP for automatic VIP failover (MASTER/BACKUP)
- **Prometheus** with dynamic scrape target configuration via Jinja2
- **Grafana** with Prometheus datasource auto-provisioned via Ansible URI module (no manual UI steps)
- **Node Exporter** deployed on every host for system metrics
- Fully **variable-driven** — no hardcoded values in tasks or templates
- **Idempotent** — safe to re-run; all tasks use `state:` and version guards
- **Tag-scoped** — run only the components you need
- **`--check` dry-run** support out of the box

---

## Prerequisites

| Requirement         | Minimum Version |
|---------------------|-----------------|
| Ansible             | 2.14+           |
| Python              | 3.9+            |
| Target OS           | Ubuntu 22.04 / Debian 12 |
| SSH access          | Key-based to all target hosts |
| `ansible.builtin`   | Included with Ansible core |

Install Ansible on your control node:

```bash
pip install ansible
```

---

## Project Structure

```
ansible-ha-stack/
├── ansible.cfg                   # Ansible runtime defaults
├── inventory/
│   └── hosts.ini                 # Host inventory with groups and per-host vars
├── group_vars/
│   ├── all.yml                   # Variables for every host
│   ├── haproxy.yml               # HAProxy + Keepalived variables
│   └── monitoring.yml            # Prometheus + Grafana variables
├── playbooks/
│   ├── site.yml                  # Master playbook — runs everything
│   ├── haproxy.yml               # HAProxy + Keepalived only
│   └── monitoring.yml            # node_exporter + Prometheus + Grafana
└── roles/
    ├── haproxy/
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/haproxy.cfg.j2
    ├── keepalived/
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/keepalived.conf.j2
    ├── node_exporter/
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/node_exporter.service.j2
    ├── prometheus/
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/
    │       ├── prometheus.yml.j2
    │       └── prometheus.service.j2
    └── grafana/
        ├── defaults/main.yml
        ├── tasks/main.yml
        └── handlers/main.yml
```

---

## Variable Reference

### group_vars/all.yml — Global

| Variable | Default | Description |
|---|---|---|
| `ansible_user` | `ubuntu` | SSH login user |
| `ansible_python_interpreter` | `/usr/bin/python3` | Python binary on targets |
| `node_exporter_version` | `1.8.1` | Node Exporter release version |
| `node_exporter_user` | `node_exporter` | System user for Node Exporter |
| `node_exporter_group` | `node_exporter` | System group for Node Exporter |
| `node_exporter_install_dir` | `/usr/local/bin` | Binary installation path |
| `node_exporter_port` | `9100` | Listening port |
| `ntp_timezone` | `UTC` | System timezone |

### group_vars/haproxy.yml — HAProxy

| Variable | Default | Description |
|---|---|---|
| `haproxy_package` | `haproxy` | apt package name |
| `haproxy_config_path` | `/etc/haproxy/haproxy.cfg` | Config file path |
| `haproxy_maxconn` | `50000` | Global max connections |
| `haproxy_timeout_connect` | `5s` | Connection timeout |
| `haproxy_timeout_client` | `30s` | Client inactivity timeout |
| `haproxy_timeout_server` | `30s` | Server response timeout |
| `haproxy_frontend_port` | `80` | Frontend listener port |
| `haproxy_frontend_name` | `http_front` | Frontend name |
| `haproxy_default_backend` | `backend_api` | Default backend when no ACL matches |
| `haproxy_stats_enabled` | `true` | Enable stats endpoint |
| `haproxy_stats_port` | `8404` | Stats listener port |
| `haproxy_stats_uri` | `/stats` | Stats URI |
| `haproxy_stats_auth_user` | `admin` | Stats basic-auth user |
| `haproxy_stats_auth_pass` | `required (Vault)` | Stats basic-auth password |
| `haproxy_acl_routes` | `[]` | List of ACL routing rules (see below) |
| `haproxy_backends` | `[]` | List of backend pool definitions (see below) |

**`haproxy_acl_routes` entry fields:**

| Field | Description |
|---|---|
| `acl_name` | ACL rule name (e.g. `is_api`) |
| `path_prefix` | URL path prefix to match (e.g. `/api`) |
| `backend_name` | Backend pool to route to |

**`haproxy_backends` entry fields:**

| Field | Description |
|---|---|
| `name` | Backend name |
| `balance` | Load-balancing algorithm (`roundrobin`, `leastconn`, `source`) |
| `options` | List of HAProxy option directives |
| `servers` | List of server entries (`name`, `host`, `port`, `options`) |

### group_vars/haproxy.yml — Keepalived

| Variable | Default | Description |
|---|---|---|
| `keepalived_package` | `keepalived` | apt package name |
| `keepalived_vip` | `192.168.10.10` | Virtual IP address |
| `keepalived_vip_cidr` | `24` | VIP subnet prefix length |
| `keepalived_interface` | `eth0` | Network interface for VRRP |
| `keepalived_vrrp_instance_name` | `VI_1` | VRRP instance identifier |
| `keepalived_vrrp_id` | `51` | Virtual router ID (1–255, unique per LAN) |
| `keepalived_auth_pass` | `required (Vault)` | VRRP authentication passphrase |
| `keepalived_check_script` | `/usr/local/bin/check_haproxy.sh` | Health check script path |
| `keepalived_check_interval` | `2` | Check interval (seconds) |
| `keepalived_check_fall` | `3` | Failures before FAULT state |
| `keepalived_check_rise` | `2` | Successes before OK state |
| `keepalived_notify_script` | `""` | Notification script (blank to disable) |

**Per-host inventory variables (set in `inventory/hosts.ini`):**

| Variable | MASTER value | BACKUP value | Description |
|---|---|---|---|
| `keepalived_state` | `MASTER` | `BACKUP` | Initial VRRP state |
| `keepalived_priority` | `110` | `100` | VRRP election priority |

### group_vars/monitoring.yml — Prometheus

| Variable | Default | Description |
|---|---|---|
| `prometheus_version` | `2.52.0` | Prometheus release version |
| `prometheus_install_dir` | `/usr/local/bin` | Binary path |
| `prometheus_config_dir` | `/etc/prometheus` | Config directory |
| `prometheus_data_dir` | `/var/lib/prometheus` | TSDB data directory |
| `prometheus_port` | `9090` | HTTP listen port |
| `prometheus_retention_time` | `30d` | Time-series retention period |
| `prometheus_log_level` | `info` | Log verbosity |
| `prometheus_global_scrape_interval` | `15s` | Default scrape interval |
| `prometheus_global_evaluation_interval` | `15s` | Rule evaluation interval |
| `prometheus_scrape_configs` | `[]` | List of scrape job definitions |
| `prometheus_rule_files` | `[]` | Alert/recording rule files |
| `prometheus_alertmanager_url` | `""` | Alertmanager URL (blank to disable) |

**`prometheus_scrape_configs` entry fields:**

| Field | Description |
|---|---|
| `job_name` | Scrape job name |
| `targets` | List of `host:port` strings |
| `labels` | Key-value map of static labels |

### group_vars/monitoring.yml — Grafana

| Variable | Default | Description |
|---|---|---|
| `grafana_package` | `grafana` | apt package name |
| `grafana_port` | `3000` | HTTP listen port |
| `grafana_admin_user` | `admin` | Admin username |
| `grafana_admin_password` | `required (Vault)` | Admin password (use vault in production) |
| `grafana_allow_signup` | `false` | Allow self-registration |
| `grafana_datasource_name` | `Prometheus` | Datasource display name |
| `grafana_datasource_url` | `http://localhost:9090` | Prometheus URL for Grafana |
| `grafana_datasource_access` | `proxy` | Access mode (`proxy` or `direct`) |
| `grafana_datasource_is_default` | `true` | Set as default datasource |
| `grafana_startup_wait` | `15` | Seconds between API readiness retries |
| `grafana_startup_retries` | `10` | Maximum readiness retry count |

---

## Usage

### 1. Configure your inventory

Edit `inventory/hosts.ini` to reflect your real host IPs:

```ini
[haproxy]
haproxy1 ansible_host=<IP1> keepalived_state=MASTER keepalived_priority=110
haproxy2 ansible_host=<IP2> keepalived_state=BACKUP keepalived_priority=100

[app_servers]
app1 ansible_host=<IP3>

[monitoring]
monitor1 ansible_host=<IP4>
```

### 2. Customise variables

Edit `group_vars/all.yml`, `group_vars/haproxy.yml`, and `group_vars/monitoring.yml`
to adjust VIPs, backend addresses, passwords, versions, etc.

> **Security note:** Store secrets (`keepalived_auth_pass`, `grafana_admin_password`,
> `haproxy_stats_auth_pass`) in an Ansible Vault file rather than plain text:
>
> ```bash
> ansible-vault encrypt_string 'mysecret' --name 'grafana_admin_password'
> ```

### 3. Run the full stack

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml
```

### 4. Run a specific component only

```bash
# Only HAProxy + Keepalived
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --tags haproxy,keepalived

# Only node_exporter
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --tags node_exporter

# Only Prometheus
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --tags prometheus

# Only Grafana
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --tags grafana
```

### 5. Syntax check and dry-run (check mode)

```bash
# Static syntax check (no SSH connection required)
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --syntax-check

# Full dry-run — shows every change that would be made without applying it
# Note: binary-install tasks and Grafana API provisioning are safely skipped
# during check mode; template diffs are still rendered via --diff.
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --check --diff
```

### 6. Limit to specific hosts

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --limit haproxy1
```

### 7. Use Ansible Vault for secrets

```bash
# Encrypt an existing variable file
ansible-vault encrypt group_vars/monitoring.yml

# Run with vault password
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --ask-vault-pass

# Or use a vault password file
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --vault-password-file ~/.vault_pass
```

### 8. Verify connectivity

```bash
ansible all -i inventory/hosts.ini -m ping
```

---

## Roles Summary

| Role | Target Group | Tag | Description |
|---|---|---|---|
| `haproxy` | `haproxy` | `haproxy` | Install and configure HAProxy with ACL routing |
| `keepalived` | `haproxy` | `keepalived` | Install Keepalived with VRRP VIP failover |
| `node_exporter` | `all` | `node_exporter` | Install Prometheus Node Exporter |
| `prometheus` | `monitoring` | `prometheus` | Install and configure Prometheus |
| `grafana` | `monitoring` | `grafana` | Install Grafana + auto-provision Prometheus datasource |

---

## Security Considerations

- Change all default passwords before deploying to production
- Use Ansible Vault to encrypt secrets in `group_vars`
- Restrict the HAProxy stats endpoint (`haproxy_stats_port`) at the firewall level
- Use a dedicated VRRP authentication password unique to your network
- Consider enabling TLS on HAProxy frontend for production traffic
- The Grafana admin password and datasource credentials should be rotated regularly

---

## Author

**Created by Akshat Kalkhanda**
- **Git** : https://github.com/akshatkalkhanda/ansible-ha-automation-stack
- **Email**: akshatkalkhanda@gmail.com

---

## License

MIT

Credential requirements: [private deployment credentials](docs/credential-security.md).
