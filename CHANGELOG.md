# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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

- [Ansible Galaxy](https://galaxy.ansible.com/devopsgroup/haproxy_keepalived)
- [GitLab Repository](https://gitlab.devopsgroup.sk/devopsgroupsk/on-premise-solution/ansible-roles/ansible-role-haproxy-keepalived)
- [Issues](https://gitlab.devopsgroup.sk/devopsgroupsk/on-premise-solution/ansible-roles/ansible-role-haproxy-keepalived/issues)
