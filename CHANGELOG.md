# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased] — v2.0.0

> Breaking major release — not backward compatible with v1.x playbooks.

### Breaking Changes

- **Namespace:** `devopsgroup` → `devopsgroupeu` (Galaxy import required new namespace).
- **`min_ansible_version`:** raised to `2.19`.
- **`keepalived_vrrp_instances` no longer ships a stub default.** The variable is
  `required: true` in `meta/argument_specs.yml`; `validate.yml` enforces a valid
  IP on each instance. Callers that relied on the empty-stub default will now
  receive a clear validation error.
- **Install method default:** changed from `source` to `package`. Set
  `haproxy_install_method: source` / `keepalived_install_method: source` to
  restore source compilation.
- Removed phantom/ghost example vars `haproxy_stats_page_enabled`,
  `haproxy_stats_page_port`, `keepalived_vrrp_unicast` from examples (replaced
  with correct var names `haproxy_stats`, `haproxy_stats_port`, `keepalived_unicast`).

### Fixed

- **Critical:** `templates/haproxy.cfg.j2` — with `haproxy_journald: true` the
  global section now conditionally emits `log /dev/log` instead of the UDP
  rsyslog destination. Previously logs were silently dropped when journald mode
  was enabled.
- **Important:** `tasks/haproxy.yml` — SSL cert dir now uses
  `{{ haproxy_config_dir }}/certs` instead of the hardcoded `/etc/haproxy/certs`
  so overriding `haproxy_config_dir` works correctly end-to-end.
- `docs/CONFIGURATION.md`: corrected `haproxy_stats_bind_addr` default from
  `0.0.0.0` to `127.0.0.1`; corrected `haproxy_version` and `keepalived_version`
  defaults; removed phantom `http-server-close` from `haproxy_options` example.
- `README.md`: corrected `haproxy_stats_bind_addr` default; corrected failover
  description (`service haproxy status` → `/usr/bin/systemctl is-active --quiet haproxy`).
- `docs/INTEGRATION.md`: backend group `k8s_masters` → `server_nodes` (matches
  `devopsgroupeu.rke2` inventory group); added Galaxy install / requirements.yml
  section.
- `examples/cloud_with_floating_ip.yml`: replaced ghost vars `haproxy_stats_page_enabled`,
  `haproxy_stats_page_port`, `keepalived_vrrp_unicast` with correct names; updated
  `keepalived_version` to `2.3.4`.
- `examples/group_vars_haproxy_example.yml` + `examples/main_yml_haproxy_config.yml`:
  removed `http-server-close` from options; updated `keepalived_version` to `2.3.4`;
  corrected `haproxy_stats_bind_addr` to `127.0.0.1`.

### Added

- `.github/workflows/galaxy.yml` — SemVer-tag Galaxy auto-publish (was the only
  role missing this workflow).
- `.github/workflows/pre-commit.yml`, `release-drafter.yml` — parity with sibling
  roles.
- `.github/ISSUE_TEMPLATE/`, `PULL_REQUEST_TEMPLATE.md`, `RELEASE_DRAFTER.yml`,
  `dependabot.yml` — GitHub repository templates.
- `examples/inventory/proxy_hosts.ini` — example inventory showing the `proxy_hosts`
  group expected by this role.

### Changed

- `meta/main.yml`: description updated to reflect package-install-first default.
- `.ansible-lint`: added `profile: production` and canonical `enable_list`
  (`no-log-password`, `loop-var-prefix`).
- `.gitlab-ci.yml`: pinned to `python:3.12`; replaced ubuntu2204/debian11 with
  canonical supported matrix (ubuntu2404, debian12, debian13, rockylinux9);
  standardized yamllint invocation to explicit dir list.
- `.github/workflows/ci.yml`: collapsed two separate lint jobs into one pip-based
  `lint` job; added `package`, `ssl`, `cloud-fip` to molecule scenario matrix;
  added `concurrency` block.
- `requirements.txt`: unified pin `ansible-core>=2.19,<2.22` (was `>=2.19,<2.20`).
- Security contact standardized to `security@devopsgroup.sk`.

## [1.0.0] - 2026-02-04

### Added
- ✨ Initial production release
- ✅ HAProxy 3.1.4 installation from source with full compilation support
- ✅ Keepalived 2.3.2 installation from source for VRRP failover
- ✅ High availability configuration with automatic VIP failover
- ✅ Comprehensive variable structure in `defaults/main.yml` (340+ lines)
- ✅ Configuration validation in `tasks/validate.yml`
- ✅ Support for multiple HAProxy frontends and backends
- ✅ SSL/TLS certificate management
- ✅ HAProxy statistics page with authentication
- ✅ Health checks with configurable intervals
- ✅ Air-gapped installation support (auto-detects local tarballs)
- ✅ Molecule testing framework with Docker driver
- ✅ GitLab CI/CD pipeline with automated testing and Galaxy publication
- ✅ Support for Ubuntu 20.04/22.04/24.04
- ✅ Support for Debian 11/12
- ✅ Support for RHEL/Rocky/AlmaLinux 8/9
- 📚 Comprehensive README.md with quick start, examples, and troubleshooting
- 📚 Air-gapped installation documentation
- 📚 GitLab CI/CD setup guide
- 📚 Multiple configuration examples (web apps, APIs, databases, TCP services)
- 🔒 Security warnings for default credentials
- 🔒 Ansible Vault integration for sensitive data

### Features
- **Universal Design**: Works on bare metal, VMs, and cloud platforms
- **VRRP Protocol**: Unicast mode for cloud compatibility
- **Load Balancing Algorithms**: roundrobin, leastconn, source, uri
- **Session Persistence**: Cookie-based and source IP persistence
- **HTTP/TCP Modes**: Full HTTP inspection or TCP passthrough
- **Dynamic Backend Configuration**: Via inventory groups or manual lists
- **Rsyslog Integration**: Centralized logging for HAProxy and Keepalived
- **Systemd Integration**: Proper service management with dependencies

### Testing
- ✅ VIP failover tested and verified (< 3 second failover time)
- ✅ Idempotence verified (no changes on second run)
- ✅ Multi-distribution testing via Molecule
- ✅ Configuration validation tests
- ✅ Service health check tests
- ✅ yamllint and ansible-lint compliance

### Configuration
- **341 configurable variables** with comprehensive documentation
- **Security-first defaults** with warnings for credentials
- **Flexible frontend/backend definitions** supporting complex topologies
- **VRRP instance configuration** with priority and router ID management
- **Automatic master/backup detection** based on inventory hostnames

### Documentation
- Complete variable reference in `defaults/main.yml`
- Quick start guide in README.md
- Architecture diagrams
- Troubleshooting section
- Multiple real-world examples
- Air-gapped deployment guide
- CI/CD integration guide

### CI/CD
- GitLab CI pipeline with three stages: lint, test, release
- yamllint and ansible-lint validation
- Parallel Molecule tests on 4 distributions
- Automatic Ansible Galaxy publication on version tags
- Manual publication trigger for main branch

### Known Limitations
- Rocky Linux 9 Docker testing limited (PAM issues in containers)
- Real hardware/VM testing recommended for RHEL-family systems
- Role code fully supports RHEL/Rocky/AlmaLinux despite Docker test limitations

## [Unreleased]

### Planned
- Additional load balancing algorithms
- Enhanced monitoring integration (Prometheus, Grafana)
- Support for HAProxy 3.2.x when released
- Additional cloud provider integrations
- Extended air-gapped documentation
- Performance tuning guide

---

## Links

- [Ansible Galaxy](https://galaxy.ansible.com/devopsgroupeu/haproxy_keepalived)
- [GitHub Repository](https://github.com/devopsgroupeu/ansible-role-haproxy-keepalived)
- [Issues](https://github.com/devopsgroupeu/ansible-role-haproxy-keepalived/issues)
