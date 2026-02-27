# Configuring Consul Storage Backend

> **Module 2 — Installing Vault Server | Topic 09**
> Consul: a battle-tested external storage backend with service discovery, health checks, and independent scaling.

---

## Why Use Consul as a Storage Backend?

While **Raft (Integrated Storage)** is now the recommended default, **Consul** remains a powerful option — especially for organizations already running Consul for service mesh or service discovery.

```
┌───────────────────────────────────────────────────────────────┐
│                     Consul Storage Analogy                      │
│                                                                 │
│  Raft = Your house has its own safe (built-in)                 │
│  Consul = You rent a safety deposit box at a bank (external)   │
│                                                                 │
│  The bank (Consul) is managed separately, has its own HA,      │
│  and your valuables are stored there encrypted.                 │
└───────────────────────────────────────────────────────────────┘
```

---

## Consul Advantages for Vault Storage

| Advantage | Description |
|-----------|-------------|
| **Durable KV Store** | Data persisted across Consul server cluster with Raft consensus |
| **Independent Scaling** | Scale Consul and Vault clusters separately |
| **Service Discovery** | Consul knows where Vault instances are automatically |
| **Health Checks** | Consul monitors Vault node health |
| **Multi-Datacenter** | Consul supports WAN federation across data centers |
| **Proven at Scale** | Battle-tested in large-scale enterprise deployments |

---

## Architecture: Vault + Consul

### Recommended Production Topology

```
┌──────────────────────────────────────────────────────────────┐
│                   AWS Region: us-east-1                        │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              Vault Cluster (3-5 nodes)               │     │
│  │                                                      │     │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐    │     │
│  │  │  Vault A    │  │  Vault B    │  │  Vault C    │    │     │
│  │  │ + Consul    │  │ + Consul    │  │ + Consul    │    │     │
│  │  │   Agent     │  │   Agent     │  │   Agent     │    │     │
│  │  │  (AZ-1a)    │  │  (AZ-1b)    │  │  (AZ-1c)    │    │     │
│  │  └─────┬──────┘  └──────┬─────┘  └──────┬─────┘    │     │
│  │        │                │                │           │     │
│  └────────┼────────────────┼────────────────┼───────────┘     │
│           │                │                │                  │
│           ▼                ▼                ▼                  │
│  ┌────────────────────────────────────────────────────┐      │
│  │          Consul Server Cluster (Dedicated)          │      │
│  │                                                     │      │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐   │      │
│  │  │  Consul A   │  │  Consul B   │  │  Consul C   │   │      │
│  │  │  (Server)   │  │  (Server)   │  │  (Server)   │   │      │
│  │  │  (AZ-1a)    │  │  (AZ-1b)    │  │  (AZ-1c)    │   │      │
│  │  └────────────┘  └────────────┘  └────────────┘   │      │
│  └────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

### Key Architecture Decisions

| Decision | Recommendation |
|----------|---------------|
| Consul cluster | **Dedicated** — don't share with other services |
| Consul agents on Vault nodes | **Yes** — local agent reduces network hops |
| Multi-AZ deployment | **Yes** — spread across availability zones |
| Consul servers | **3 or 5** — same odd-number rule as Vault/Raft |

---

## How Local Consul Agents Work

Each Vault node runs a **local Consul agent** (client mode) that talks to the Consul server cluster:

```
┌──────────────────────────────────┐
│          Vault Node              │
│                                  │
│  ┌────────────┐  ┌───────────┐  │
│  │   Vault     │──│  Consul    │ ─────► Consul Server Cluster
│  │  Process    │  │  Agent     │  │
│  │             │  │ (Client)   │  │
│  └────────────┘  └───────────┘  │
│                                  │
│  Vault writes to 127.0.0.1:8500 │
│  Agent forwards to server cluster│
└──────────────────────────────────┘
```

**Why a local agent?**
- Vault connects to `127.0.0.1:8500` (fast, no network hop)
- Agent handles server discovery, retry logic, and health reporting
- Agent caches and optimizes Consul queries

---

## Vault Configuration (vault.hcl)

```hcl
storage "consul" {
  address      = "127.0.0.1:8500"
  path         = "vault/"
  token        = "vault-acl-token-here"
  scheme       = "http"
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_disable     = 0
  tls_cert_file   = "/etc/vault.d/tls/vault-cert.pem"
  tls_key_file    = "/etc/vault.d/tls/vault-key.pem"
}

seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "arn:aws:kms:us-east-1:123456789:key/abc-123-..."
}

api_addr     = "http://10.0.1.10:8200"
cluster_addr = "http://10.0.1.10:8201"
ui           = true
log_level    = "INFO"
```

### Storage Stanza Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `address` | `127.0.0.1:8500` | Local Consul agent address |
| `path` | `vault/` | KV prefix in Consul where Vault stores data |
| `token` | ACL token | Consul ACL token with appropriate permissions |
| `scheme` | `http` or `https` | Protocol for Consul communication |
| `service` | `vault` | Service name registered in Consul (default: "vault") |
| `check_timeout` | `5s` | Timeout for health checks |

---

## Consul Server Configuration

### consul-server.json (on Consul server nodes)

```json
{
  "server": true,
  "bootstrap_expect": 3,
  "node_name": "consul-server-1",
  "datacenter": "us-east-1",
  "data_dir": "/opt/consul/data",
  "bind_addr": "10.0.2.10",
  "client_addr": "0.0.0.0",
  "retry_join": [
    "10.0.2.10",
    "10.0.2.20",
    "10.0.2.30"
  ],
  "ui_config": {
    "enabled": true
  },
  "acl": {
    "enabled": true,
    "default_policy": "deny",
    "enable_token_persistence": true
  },
  "performance": {
    "raft_multiplier": 1
  }
}
```

### Consul Agent Configuration (on Vault nodes)

```json
{
  "server": false,
  "node_name": "vault-node-1",
  "datacenter": "us-east-1",
  "data_dir": "/opt/consul/data",
  "bind_addr": "10.0.1.10",
  "retry_join": [
    "10.0.2.10",
    "10.0.2.20",
    "10.0.2.30"
  ],
  "acl": {
    "enabled": true,
    "tokens": {
      "agent": "consul-agent-acl-token"
    }
  }
}
```

---

## Consul ACL Token for Vault

Vault needs a Consul ACL token with specific permissions:

```hcl
# vault-consul-policy.hcl
key_prefix "vault/" {
  policy = "write"
}
node_prefix "" {
  policy = "read"
}
service "vault" {
  policy = "write"
}
agent_prefix "" {
  policy = "read"
}
session_prefix "" {
  policy = "write"
}
```

```bash
# Create the policy
consul acl policy create -name vault-policy -rules @vault-consul-policy.hcl

# Create the token
consul acl token create -description "Vault Storage Token" -policy-name vault-policy
```

> **Principle of least privilege:** Only grant what Vault needs — write to `vault/` KV path, write to its own service, and session management for leader election.

---

## Raft vs Consul: Side-by-Side

```
┌──────────────────────┬───────────────────┬───────────────────┐
│   Feature            │   Raft            │   Consul          │
├──────────────────────┼───────────────────┼───────────────────┤
│ External dependency  │ None              │ Consul cluster    │
│ Setup complexity     │ Simple            │ Complex (2 clusters)│
│ HA mechanism         │ Built-in Raft     │ Consul sessions   │
│ Service discovery    │ Cloud auto-join   │ Consul native     │
│ Health checks        │ Raft heartbeats   │ Consul checks     │
│ Scaling              │ Scale Vault only  │ Scale independently│
│ Backup               │ raft snapshot     │ consul snapshot   │
│ Multi-DC             │ Performance repl  │ Consul WAN        │
│ Recommendation       │ ✅ Preferred      │ Supported         │
│ Operational burden   │ Lower             │ Higher            │
└──────────────────────┴───────────────────┴───────────────────┘
```

---

## Quick Recap

```
Consul Storage Backend
│
├── Architecture:
│   ├── Vault nodes + local Consul agents
│   └── Dedicated Consul server cluster (3-5 nodes)
│
├── Vault Config (vault.hcl):
│   storage "consul" {
│     address = "127.0.0.1:8500"
│     path    = "vault/"
│     token   = "<acl-token>"
│   }
│
├── Consul Agent on Vault Nodes:
│   └── Client mode → forwards to Consul servers
│
├── Consul ACL:
│   ├── key_prefix "vault/" { write }
│   ├── service "vault" { write }
│   └── session_prefix "" { write }
│
├── Advantages:
│   ├── Independent scaling
│   ├── Service discovery & health checks
│   └── Multi-datacenter support
│
└── Disadvantages:
    ├── Two clusters to manage
    ├── Higher operational complexity
    └── Network dependency for storage
```

---

## Key Exam Tips

1. Vault connects to the **local Consul agent** (`127.0.0.1:8500`), not directly to Consul servers
2. Storage path in Consul KV is configured with the `path` parameter (default: `vault/`)
3. Vault requires a **Consul ACL token** scoped to its KV path, service, and sessions
4. **Dedicated Consul cluster** is recommended — don't share with other workloads
5. Consul handles **HA leader election** via sessions (different from Raft's built-in election)
6. Data stored in Consul is **encrypted by Vault** before writing — Consul never sees plaintext
7. Multi-AZ deployment protects against availability zone failures
8. **Raft is preferred** for new deployments; Consul is for orgs already running Consul
9. `bootstrap_expect` in Consul server config = number of servers needed to form initial cluster
10. Consul client agents on Vault nodes handle **service registration**, making Vault discoverable

---

## Official References

- [Vault Consul Storage Backend](https://developer.hashicorp.com/vault/docs/configuration/storage/consul)
- [Consul Architecture](https://developer.hashicorp.com/consul/docs/architecture)
- [Consul ACL System](https://developer.hashicorp.com/consul/docs/security/acl)
- [Vault + Consul Reference Architecture](https://developer.hashicorp.com/vault/tutorials/day-one-consul/reference-architecture)
- [KodeKloud: HashiCorp Vault](https://kodekloud.com)
