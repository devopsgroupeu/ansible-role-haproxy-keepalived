# Integration

Guide for integrating HAProxy + Keepalived with various systems and cloud providers.

## Kubernetes / Container Platform Integration

### Kubernetes API Load Balancing

```yaml
haproxy_frontends:
  - name: k8s_api
    address: "*"
    port: 6443
    default_backend: k8s_masters
    mode: tcp

haproxy_backends:
  - name: k8s_masters
    balance: roundrobin
    mode: tcp
    servers: "{{ groups['k8s_masters'] | default([]) }}"
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
    servers: "{{ groups['k8s_masters'] | default([]) }}"
    port: 6443
    options:
      - check

  - name: k8s_join_backend
    balance: roundrobin
    mode: tcp
    servers: "{{ groups['k8s_masters'] | default([]) }}"
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
floating_ip_address: "95.217.27.127"
cloud_server_name: "{{ inventory_hostname }}"
```

**Requirements:**
- Hetzner Cloud account
- API token with Read & Write permissions
- Floating IP created
- Servers in same private network

### DigitalOcean

**Configuration:**
```yaml
cloud_floating_ip_enabled: true
cloud_api_endpoint: "https://api.digitalocean.com/v2"
cloud_api_token: "{{ vault_cloud_api_token }}"
floating_ip_address: "159.65.1.100"
cloud_server_name: "{{ inventory_hostname }}"
```

**Note:** Modify `/usr/local/bin/cloud-floating-ip-notify.sh` for DigitalOcean API format.

### AWS Elastic IP

For AWS, use AWS CLI or boto3 instead of the generic script. Example integration with AWS:

```bash
# Install AWS CLI on servers
apt install -y awscli

# Configure IAM role with EC2 permissions
# Modify notify script to use: aws ec2 associate-address
```

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
    port: 3306
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
    port: 5432
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
    ssl_enabled: true
    ssl_certificate: "/etc/haproxy/certs/example.com.pem"
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

The role compiles HAProxy and Keepalived from source. In air-gapped environments where outbound internet access is blocked, pre-download the source tarballs and place them in the `files/` directory of the role before running:

```bash
# On internet-connected machine
wget https://www.haproxy.org/download/3.1/src/haproxy-3.1.4.tar.gz
wget https://www.keepalived.org/software/keepalived-2.3.2.tar.gz

# Place in role files directory
cp haproxy-3.1.4.tar.gz keepalived-2.3.2.tar.gz /path/to/role/files/
```

The role will copy the tarballs from `files/` to the target servers instead of downloading them.

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
