# Configuration Reference

Full variable reference for the HAProxy + Keepalived role. All variables are defined with defaults in [`defaults/main.yml`](../defaults/main.yml).

---

## Table of Contents

- [Load Balancer Nodes](#load-balancer-nodes)
- [Install Method & Source Build](#install-method--source-build)
- [HAProxy Global](#haproxy-global)
- [SSL/TLS Hardening](#ssltls-hardening)
- [HAProxy Statistics](#haproxy-statistics)
- [Frontends](#frontends)
- [Backends](#backends)
- [Listen Sections](#listen-sections)
- [Raw Config Escape Hatches](#raw-config-escape-hatches)
- [Keepalived](#keepalived)
- [Keepalived VRRP Tuning](#keepalived-vrrp-tuning)
- [Keepalived VRRP Authentication](#keepalived-vrrp-authentication)
- [Cloud Floating IP](#cloud-floating-ip)
- [Directories](#directories)
- [Advanced Examples](#advanced-examples)

---

## Load Balancer Nodes

| Variable | Default | Description |
|----------|---------|-------------|
| `haproxy_loadbalancer_master` | `""` | `inventory_hostname` of the master proxy node. **Required.** |
| `haproxy_loadbalancer_backup` | `""` | `inventory_hostname` of the backup proxy node. **Required.** |

The master receives Keepalived priority 100, the backup receives 90. Both run HAProxy; only the master holds the VIP at any time.

---

## Install Method & Source Build

By default both components install from distribution/repo **packages**. Set the
`*_install_method` to `source` to compile from upstream tarballs (needed for the
latest versions, custom build flags, or air-gapped builds).

| Variable | Default | Description |
|----------|---------|-------------|
| `haproxy_install_method` | `package` | `package` (distro/repo) or `source` (compile) |
| `haproxy_branch` | `""` | Package-mode repo branch (e.g. `3.0`, `3.4`); empty = distro default |
| `haproxy_binary` | method-aware | Path to the haproxy binary (`/usr/local/sbin` source, `/usr/sbin` package) |
| `haproxy_src_sha256` | `""` | SHA256 of the source tarball — **required in source mode** (`sha256:<hex>` or a `.sha256` URL) |
| `haproxy_make_flags_default` | see `defaults/main.yml` | Base `make` flags for source builds |
| `haproxy_make_flags` | derived | Effective `make` flags (adds QUIC on HAProxy >= 2.9); override to fully customize |
| `haproxy_remove_build_deps` | `false` | Purge the HAProxy build toolchain after a source install |
| `haproxy_offline_install` | `false` | Copy a pre-staged tarball from the controller instead of downloading |
| `haproxy_src_local_path` | `""` | Controller path to the pre-staged HAProxy tarball (offline mode) |
| `keepalived_install_method` | `package` | `package` or `source` |
| `keepalived_binary` | method-aware | Path to the keepalived binary |
| `keepalived_src_sha256` | `""` | SHA256 of the source tarball — **required in source mode** |
| `keepalived_configure_flags_default` | see `defaults/main.yml` | Base `./configure` flags for source builds |
| `keepalived_configure_flags` | = default list | Effective `./configure` flags; override the whole list to customize |
| `keepalived_remove_build_deps` | `{{ haproxy_remove_build_deps }}` | Purge the Keepalived build toolchain after a source install |

---

## HAProxy Global

| Variable | Default | Description |
|----------|---------|-------------|
| `haproxy_enabled` | `true` | Install and configure HAProxy |
| `haproxy_version` | `3.4.0` | Version to install from source (LTS; package mode ignores this) |
| `haproxy_config` | `""` | Full config as a string — bypasses all template generation |
| `haproxy_user` | `haproxy` | System user for the HAProxy process |
| `haproxy_group` | `haproxy` | System group for the HAProxy process |
| `haproxy_log_facility` | `local2` | Syslog facility (`local0`–`local7`) |
| `haproxy_mode` | `http` | Default mode: `http` or `tcp` |
| `haproxy_maxconn` | `3000` | Maximum simultaneous connections |
| `haproxy_retries` | `3` | Connection retry attempts on failure |
| `haproxy_timeout_http_request` | `10s` | Max time to receive complete HTTP request |
| `haproxy_timeout_connect` | `10s` | Max time to connect to backend |
| `haproxy_timeout_client` | `1m` | Client inactivity timeout |
| `haproxy_timeout_server` | `1m` | Server inactivity timeout |
| `haproxy_timeout_http_keep_alive` | `10s` | Max time to wait for next HTTP request |
| `haproxy_timeout_check` | `10s` | Health check timeout |
| `haproxy_options` | see below | Default options list for all frontends/backends |

Default `haproxy_options`:
```yaml
haproxy_options:
  - httplog
  - dontlognull
  - forwardfor except 127.0.0.0/8
  - redispatch
```

| `haproxy_journald` | `false` | Send HAProxy logs to journald via `/dev/log` instead of rsyslog UDP |

---

## SSL/TLS Hardening

Applied to the global `ssl-default-bind-*` directives and per-bind `ssl-min-ver`.

| Variable | Default | Description |
|----------|---------|-------------|
| `haproxy_ssl_min_ver` | `TLSv1.2` | Minimum TLS version accepted on SSL binds |
| `haproxy_ssl_default_bind_ciphers` | see `defaults/main.yml` | Cipher list for TLS 1.2 and below |
| `haproxy_ssl_default_bind_ciphersuites` | see `defaults/main.yml` | Cipher suites for TLS 1.3 |

---

## HAProxy Statistics

| Variable | Default | Description |
|----------|---------|-------------|
| `haproxy_stats` | `true` | Enable the statistics page |
| `haproxy_stats_port` | `1936` | Port to listen on |
| `haproxy_stats_bind_addr` | `127.0.0.1` | Bind address — loopback by default; widen only if firewalled |
| `haproxy_stats_page_uri` | `/haproxy/stats` | URI path |
| `haproxy_stats_page_user` | `admin` | Username — **change this** |
| `haproxy_stats_page_pass` | `admin` | Password — **change this, use Vault** |
| `haproxy_stats_admin` | `false` | Enable live backend control (enable/disable/drain) from the stats UI — off by default; opt in explicitly |

> The role's validation will **fail** if `haproxy_stats_page_pass` is left as `"admin"`. Store it in Ansible Vault:
> ```yaml
> haproxy_stats_page_pass: "{{ vault_haproxy_stats_pass }}"
> ```

---

## Frontends

```yaml
haproxy_frontends:
  - name: http_frontend          # Required — unique name
    address: "*"                 # Required — bind address
    port: 80                     # Required — bind port
    default_backend: app_backend # Required — backend to route to
    mode: http                   # Optional — http (default) or tcp
    ssl: false                   # Optional — enable SSL termination
    crts: []                     # Optional — list of PEM cert paths (requires ssl: true)
    monitor_uri: /health         # Optional — expose a health endpoint on this frontend
    no_log: false                # Optional — disable logging for this frontend
```

### HTTPS frontend example

```yaml
haproxy_frontends:
  - name: https_frontend
    address: "*"
    port: 443
    default_backend: app_backend
    mode: http
    ssl: true
    crts:
      - /etc/haproxy/certs/example.com.pem
```

> PEM file must contain the full chain + private key concatenated:
> ```bash
> cat cert.crt intermediate.crt private.key > /etc/haproxy/certs/example.com.pem
> ```

### TCP passthrough example

```yaml
haproxy_frontends:
  - name: k8s_api_frontend
    address: "*"
    port: 6443
    default_backend: k8s_api_backend
    mode: tcp
```

---

## Backends

```yaml
haproxy_backend_default_balance: roundrobin

haproxy_backends:
  - name: app_backend            # Required — unique name
    servers: []                  # Required — see server specification below
    port: 8080                   # Required (unless specified per-server)
    mode: http                   # Optional — http (default) or tcp
    balance: roundrobin          # Optional — load balancing algorithm
    httpcheck: true              # Optional — enable HTTP health checks
    httpcheck_method: GET        # Optional — health check HTTP method
    http_check:                  # Optional — advanced health check
      send:
        method: GET
        uri: /health
      expect: status 200
    http_send_name_header: ""    # Optional — send server name in this header
    options:                     # Optional — per-server options
      - check
      - inter 2s
      - rise 2
      - fall 3
```

### Load balancing algorithms

| Value | Description |
|-------|-------------|
| `roundrobin` | Distributes requests evenly (default) |
| `leastconn` | Routes to server with fewest active connections (good for long-lived connections) |
| `source` | Routes based on client IP hash (session persistence without cookies) |
| `uri` | Routes based on request URI hash (good for caching proxies) |

### Server specification methods

**Method 1 — Inventory group (recommended):**
```yaml
servers: "{{ groups['app_servers'] | default([]) }}"
```
Requires `gather_facts: true`. IPs resolved automatically from inventory.

**Method 2 — Hostname list:**
```yaml
servers:
  - app-01
  - app-02
  - app-03
```

**Method 3 — Explicit IPs:**
```yaml
servers:
  - name: app-01
    address: 10.0.1.10
    port: 8080
  - name: app-02
    address: 10.0.1.11
    port: 8080
```

---

## Listen Sections

Combined frontend + backend in a single block. Less common — prefer separate `haproxy_frontends` and `haproxy_backends` for clarity.

```yaml
haproxy_listen_default_balance: roundrobin
haproxy_listens: []
```

---

## Raw Config Escape Hatches

For directives the structured variables do not cover. Content is appended
verbatim — only validated by `haproxy -c`.

| Variable | Default | Description |
|----------|---------|-------------|
| `haproxy_config` | `""` | Full config as a string — bypasses all template generation |
| `haproxy_global_extra` | `""` | Lines appended to the `global` section |
| `haproxy_defaults_extra` | `""` | Lines appended to the `defaults` section |
| `haproxy_custom_sections` | `""` | Appended verbatim at end of file (e.g. `peers` sections) |
| `haproxy_resolvers` | `[]` | Named DNS resolvers — list of `{name, nameservers: ["id ip:port", ...]}` |
| `haproxy_errorfiles` | `[]` | Custom error pages — list of `{code, path}` |

---

## Keepalived

| Variable | Default | Description |
|----------|---------|-------------|
| `keepalived_enabled` | `true` | Install and configure Keepalived |
| `keepalived_version` | `2.3.4` | Version to install from source (package mode ignores this) |
| `keepalived_network_interface` | `eth0` | Interface to bind VRRP to — run `ip link` to confirm |
| `keepalived_unicast` | `true` | Use unicast instead of multicast (required for most cloud/VM environments) |
| `keepalived_peer_group` | `proxy_hosts` | Inventory group used for automatic unicast peer discovery |
| `keepalived_unicast_peers` | `[]` | Explicit unicast peer IPs — overrides auto-discovery from `keepalived_peer_group` |
| `keepalived_log_facility` | `local1` | Syslog facility set via a systemd drop-in (`-S` flag) |
| `keepalived_haproxy_check_script` | `/usr/bin/systemctl is-active --quiet haproxy` | Track-script command (absolute path required under script security) |

### VRRP instances

```yaml
keepalived_vrrp_instances:
  - vrrp_instance_name: VI_1         # Required — unique identifier
    virtual_ipaddress: "10.0.0.100"  # Required — the floating VIP
    virtual_router_id: 51            # Required — 1-255, unique on the subnet
    priority: 100                    # Optional — auto-assigned from master/backup
    state: MASTER                    # Optional — auto-assigned from master/backup
```

Multiple VIPs:
```yaml
keepalived_vrrp_instances:
  - vrrp_instance_name: VI_1
    virtual_ipaddress: "10.0.0.100"
    virtual_router_id: 51
  - vrrp_instance_name: VI_2
    virtual_ipaddress: "10.0.0.101"
    virtual_router_id: 52
```

> `virtual_router_id` must be unique across all VRRP instances on the same network segment. Conflicts cause split-brain.

---

## Keepalived VRRP Tuning

VRRP timing, preemption, and sync-group behaviour (most are per-instance overridable).

| Variable | Default | Description |
|----------|---------|-------------|
| `keepalived_nopreempt` | `false` | When true, a recovered master will NOT reclaim the VIP (both nodes rendered as BACKUP) |
| `keepalived_preempt_delay` | `0` | Seconds to wait before preempting (`0` = instant; ignored when `nopreempt` is true) |
| `keepalived_advert_int` | `1` | VRRP advertisement interval in seconds |
| `keepalived_vrrp_version` | `2` | VRRP protocol version (`2` or `3`) |
| `keepalived_garp_master_delay` | `0` | Seconds before gratuitous ARP after becoming master (`0` = default) |
| `keepalived_garp_master_refresh` | `0` | Periodic gratuitous ARP refresh interval in seconds (`0` = disabled) |
| `keepalived_sync_groups` | `[]` | Sync groups — list of `{name, instances: [...]}` to fail instances over together |

---

## Keepalived VRRP Authentication

Native VRRP authentication (VRRPv2 only). Disabled by default.

| Variable | Default | Description |
|----------|---------|-------------|
| `keepalived_auth_type` | `""` | `PASS` or `AH`; empty = no authentication block rendered |
| `keepalived_auth_pass` | `""` | Shared secret (max 8 chars, cleartext in the VRRP header) — **store in Ansible Vault** |

```yaml
keepalived_auth_type: "PASS"
keepalived_auth_pass: "{{ vault_keepalived_auth_pass }}"
```

> `PASS` is cleartext on the wire and limited to 8 characters — combine with
> network segmentation / firewalling of VRRP (IP protocol 112).

---

## Cloud Floating IP

Optional integration that moves a cloud provider floating IP via API when VRRP state changes.

| Variable | Default | Description |
|----------|---------|-------------|
| `cloud_floating_ip_enabled` | `false` | Enable cloud floating IP management |
| `cloud_api_endpoint` | `https://api.hetzner.cloud/v1` | Cloud provider API URL |
| `cloud_api_token` | `""` | API token — **store in Ansible Vault** |
| `cloud_floating_ip_address` | `""` | The floating IP to manage |
| `cloud_server_name` | `{{ inventory_hostname }}` | Server identifier for API calls |
| `cloud_floating_ip_interface` | `eth0` | Interface to assign floating IP to |
| `cloud_floating_ip_log` | `/var/log/cloud-floating-ip.log` | Log file for the notify script (raw API bodies are not logged) |

```yaml
cloud_floating_ip_enabled: true
cloud_api_endpoint: "https://api.hetzner.cloud/v1"
cloud_api_token: "{{ vault_cloud_api_token }}"
cloud_floating_ip_address: "95.217.1.100"
```

---

## Directories

Installation and configuration paths. Generally safe to leave at defaults unless
you have specific filesystem requirements.

| Variable | Default | Description |
|----------|---------|-------------|
| `haproxy_config_dir` | `/etc/haproxy` | HAProxy config directory (`haproxy.cfg`, `certs/`) |
| `haproxy_socket_dir` | `/var/lib/haproxy` | Runtime directory for Unix sockets and chroot |
| `haproxy_admin_socket_dir` | `/run/haproxy` | Admin runtime socket dir — must live OUTSIDE `haproxy_socket_dir` (chroot) |
| `haproxy_admin_socket` | `{{ haproxy_admin_socket_dir }}/admin.sock` | Admin socket path for hitless reloads |
| `haproxy_hard_stop_after` | `30s` | Force-stop old workers this long after a reload (drains long-lived connections) |
| `haproxy_install_dir` | `/usr/local/src` | Source compilation directory (source mode only) |
| `haproxy_ssl_cert_src` | `""` | Controller dir of SSL certs to deploy into `{{ haproxy_config_dir }}/certs/`; empty = none |
| `keepalived_config_dir` | `/etc/keepalived` | Keepalived config directory (`keepalived.conf`) |

---

## Advanced Examples

### Full HA setup with HTTPS and health checks

```yaml
haproxy_loadbalancer_master: "proxy-01"
haproxy_loadbalancer_backup: "proxy-02"

keepalived_enabled: true
keepalived_network_interface: eth0
keepalived_vrrp_instances:
  - vrrp_instance_name: VI_1
    virtual_ipaddress: "10.0.0.100"
    virtual_router_id: 51

haproxy_stats: true
haproxy_stats_page_pass: "{{ vault_haproxy_stats_pass }}"

haproxy_frontends:
  - name: http_frontend
    address: "*"
    port: 80
    default_backend: app_backend
    mode: http
    monitor_uri: /health

  - name: https_frontend
    address: "*"
    port: 443
    default_backend: app_backend
    mode: http
    ssl: true
    crts:
      - /etc/haproxy/certs/example.com.pem

haproxy_backends:
  - name: app_backend
    balance: leastconn
    mode: http
    http_check:
      send:
        method: GET
        uri: /api/health
      expect: status 200
    servers: "{{ groups['app_servers'] }}"
    port: 8080
    options:
      - check
      - inter 5s
      - rise 2
      - fall 3
```

### TCP load balancer (database, Kubernetes API)

```yaml
haproxy_frontends:
  - name: postgres_frontend
    address: "*"
    port: 5432
    default_backend: postgres_backend
    mode: tcp

haproxy_backends:
  - name: postgres_backend
    balance: leastconn
    mode: tcp
    servers:
      - name: db-01
        address: 10.0.2.10
        port: 5432
      - name: db-02
        address: 10.0.2.11
        port: 5432
    options:
      - check
      - inter 10s
```
