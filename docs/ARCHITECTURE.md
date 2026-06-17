# Architecture

Visual reference for the HAProxy + Keepalived role — how components are laid out, how traffic flows, how failover works, and how the Ansible role executes.

---

## Table of Contents

- [HA Topology](#ha-topology)
- [Request Flow](#request-flow)
- [VRRP Failover Sequence](#vrrp-failover-sequence)
- [Cloud Floating IP Failover](#cloud-floating-ip-failover)
- [Role Task Execution Order](#role-task-execution-order)

---

## HA Topology

Active-passive setup: both nodes run HAProxy; Keepalived assigns the VIP to the master. If the master fails, the VIP moves to the backup within 2–3 seconds.

```mermaid
graph TD
    Client(["Client"])
    VIP["Virtual IP (VIP)\nfloating address"]

    subgraph proxy-01 ["proxy-01 — MASTER (priority 100)"]
        K1["Keepalived\nVRRP state: MASTER"]
        H1["HAProxy\n:80 / :443 / :6443"]
    end

    subgraph proxy-02 ["proxy-02 — BACKUP (priority 90)"]
        K2["Keepalived\nVRRP state: BACKUP"]
        H2["HAProxy\n:80 / :443 / :6443"]
    end

    subgraph backends ["Backend Pool"]
        B1["app-01"]
        B2["app-02"]
        B3["app-03"]
    end

    Client -->|"requests"| VIP
    VIP -->|"active"| H1
    VIP -.->|"standby (takes over on failure)"| H2
    H1 -->|"load balanced"| B1
    H1 --> B2
    H1 --> B3
    H2 -.-> B1
    H2 -.-> B2
    H2 -.-> B3

    K1 <-->|"VRRP advertisements\n(unicast)"| K2
```

---

## Request Flow

Path of a single HTTP request from client to backend.

```mermaid
flowchart LR
    C(["Client"])

    subgraph lb ["Load Balancer (active node)"]
        FE["HAProxy Frontend\nbind *:80"]
        BE["HAProxy Backend\nbalance: roundrobin"]
        HC["Health Checks\nHTTP GET /health"]
    end

    subgraph pool ["Backend Pool"]
        S1["server-01\n:8080 ✓"]
        S2["server-02\n:8080 ✓"]
        S3["server-03\n:8080 ✗ (down)"]
    end

    C -->|"HTTP :80"| FE
    FE -->|"route to backend"| BE
    BE -->|"check"| HC
    HC -.->|"healthy"| S1
    HC -.->|"healthy"| S2
    HC -.->|"unhealthy — skipped"| S3
    BE -->|"forward"| S1
    BE -->|"forward"| S2
```

---

## VRRP Failover Sequence

What happens when the master node's HAProxy stops responding.

```mermaid
sequenceDiagram
    participant Client
    participant VIP as Virtual IP
    participant Master as proxy-01 (MASTER)
    participant Backup as proxy-02 (BACKUP)

    Note over Master,Backup: Normal operation — proxy-01 holds VIP

    Client->>VIP: HTTP request
    VIP->>Master: forwarded
    Master-->>Client: response

    Note over Master: HAProxy health check fails
    Master->>Master: Keepalived detects failure
    Master->>Backup: VRRP: no more advertisements

    Note over Backup: No VRRP seen — election starts
    Backup->>Backup: Priority 90 wins by default
    Backup->>VIP: Gratuitous ARP — take over VIP

    Note over VIP: VIP now owned by proxy-02

    Client->>VIP: HTTP request
    VIP->>Backup: forwarded
    Backup-->>Client: response

    Note over Master: proxy-01 recovers
    Master->>Master: HAProxy restarts
    Master->>Backup: VRRP advertisements resume (priority 100)
    Backup->>VIP: Release VIP (preempt)
    Master->>VIP: Reclaim VIP
```

---

## Cloud Floating IP Failover

When `cloud_floating_ip_enabled: true`, a notify script calls the cloud provider API on every VRRP state transition to move the public Floating IP to the new master.

```mermaid
sequenceDiagram
    participant Keepalived as Keepalived (new MASTER)
    participant Script as /usr/local/bin/cloud-floating-ip-notify.sh
    participant API as Cloud Provider API
    participant FIP as Floating IP

    Note over Keepalived: VRRP state → MASTER

    Keepalived->>Script: notify_master trigger
    Script->>API: GET /servers?name={hostname}
    API-->>Script: server_id
    Script->>API: POST /floating_ips/{id}/actions/assign\n{ server: server_id }
    API-->>Script: 201 OK
    Script->>FIP: Floating IP re-assigned

    Note over FIP: External traffic now routes to new master

    Note over Keepalived: VRRP state → BACKUP (on old master)
    Keepalived->>Script: notify_backup trigger
    Script->>Script: No API call needed\n(new master already claimed FIP)
```

---

## Role Task Execution Order

Tasks that run when the role is applied to a proxy node.

```mermaid
flowchart TD
    Start(["`**ansible-role-haproxy-keepalived**
    applied to proxy node`"])

    Validate["**validate.yml**
    Check required vars
    Reject default credentials
    Confirm VIP is set"]

    Setup["**setup.yml**
    Install OS packages
    Install build dependencies
    Compile HAProxy from source
    Compile Keepalived from source"]

    HAProxy["**haproxy.yml**
    Deploy haproxy.cfg
    Create haproxy user/group
    Enable + start haproxy.service"]

    Keepalived["**keepalived.yml**
    Deploy keepalived.conf
    Assign MASTER / BACKUP state
    Enable + start keepalived.service"]

    Cloud{"cloud_floating_ip\n_enabled?"}

    CloudTask["**cloud_floating_ip.yml**
    Deploy notify script
    Configure VRRP notify hooks
    Reload keepalived"]

    Done(["Done — HA load balancer active"])

    Start --> Validate
    Validate --> Setup
    Setup --> HAProxy
    HAProxy --> Keepalived
    Keepalived --> Cloud
    Cloud -->|yes| CloudTask
    Cloud -->|no| Done
    CloudTask --> Done
```
