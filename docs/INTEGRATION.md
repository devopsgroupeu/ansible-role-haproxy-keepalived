# Integration

Guide for integrating HAProxy + Keepalived with various systems and cloud providers.

## Consuming via requirements.yml

Install from Ansible Galaxy:

```bash
ansible-galaxy role install devopsgroupeu.haproxy-keepalived
```

Or pin a version via `requirements.yml`:

```yaml
---
roles:
  - name: devopsgroupeu.haproxy-keepalived
    src: https://github.com/devopsgroupeu/ansible-role-haproxy-keepalived
    version: "v1.0.0"
    scm: git
```

This role is typically used together with [`devopsgroupeu.rke2`](https://github.com/devopsgroupeu/ansible-role-rke2):
the floating VIP defined in `keepalived_vrrp_instances[].virtual_ipaddress` becomes the
address RKE2 agents and external clients use to reach the control-plane (the RKE2 server URL).

## Kubernetes / Container Platform Integration

### Kubernetes API Load Balancing

Hosts in the `server_nodes` inventory group are the RKE2 control-plane nodes.

```yaml
haproxy_frontends:
  - name: k8s_api
    address: "*"
    port: 6443
    default_backend: k8s_api_servers
    mode: tcp

haproxy_backends:
  - name: k8s_api_servers
    balance: roundrobin
    mode: tcp
    servers: "{{ groups['server_nodes'] | default([]) }}"
    port: 6443
    options:
      - check
      - check-ssl
      - verify none
```

### Kubernetes with Multiple Ports

```yaml
haproxy_frontends:
  - name: k8s_api
    address: "*"
    port: 6443
    default_backend: k8s_api_backend
    mode: tcp

  - name: k8s_join
    address: "*"
    port: 9345
    default_backend: k8s_join_backend
    mode: tcp

haproxy_backends:
  - name: k8s_api_backend
    balance: roundrobin
    mode: tcp
    servers: "{{ groups['server_nodes'] | default([]) }}"
    port: 6443
    options:
      - check

  - name: k8s_join_backend
    balance: roundrobin
    mode: tcp
    servers: "{{ groups['server_nodes'] | default([]) }}"
    port: 9345
    options:
      - check
```

## Cloud Provider Integration

### Hetzner Cloud

**Configuration:**
```yaml
cloud_floating_ip_enabled: true
cloud_api_endpoint: "https://api.hetzner.cloud/v1"
cloud_api_token: "{{ vault_cloud_api_token }}"
cloud_floating_ip_address: "95.217.27.127"
cloud_server_name: "{{ inventory_hostname }}"
```

**Requirements:**
- Hetzner Cloud account
- API token with Read & Write permissions
- Floating IP created
- Servers in same private network

### Other cloud providers

The built-in floating-IP notify script targets the Hetzner Cloud API only.
For other providers (DigitalOcean, AWS Elastic IP, etc.) you must supply your
own notify script — set `cloud_floating_ip_enabled: false` and call your
custom script from a Keepalived `notify` stanza instead.

## Database Load Balancing

### MySQL/MariaDB

```yaml
haproxy_listens:
  - name: mysql
    address: "*"
    port: 3306
    mode: tcp
    balance: leastconn
    servers:
      - mysql01.local
      - mysql02.local
    options:
      - check
```

### PostgreSQL

```yaml
haproxy_listens:
  - name: postgresql
    address: "*"
    port: 5432
    mode: tcp
    balance: leastconn
    servers:
      - pg01.local
      - pg02.local
    options:
      - check
```

## Web Application Integration

### Multiple Applications

```yaml
haproxy_frontends:
  - name: http_frontend
    address: "*"
    port: 80
    mode: http
    default_backend: app1_backend

haproxy_backends:
  - name: app1_backend
    balance: roundrobin
    mode: http
    servers:
      - app1-server1.local
      - app1-server2.local
    port: 8080
    options:
      - check

  - name: app2_backend
    balance: roundrobin
    mode: http
    servers:
      - app2-server1.local
      - app2-server2.local
    port: 8080
    options:
      - check
```

## SSL/TLS Integration

### Let's Encrypt with HAProxy

```yaml
haproxy_frontends:
  - name: https_frontend
    address: "*"
    port: 443
    default_backend: web_backend
    mode: http
    ssl: true
    crts:
      - /etc/haproxy/certs/example.com.pem
```

**Certificate format:**
```bash
cat fullchain.pem privkey.pem > /etc/haproxy/certs/example.com.pem
chmod 600 /etc/haproxy/certs/example.com.pem
```

## Monitoring Integration

### Prometheus Exporter

Install HAProxy exporter:
```bash
wget https://github.com/prometheus/haproxy_exporter/releases/download/v0.15.0/haproxy_exporter-0.15.0.linux-amd64.tar.gz
tar xvf haproxy_exporter-0.15.0.linux-amd64.tar.gz
cp haproxy_exporter-0.15.0.linux-amd64/haproxy_exporter /usr/local/bin/

# Run exporter
haproxy_exporter --haproxy.scrape-uri="http://admin:password@localhost:1936/haproxy/stats;csv"
```

### Grafana Dashboard

Use HAProxy stats page or Prometheus metrics for Grafana.

## Ansible Integration Examples

### With Inventory Groups

```yaml
haproxy_backends:
  - name: all_servers
    balance: roundrobin
    mode: http
    servers: "{{ groups['web_servers'] | default([]) }}"
    port: 80
    options:
      - check
```

### With Variables

```yaml
haproxy_backends:
  - name: dynamic_backend
    balance: roundrobin
    mode: http
    servers: "{{ backend_servers }}"
    port: "{{ backend_port }}"
    options:
      - check
```

## Air-Gapped Environments

Package mode is the default. For an air-gapped source build of HAProxy where
outbound internet access is blocked, pre-download the HAProxy source tarball on
the Ansible controller and point the role at it with the offline variables:

```bash
# On an internet-connected machine
wget https://www.haproxy.org/download/3.4/src/haproxy-3.4.0.tar.gz
```

```yaml
# group_vars or host_vars
haproxy_install_method: source
haproxy_offline_install: true
haproxy_src_local_path: "/path/on/controller/haproxy-3.4.0.tar.gz"
```

The role copies the tarball from the controller path to `haproxy_install_dir` on
the target servers and then proceeds with the normal compile/install flow.
A checksum (`haproxy_src_sha256`) is still required even in offline mode.

> Offline copy-from-controller staging is implemented for HAProxy only. For an
> air-gapped Keepalived install, host the source tarball on a reachable internal
> mirror and set `keepalived_install_method: source` with `keepalived_src_sha256`,
> or install Keepalived via `keepalived_install_method: package` from an internal
> package repository.

## High Availability Patterns

### Active-Passive

Default configuration with one master, one backup:

```yaml
haproxy_loadbalancer_master: "proxy-01"
haproxy_loadbalancer_backup: "proxy-02"
```

### Active-Active

Both servers handle traffic with VIP failover:

```yaml
keepalived_vrrp_instances:
  - vrrp_instance_name: VI_1
    virtual_ipaddress: "192.168.1.100"
    virtual_router_id: 51
```

Both HAProxy instances are always active, VRRP only manages VIP assignment.

## Additional Resources

- [HAProxy Documentation](https://www.haproxy.com/documentation/)
- [Keepalived Documentation](https://keepalived.readthedocs.io/)
- [Configuration Guide](CONFIGURATION.md)
- [Troubleshooting Guide](TROUBLESHOOTING.md)
