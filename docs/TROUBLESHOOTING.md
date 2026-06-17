# Troubleshooting

Common issues and how to resolve them.

---

## Table of Contents

- [HAProxy fails to start](#haproxy-fails-to-start)
- [Keepalived fails to start](#keepalived-fails-to-start)
- [VIP not assigned to master node](#vip-not-assigned-to-master-node)
- [VIP not failing over to backup](#vip-not-failing-over-to-backup)
- [Backend servers marked as DOWN](#backend-servers-marked-as-down)
- [Stats page not accessible](#stats-page-not-accessible)
- [SSL certificate errors](#ssl-certificate-errors)
- [Cloud floating IP not reassigning](#cloud-floating-ip-not-reassigning)
- [Useful commands reference](#useful-commands-reference)

---

## HAProxy fails to start

**Symptom:** `systemctl status haproxy` shows `failed`.

```bash
# Check syntax before starting
haproxy -c -f /etc/haproxy/haproxy.cfg

# View logs
journalctl -u haproxy -n 100 --no-pager
```

**Common causes:**

| Error | Cause | Fix |
|-------|-------|-----|
| `cannot bind socket [0.0.0.0:80]` | Port already in use | `ss -tlnp \| grep :80` — stop the conflicting process |
| `cannot open the file '/etc/haproxy/certs/...'` | SSL cert path wrong or missing | Verify `crts` paths exist on the target host |
| `parsing [haproxy.cfg]: unknown keyword 'X'` | Config syntax error | Check the generated config: `cat /etc/haproxy/haproxy.cfg` |
| `backend 'X' has no server` | Empty `servers` list resolved to nothing | Verify inventory group exists and `gather_facts: true` is set |

---

## Keepalived fails to start

**Symptom:** `systemctl status keepalived` shows `failed`.

```bash
journalctl -u keepalived -n 100 --no-pager
```

**Common causes:**

| Error | Cause | Fix |
|-------|-------|-----|
| `Cant find interface eth0` | Wrong interface name | Run `ip link` on target — update `keepalived_network_interface` |
| `VRRP_Instance ... Entering FAULT state` | HAProxy not running | Fix HAProxy first — Keepalived tracks it via `track_script` |
| `bogus VRRP packet received` | Duplicate `virtual_router_id` on network | Choose a unique ID (1–255) not used by other VRRP instances on the same subnet |

---

## VIP not assigned to master node

**Symptom:** Neither node has the VIP (`ip addr show` shows nothing on both).

```bash
# Check VRRP state on both nodes
journalctl -u keepalived | grep -E "MASTER|BACKUP|FAULT"

# Check if HAProxy is running (tracked by Keepalived)
systemctl status haproxy
```

**Common causes:**

- HAProxy is down on the master node — Keepalived will drop to FAULT state if the tracked service fails. Fix HAProxy first.
- `keepalived_unicast: true` but nodes can't reach each other on UDP port 112 — check firewall rules between proxy nodes.
- `virtual_router_id` collision — another system on the same subnet is using the same ID.

---

## VIP not failing over to backup

**Symptom:** Master node fails but VIP stays on master (or disappears entirely).

```bash
# Simulate failure — stop HAProxy on master
systemctl stop haproxy

# Watch Keepalived logs on backup node
journalctl -u keepalived -f
```

Expected: within ~3 seconds the backup should log `Entering MASTER state` and the VIP appears on the backup.

**Common causes:**

- Firewall blocking VRRP traffic between nodes — allow protocol 112 (VRRP) and/or UDP 112 between proxy hosts.
- With `keepalived_unicast: true`, check that `proxy_hosts` inventory group contains both nodes so the unicast peer list is correct.
- `haproxy_loadbalancer_master` / `haproxy_loadbalancer_backup` set to wrong hostnames — they must match `inventory_hostname` exactly.

---

## Backend servers marked as DOWN

**Symptom:** HAProxy stats page shows backend servers in red/DOWN state.

```bash
# Test health check endpoint manually
curl -v http://<backend-ip>:<port>/health

# Check HAProxy stats via socket
echo "show stat" | socat stdio /var/lib/haproxy/stats | cut -d',' -f1,2,18
```

**Common causes:**

- Health check endpoint (`httpcheck: true`) returns non-2xx — verify the backend app is running and the check URI is correct.
- Wrong port in backend definition — verify `port` matches what the backend app listens on.
- Firewall between proxy and backend nodes blocking the configured port.
- `mode: tcp` with `httpcheck: true` — HTTP checks require `mode: http`. Use `option tcp-check` for TCP mode instead.

---

## Stats page not accessible

**Symptom:** `curl http://<vip>:1936/haproxy/stats` returns connection refused or 401.

```bash
# Verify HAProxy is listening on the stats port
ss -tlnp | grep 1936

# Test with credentials
curl -u admin:<password> http://<host>:1936/haproxy/stats
```

**Common causes:**

- `haproxy_stats: false` — set to `true` and re-run.
- Wrong credentials — `haproxy_stats_page_user` / `haproxy_stats_page_pass` must match what you're using.
- `haproxy_stats_bind_addr: "127.0.0.1"` — stats only accessible from localhost. Change to `"0.0.0.0"` for remote access.
- Default password `"admin"` — the role's validation deliberately fails with the default password as a security measure. Set a real password.

---

## SSL certificate errors

**Symptom:** Browser shows certificate warning or `curl` returns SSL errors.

```bash
# Verify cert file exists and is readable
ls -la /etc/haproxy/certs/

# Test SSL manually
openssl s_client -connect <vip>:443 -servername <hostname>
```

**Common causes:**

- Certificate file path in `crts` list doesn't exist on the target host — copy the PEM file first or use the `files/certs/` directory.
- PEM file is missing the full chain — HAProxy requires the cert + intermediate + key in a single PEM file:
  ```bash
  cat cert.crt intermediate.crt > full-chain.pem
  cat full-chain.pem private.key > /etc/haproxy/certs/example.com.pem
  ```
- Frontend not set to `ssl: true` — verify the frontend definition includes `ssl: true` and the `crts` list.

---

## Cloud floating IP not reassigning

**Symptom:** With `cloud_floating_ip_enabled: true`, the floating IP stays on the old master after failover.

```bash
# Check notify script exists
ls -la /etc/keepalived/

# Run notify script manually to test API call
/etc/keepalived/cloud-floating-ip-notify.sh MASTER

# Check API token is valid
curl -H "Authorization: Bearer <token>" https://api.hetzner.cloud/v1/floating_ips
```

**Common causes:**

- `cloud_api_token` not set or expired — store in Ansible Vault and verify it works with a manual API call.
- `cloud_floating_ip_address` not matching the actual Hetzner floating IP — verify in the Hetzner Cloud console.
- Firewall on proxy nodes blocking outbound HTTPS to the Hetzner Cloud API endpoint.
- Wrong `cloud_api_endpoint` — the built-in script targets Hetzner Cloud (`https://api.hetzner.cloud/v1`) only.

---

## Useful commands reference

```bash
# --- HAProxy ---
systemctl status haproxy
systemctl reload haproxy                       # Reload config without dropping connections
haproxy -c -f /etc/haproxy/haproxy.cfg        # Validate config syntax
journalctl -u haproxy -f

# --- Keepalived ---
systemctl status keepalived
journalctl -u keepalived -f
journalctl -u keepalived | grep -E "MASTER|BACKUP|FAULT|transition"

# --- VIP status ---
ip addr show                                   # Check which node has the VIP
ip addr show eth0 | grep inet                  # Filter to specific interface

# --- Backend health via HAProxy socket ---
echo "show stat" | socat stdio /var/lib/haproxy/stats
echo "show info" | socat stdio /var/lib/haproxy/stats

# --- Stats page ---
curl -u admin:<pass> http://localhost:1936/haproxy/stats

# --- Config files ---
# HAProxy config:    /etc/haproxy/haproxy.cfg
# Keepalived config: /etc/keepalived/keepalived.conf
# SSL certs:         /etc/haproxy/certs/
```
