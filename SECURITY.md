# Security Policy

## Supported Versions

We release patches for security vulnerabilities in the following versions:

| Version | Supported          |
| ------- | ------------------ |
| 1.x.x   | :white_check_mark: |
| < 1.0   | :x:                |

## Reporting a Vulnerability

We take the security of our Ansible role seriously. If you discover a security vulnerability, please follow these steps:

### 1. **Do Not** Open a Public Issue

Please do not report security vulnerabilities through public GitHub issues, discussions, or pull requests.

### 2. Report Privately

Instead, please report security vulnerabilities by emailing:

**Email:** security@devopsgroup.sk

Please include the following information in your report:

- Type of vulnerability (e.g., privilege escalation, information disclosure, code injection)
- Full paths of source file(s) related to the vulnerability
- Location of the affected source code (tag/branch/commit or direct URL)
- Step-by-step instructions to reproduce the issue
- Proof-of-concept or exploit code (if possible)
- Impact of the vulnerability, including how an attacker might exploit it

### 3. Response Timeline

- We will acknowledge receipt of your vulnerability report within **3 business days**
- We will send you regular updates about our progress
- We aim to release a security patch within **30 days** for critical vulnerabilities
- Once the vulnerability is fixed, we will publicly disclose it (with your consent)

### 4. Coordinated Disclosure

We request that you:

- Give us reasonable time to fix the vulnerability before public disclosure
- Make a good faith effort to avoid privacy violations, data destruction, and service interruption
- Do not exploit the vulnerability beyond what is necessary to demonstrate it

## Security Best Practices

When using this Ansible role, we recommend:

### HAProxy Security

1. **Change Default Credentials**
   ```yaml
   haproxy_stats_page_user: "your_username"
   haproxy_stats_page_pass: "strong_password_here"
   ```
   Never use the default `admin/admin` credentials in production.

2. **Restrict Stats Page Access**
   ```yaml
   haproxy_stats_bind_addr: "127.0.0.1"  # localhost only
   # Or use specific IP
   haproxy_stats_bind_addr: "10.0.1.10"
   ```

3. **Use SSL/TLS for Frontends**
   ```yaml
   haproxy_frontends:
     - name: secure_frontend
       ssl: true
       crts:
         - /etc/haproxy/certs/certificate.pem
   ```

4. **Keep HAProxy Updated**
   - Regularly update the `haproxy_version` variable to the latest stable release
   - Monitor [HAProxy Security Advisories](https://www.haproxy.org/)

### Keepalived Security

1. **Secure Network Interface**
   - Ensure the VRRP interface is on a trusted network
   - Use firewall rules to restrict VRRP protocol (IP protocol 112) to trusted nodes

2. **Authentication (Optional Enhancement)**
   - Consider enabling VRRP authentication in production environments (VRRPv2 only)
   - Set `keepalived_auth_type` (`PASS` or `AH`) and store `keepalived_auth_pass` in Ansible Vault:
     ```yaml
     keepalived_auth_type: "PASS"
     keepalived_auth_pass: "{{ vault_keepalived_auth_pass }}"
     ```
   - Note: VRRPv2 `PASS` is cleartext in the VRRP header (max 8 chars) — pair it with network segmentation

### Infrastructure Security

1. **Air-Gapped Installations**
   - The role supports air-gapped environments
   - Pre-download packages to a secure location
   - Verify checksums of downloaded packages

2. **Principle of Least Privilege**
   - Run HAProxy and Keepalived with dedicated system users
   - The role creates `haproxy` system user with minimal permissions

3. **Log Monitoring**
   - Monitor HAProxy logs: `/var/log/haproxy-*.log`
   - Monitor Keepalived logs: `/var/log/keepalived.log`
   - Set up alerts for suspicious activities

4. **File Permissions**
   - Configuration files are set to `0644` (readable by all, writable by root)
   - Certificate files are set to `0640` (readable by haproxy group only)
   - Socket directory is set to `0750` (accessible by haproxy group only)

## Known Security Considerations

### Default Stats Page

The default configuration enables HAProxy statistics page with default credentials. This is intended for development/testing only. **Always change credentials before production deployment.**

### Virtual IP Management

Keepalived manages Virtual IPs (VIPs) using VRRP protocol, which by default does not use authentication. In production environments with untrusted networks, consider:

- Network segmentation (VLAN isolation)
- Firewall rules restricting VRRP traffic
- Custom Keepalived configuration with authentication

### Installation Method

This role installs HAProxy and Keepalived from distribution/repo **packages by
default** (`haproxy_install_method: package`, `keepalived_install_method: package`).
Source compilation is opt-in (`*_install_method: source`) for cases that need the
latest versions, custom build flags, or air-gapped builds:

- **Package mode (default):** patched by the distribution's security updates
- **Source mode (opt-in):** build-time dependencies and pinned versions must be kept updated

We recommend:
- In package mode, keeping the OS and HAProxy/Keepalived packages patched
- In source mode, regularly updating `haproxy_version` / `keepalived_version`
- Monitoring upstream security advisories
- Testing updates in non-production environments first

## Security Updates

We will announce security updates through:

1. GitHub Security Advisories
2. Git tags with security notes
3. CHANGELOG.md with `[SECURITY]` prefix

Subscribe to repository notifications to stay informed.

## Hall of Fame

We appreciate security researchers who responsibly disclose vulnerabilities. Contributors will be acknowledged here (with permission):

_No security vulnerabilities have been reported yet._

## Questions?

If you have questions about this security policy, please contact:

**Email:** security@devopsgroup.sk

---

**Last Updated:** 2026-02-01
