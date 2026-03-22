# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-03-22

### Added

#### Project Structure
- Top-level `ansible-ha-stack/` project directory
- `ansible.cfg` with pipelining, YAML callback, and SSH multiplexing defaults
- `inventory/hosts.ini` with `haproxy`, `app_servers`, and `monitoring` groups
- Per-host Keepalived state/priority variables in inventory (`keepalived_state`, `keepalived_priority`)

#### group_vars
- `group_vars/all.yml` — global variables covering node_exporter version/paths and system defaults
- `group_vars/haproxy.yml` — full HAProxy (ACL routes, backends, stats, timeouts) and Keepalived (VIP, VRRP, auth, health-check) configuration
- `group_vars/monitoring.yml` — Prometheus (scrape configs, retention, binary version) and Grafana (admin credentials, datasource URL, API provisioning) configuration

#### Role: haproxy
- `defaults/main.yml` — all HAProxy role defaults
- `tasks/main.yml` — apt install, config directory, template deploy with `haproxy -c` validation, service enable/start
- `handlers/main.yml` — restart HAProxy on config change
- `templates/haproxy.cfg.j2` — Jinja2 template with global/defaults blocks, stats frontend, main HTTP frontend with ACL loop, and backend pool loop with per-backend options and servers

#### Role: keepalived
- `defaults/main.yml` — all Keepalived role defaults
- `tasks/main.yml` — apt install, config directory, health-check script deployment, config template deploy, service enable/start
- `handlers/main.yml` — restart Keepalived on config change
- `templates/keepalived.conf.j2` — VRRP instance with per-host MASTER/BACKUP state, priority 110/100, auth passphrase, VIP definition, and `track_script` for HAProxy health

#### Role: node_exporter
- `defaults/main.yml` — all Node Exporter role defaults
- `tasks/main.yml` — system user/group creation, textfile collector directory, version-guarded binary download/install, systemd unit deployment, service enable/start
- `handlers/main.yml` — systemd daemon reload and node_exporter restart
- `templates/node_exporter.service.j2` — systemd unit file with variable listen port and collector flags

#### Role: prometheus
- `defaults/main.yml` — all Prometheus role defaults
- `tasks/main.yml` — system user/group creation, directory setup, version-guarded binary download/install with promtool, config deploy with `promtool check config` validation, systemd unit deploy, service enable/start
- `handlers/main.yml` — systemd daemon reload and Prometheus restart
- `templates/prometheus.yml.j2` — Jinja2 template looping over `prometheus_scrape_configs` for dynamic scrape targets and labels
- `templates/prometheus.service.j2` — systemd unit file with full flag set (config, data dir, retention, port, log level)

#### Role: grafana
- `defaults/main.yml` — all Grafana role defaults
- `tasks/main.yml` — GPG key + apt repo setup, package install, config file patching, service enable/start, API readiness wait loop, idempotent datasource create/update via `ansible.builtin.uri`
- `handlers/main.yml` — restart Grafana on config change

#### Playbooks
- `playbooks/site.yml` — master playbook importing `monitoring.yml` then `haproxy.yml` with full tag support
- `playbooks/haproxy.yml` — targets `haproxy` group, applies `haproxy` and `keepalived` roles with `become: true`
- `playbooks/monitoring.yml` — three plays: node_exporter on `all`, Prometheus on `monitoring`, Grafana on `monitoring`

#### Documentation
- `README.md` — ASCII architecture diagram, prerequisites table, full project structure tree, complete variable reference table for all five roles, step-by-step usage commands (full run, tag-scoped runs, dry-run, vault, connectivity check), roles summary table, and security considerations
- `CHANGELOG.md` — this file, starting at v1.0.0

[1.0.0]: https://github.com/akshatkalkhanda/ansible-ha-automation-stack/releases/tag/v1.0.0