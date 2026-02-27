# Running Vault Server in Production

> **Module 2 — Installing Vault Server | Topic 07**
> Production deployment: persistent storage, HA, systemd, and real-world topologies.

---

## Production ≠ Dev Mode

If dev mode is a toy safe in your bedroom, **production mode** is a **hardened bank vault** — multi-layered security, redundancy, monitoring, and access control at every level.

```
┌───────────────────────────────────────────────────────────────┐
│                  Dev  ──→  Production                          │
│                                                                │
│  In-memory    ──→  Persistent storage (Raft / Consul)         │
│  Auto-unseal  ──→  Manual init + KMS auto-unseal              │
│  HTTP          ──→  TLS required                               │
│  Single node   ──→  HA cluster (3-5 nodes)                    │
│  Root token    ──→  Policies + auth methods                    │
│  No monitoring ──→  Audit logs + metrics + alerting            │
└───────────────────────────────────────────────────────────────┘
```

---

## 5 Key Considerations for Production

| # | Consideration | Why It Matters |
|---|--------------|----------------|
| 1 | **Persistent Nodes** | Vault data must survive restarts — use Raft or Consul storage |
| 2 | **HA-Enabled Backend** | Eliminates single point of failure — active/standby architecture |
| 3 | **Network Proximity** | Storage backend should be low-latency to Vault (same region/AZ) |
| 4 | **Automation** | Use Packer for images, Terraform for infra, Ansible for config |
| 5 | **Process Management** | systemd for lifecycle management, restart on failure, logging |

---

## systemd: Managing Vault as a Service

In production, Vault runs as a **systemd service** — not a foreground process.

### Why systemd?

| Feature | Benefit |
|---------|---------|
| Auto-start on boot | Vault comes up after a reboot |
| Restart on failure | Self-healing if Vault crashes |
| Security hardening | `ProtectSystem`, `PrivateTmp`, `CAP_IPC_LOCK` |
| Journal logging | `journalctl -u vault` for centralized logs |
| Dependency management | Starts only after network is ready |

### Essential Commands

```bash
sudo systemctl enable vault     # Auto-start on boot
sudo systemctl start vault      # Start now
sudo systemctl stop vault       # Graceful stop
sudo systemctl restart vault    # Stop + Start
sudo systemctl status vault     # Current state + recent logs

sudo journalctl -u vault -f     # Tail live logs
sudo journalctl -u vault --since "1 hour ago"  # Recent logs
```

---

## Production Deployment Topologies

### 1. Single Node (Not Recommended)

```
┌─────────────────────┐
│   Single Vault Node  │
│   + Raft Storage     │
│   (No HA)            │
└─────────────────────┘
```

- Good for: Dev/test environments, learning
- Bad for: Production (single point of failure)

---

### 2. Multi-Node with Integrated Storage (Raft) ⭐ Recommended

```
┌─────────────────────────────────────────────────────┐
│              Raft Cluster (3 or 5 Nodes)             │
│                                                      │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐       │
│   │  Node A   │   │  Node B   │   │  Node C   │       │
│   │ (Leader)  │◄─►│(Follower) │◄─►│(Follower) │       │
│   │  Active   │   │ Standby   │   │ Standby   │       │
│   │  :8200    │   │  :8200    │   │  :8200    │       │
│   │  :8201    │   │  :8201    │   │  :8201    │       │
│   └──────────┘   └──────────┘   └──────────┘       │
│         ▲               ▲               ▲            │
│         └───────────────┼───────────────┘            │
│              Raft Consensus (Port 8201)              │
└─────────────────────────────────────────────────────┘
```

- **Port 8200**: Client API requests
- **Port 8201**: Cluster communication (Raft consensus)
- Built-in storage — no external dependency
- Leader election is automatic
- Recommended: **5 nodes** for production (tolerates 2 failures)

---

### 3. Multi-Node with Consul Storage

```
┌─────────────────────────────────────────────────────┐
│              Vault Cluster (3-5 Nodes)               │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐       │
│   │ Vault A   │   │ Vault B   │   │ Vault C   │       │
│   │ + Consul  │   │ + Consul  │   │ + Consul  │       │
│   │   Agent   │   │   Agent   │   │   Agent   │       │
│   └────┬─────┘   └────┬─────┘   └────┬─────┘       │
│        │               │               │              │
│        ▼               ▼               ▼              │
│   ┌──────────────────────────────────────────┐       │
│   │        Consul Server Cluster (3-5)        │       │
│   │     (Dedicated, HA, Multi-AZ)             │       │
│   └──────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────┘
```

- Vault stores encrypted data in Consul's KV store
- Consul handles HA leader election
- Requires managing TWO clusters

---

## Topology Comparison

| Feature | Raft (Integrated) | Consul Backend |
|---------|-------------------|----------------|
| External dependency | None | Consul cluster |
| HA support | Built-in | Via Consul |
| Operational complexity | Lower | Higher (2 clusters) |
| Network latency | Local disk | Network hop to Consul |
| Scaling | Scale Vault only | Scale both independently |
| Backup | `vault operator raft snapshot` | Consul snapshot + Vault |
| HashiCorp recommendation | ✅ Preferred | Supported |

---

## 8 Steps to Deploy Production Vault

```
Step 1 ──→ Download Vault binary
Step 2 ──→ Create vault system user
Step 3 ──→ Create directories (/etc/vault.d, storage path)
Step 4 ──→ Write vault.hcl configuration
Step 5 ──→ Write systemd unit file
Step 6 ──→ Enable + start the service
Step 7 ──→ Initialize Vault (vault operator init)
Step 8 ──→ Configure auth methods, policies, secrets engines
```

### Quick Command Reference

```bash
# Step 1: Install
sudo unzip vault.zip && sudo mv vault /usr/local/bin/

# Step 2: System user
sudo useradd --system --home /var/lib/vault --shell /sbin/nologin vault

# Step 3: Directories
sudo mkdir -p /etc/vault.d /opt/vault/data
sudo chown -R vault:vault /etc/vault.d /opt/vault

# Step 4: Config (see Topic 08/09 for storage details)
sudo vi /etc/vault.d/vault.hcl

# Step 5: systemd unit
sudo vi /etc/systemd/system/vault.service

# Step 6: Start
sudo systemctl daemon-reload
sudo systemctl enable vault
sudo systemctl start vault

# Step 7: Initialize
export VAULT_ADDR='https://<vault-ip>:8200'
vault operator init

# Step 8: Configure
vault auth enable userpass
vault policy write admin admin.hcl
vault secrets enable -path=kv kv-v2
```

---

## Quick Recap

```
Production Vault
│
├── Storage: Raft (recommended) or Consul
├── HA: 3-5 node cluster
├── Process: systemd (auto-restart, security, logging)
├── TLS: Always enabled
├── Unseal: Auto-unseal (KMS) or Shamir keys
│
├── Topologies:
│   ├── Single node    → Dev/test only
│   ├── Multi-node Raft → ✅ Recommended
│   └── Multi-node Consul → Supported, more complex
│
└── Key Commands:
    systemctl start/stop/status vault
    journalctl -u vault -f
    vault operator init
    vault status
```

---

## Key Exam Tips

1. Production Vault **always** needs a persistent storage backend (Raft or Consul)
2. **Integrated Storage (Raft)** is HashiCorp's recommended backend — no external dependencies
3. systemd provides auto-restart, security hardening, and boot-time startup
4. Port **8200** = API/client, Port **8201** = cluster/Raft communication
5. A **5-node cluster** tolerates 2 node failures (quorum = 3)
6. A **3-node cluster** tolerates 1 node failure (quorum = 2)
7. Vault should run as a **dedicated system user**, never root
8. **CAP_IPC_LOCK** capability lets Vault lock memory to prevent secrets from being swapped to disk

---

## Official References

- [Vault Production Hardening](https://developer.hashicorp.com/vault/tutorials/operations/production-hardening)
- [Vault Reference Architecture](https://developer.hashicorp.com/vault/tutorials/day-one-raft/raft-reference-architecture)
- [systemd Integration](https://developer.hashicorp.com/vault/tutorials/operations/production-hardening#systemd)
- [KodeKloud: HashiCorp Vault](https://kodekloud.com)
