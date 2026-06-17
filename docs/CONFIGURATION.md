# Configuration Reference

Full variable reference for the HAProxy + Keepalived role. All variables are defined with defaults in [`defaults/main.yml`](../defaults/main.yml).

---

## Table of Contents

- [Load Balancer Nodes](#load-balancer-nodes)
- [HAProxy Global](#haproxy-global)
- [HAProxy Statistics](#haproxy-statistics)
- [Frontends](#frontends)
- [Backends](#backends)
- [Listen Sections](#listen-sections)
- [Keepalived](#keepalived)
- [Cloud Floating IP](#cloud-floating-ip)
- [Advanced Examples](#advanced-examples)

---

## Load Balancer Nodes

| Variable | Default | Description |
|----------|---------|-------------|
| `loadbalancer_master` | `""` | `inventory_hostname` of the master proxy node. **Required.** |
| `loadbalancer_backup` | `""` | `inventory_hostname` of the backup proxy node. **Required.** |

The master receives Keepalived priority 100, the backup receives 90. Both run HAProxy; only the master holds the VIP at any time.

---

## HAProxy Global

| Variable | Default | Description |
|----------|---------|-------------|
| `haproxy_enabled` | `true` | Install and configure HAProxy |
| `haproxy_version` | `3.1.4` | Version to install from source |
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
  - http-server-close
  - forwardfor except 127.0.0.0/8
  - redispatch
```

---

## HAProxy Statistics

| Variable | Default | Description |
|----------|---------|-------------|
| `haproxy_stats` | `true` | Enable the statistics page |
| `haproxy_stats_port` | `1936` | Port to listen on |
| `haproxy_stats_bind_addr` | `0.0.0.0` | Bind address (`127.0.0.1` for localhost only) |
| `haproxy_stats_page_uri` | `/haproxy/stats` | URI path |
| `haproxy_stats_page_user` | `admin` | Username — **change this** |
| `haproxy_stats_page_pass` | `admin` | Password — **change this, use Vault** |

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

## Keepalived

| Variable | Default | Description |
|----------|---------|-------------|
| `keepalived_enabled` | `true` | Install and configure Keepalived |
| `keepalived_version` | `2.3.2` | Version to install from source |
| `keepalived_network_interface` | `eth0` | Interface to bind VRRP to — run `ip link` to confirm |
| `keepalived_unicast` | `true` | Use unicast instead of multicast (required for most cloud/VM environments) |

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

## Cloud Floating IP

Optional integration that moves a cloud provider floating IP via API when VRRP state changes.

| Variable | Default | Description |
|----------|---------|-------------|
| `cloud_floating_ip_enabled` | `false` | Enable cloud floating IP management |
| `cloud_api_endpoint` | `https://api.hetzner.cloud/v1` | Cloud provider API URL |
| `cloud_api_token` | `""` | API token — **store in Ansible Vault** |
| `floating_ip_address` | `""` | The floating IP to manage |
| `cloud_server_name` | `{{ inventory_hostname }}` | Server identifier for API calls |
| `cloud_floating_ip_interface` | `eth0` | Interface to assign floating IP to |

```yaml
cloud_floating_ip_enabled: true
cloud_api_endpoint: "https://api.hetzner.cloud/v1"
cloud_api_token: "{{ vault_cloud_api_token }}"
floating_ip_address: "95.217.1.100"
```

---

## Advanced Examples

### Full HA setup with HTTPS and health checks

```yaml
loadbalancer_master: "proxy-01"
loadbalancer_backup: "proxy-02"

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
