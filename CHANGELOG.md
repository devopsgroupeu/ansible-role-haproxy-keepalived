# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Fixed
- Cloud Floating IP integration now runs **before** Keepalived starts, so the
  notify script it deploys exists when Keepalived first reads its config. Under
  `enable_script_security`, the previous order made Keepalived refuse to start.
- The Keepalived log-facility systemd drop-in keeps `--dont-fork`. Without it the
  packaged `Type=notify` unit forks, the readiness notification comes from a
  child PID, and systemd fails the service start.
- HAProxy is now **restarted** (not soft-reloaded) on a config change. A soft
  reload over the package's default auto-started process could leave the new
  frontends unbound (no listening sockets).

### Changed
- Documentation is provider-agnostic. The role runs on any VMs; the floating-IP
  failover feature notes that its bundled notify script targets the Hetzner Cloud
  API (swap the script for other providers), and provider names otherwise appear
  only as illustrative caveats.

## [1.0.0] - 2026-06-23

Initial public release on the devopsgroupeu Ansible Galaxy namespace. Installs and
configures HAProxy with Keepalived in active-passive high availability: package
install by default (source compilation opt-in), VRRP-managed VIP with automatic
failover, SSL/TLS termination, statistics, cloud floating-IP failover, and
air-gapped support.

### Notes for users of the legacy devopsgroup.* roles
- Namespace changed from `devopsgroup` to `devopsgroupeu` — update your
  `requirements.yml` role source and any `meta/main.yml` dependency references.
- `min_ansible_version` is `2.19`.
- Install method default changed from `source` to `package`. Set
  `haproxy_install_method: source` / `keepalived_install_method: source` to
  restore source compilation.
- `keepalived_vrrp_instances` no longer ships a stub default; `validate.yml`
  enforces a valid VIP on each instance when keepalived is enabled.
- Example variable names corrected to the names the role actually reads:
  `haproxy_stats`, `haproxy_stats_port`, `keepalived_unicast`.

> Pre-Galaxy internal development history (v1.0.0 under the legacy `devopsgroup`
> namespace) is preserved in the git history.

---

## Links

- [Ansible Galaxy](https://galaxy.ansible.com/ui/standalone/roles/devopsgroupeu/haproxy-keepalived/)
- [GitHub Repository](https://github.com/devopsgroupeu/ansible-role-haproxy-keepalived)
- [Issues](https://github.com/devopsgroupeu/ansible-role-haproxy-keepalived/issues)
