# Storage Backends

> **Module 1 — Learning the Vault Architecture**
> Topic: Where and how Vault persists ALL of its data.

---

## Overview

The storage backend is Vault's **hard drive** — it's where every piece of encrypted data lives: secrets, policies, configurations, tokens, leases, everything. Vault itself doesn't care *which* backend you pick — it encrypts everything before writing and decrypts after reading. The backend is just a dumb data store.

Think of it like choosing a **warehouse** for a business:

- **Consul** = A premium warehouse with 24/7 security and climate control (mature, proven HA)
- **Integrated Storage (Raft)** = Your own built-in storage room (no external dependency, HA built-in)
- **S3** = A cheap self-storage unit (simple, scalable, but no HA)
- **In-Memory** = Your pocket (fast, but everything vanishes when you leave)

```
┌──────────────────────────────────────────────────────────┐
│                      VAULT SERVER                         │
│                                                           │
│  Secrets ──┐                                              │
│  Policies ─┤── encrypt ──▶ Storage Backend ──▶ Disk/Cloud │
│  Tokens ───┤                                              │
│  Leases ───┘     All data is encrypted BEFORE             │
│                  being written to the backend              │
└──────────────────────────────────────────────────────────┘
```

> **Key point:** Vault encrypts all data before it hits the storage backend. Even if someone gains direct access to Consul or S3, they see only ciphertext.

---

## 1. Supported Storage Backends

### Full Comparison Table

| Backend | Edition | HA Support | Best For |
|---------|---------|-----------|----------|
| **Consul** | Open Source & Enterprise | ✅ Yes | Production HA clusters with mature ecosystem |
| **Integrated Storage (Raft)** | Open Source & Enterprise | ✅ Yes | Production HA without external dependencies |
| **S3** | Open Source | ❌ No | Simple scalable object storage |
| **DynamoDB** | Open Source | ⚠️ Single-region | Key/value store on AWS |
| **Azure Blob Storage** | Open Source | ❌ No | Blob-based storage on Microsoft Azure |
| **Google Cloud Storage** | Open Source | ❌ No | Object storage on GCP |
| **etcd, MySQL, PostgreSQL** | Community | Varies | Custom database deployments |
| **In-Memory** | Open Source | ❌ No | Dev mode / testing only |
| **Filesystem** | Open Source | ❌ No | Single-node non-production |

### Enterprise-Supported Backends

Only **two backends** are officially supported for HashiCorp Enterprise:

```
Enterprise Support
┌─────────────────────────────────┐
│                                  │
│  1. Consul    ← The veteran     │
│  2. Raft      ← The new default │
│                                  │
│  Everything else = Community     │
│  (use at your own risk in prod)  │
└─────────────────────────────────┘
```

---

## 2. Storage Backend Architecture

### Single Cluster — One Backend

Every Vault cluster has **exactly one** storage backend. All nodes in the cluster (active + standbys) share it:

```
                    Vault Cluster
    ┌──────────────────────────────────────┐
    │                                       │
    │  ┌────────┐  ┌────────┐  ┌────────┐ │
    │  │ Active │  │Standby │  │Standby │ │
    │  │  Node  │  │ Node 1 │  │ Node 2 │ │
    │  └───┬────┘  └───┬────┘  └───┬────┘ │
    │      │           │           │       │
    └──────┼───────────┼───────────┼───────┘
           │           │           │
           ▼           ▼           ▼
    ┌──────────────────────────────────────┐
    │       ONE Storage Backend             │
    │       (e.g., Consul or Raft)         │
    └──────────────────────────────────────┘
```

### Multi-Cluster — Separate Backends

If you have multiple clusters (e.g., for geographic redundancy), **each cluster gets its own backend**. Replication happens between Vault nodes (Enterprise feature), NOT between storage backends directly:

```
  US-East Cluster                    EU-West Cluster
  ┌─────────────┐                   ┌─────────────┐
  │ Vault Nodes │ ◄──Replication──▶ │ Vault Nodes │
  │      │      │    (Vault API)    │      │      │
  └──────┼──────┘                   └──────┼──────┘
         │                                  │
         ▼                                  ▼
  ┌─────────────┐                   ┌─────────────┐
  │ Consul (US) │     NO direct     │ Consul (EU) │
  │             │ ◄──connection──▶  │             │
  └─────────────┘                   └─────────────┘
```

---

## 3. Consul vs Integrated Storage (Raft)

These are the two big players. Here's how they compare:

| Feature | Consul | Integrated Storage (Raft) |
|---------|--------|--------------------------|
| **External dependency** | Yes (need Consul cluster) | No (built into Vault) |
| **HA support** | ✅ Yes | ✅ Yes |
| **Enterprise support** | ✅ Yes | ✅ Yes (since Vault 1.4) |
| **Feature parity** | Full | Full (since Vault 1.7) |
| **Cloud auto-join** | Via Consul | Built-in (`retry_join`) |
| **Automated backups** | Via Consul snapshots | Built-in (`vault operator raft snapshot`) |
| **Autopilot** | Via Consul | Built-in |
| **Operational complexity** | Higher (manage 2 clusters) | Lower (just Vault) |
| **Service mesh / DNS** | Yes (bonus features) | No |
| **Data location** | Consul's KV store | Local filesystem on each node |

### When to Choose Which

```
Need Consul for other things (service mesh, DNS)?
  └─ YES → Use Consul storage
  └─ NO  → ┐
            │
            ├─ Want simplest ops? → Use Raft (Integrated Storage)
            ├─ New deployment?    → Use Raft (recommended default)
            └─ Existing Consul?   → Either works, Consul is fine
```

> **Trend:** Integrated Storage (Raft) is now the recommended default for new deployments since reaching feature parity in Vault 1.7.

---

## 4. Choosing a Storage Backend — Decision Flowchart

```
                        START
                          │
                          ▼
                  ┌───────────────┐
                  │ Production    │
                  │ environment?  │
                  └───────┬───────┘
                    │           │
                   YES          NO
                    │           │
                    ▼           ▼
            ┌──────────┐   ┌──────────────┐
            │ Need HA? │   │ Testing/CI?  │
            └────┬─────┘   └──────┬───────┘
              │       │        │        │
             YES      NO      YES       NO
              │       │        │        │
              ▼       ▼        ▼        ▼
         ┌─────────┐ ┌────┐ ┌────────┐ ┌──────────┐
         │Consul or│ │S3  │ │In-Mem  │ │Filesystem│
         │  Raft   │ │DDB │ │(dev)   │ │          │
         └─────────┘ │GCS │ └────────┘ └──────────┘
                      │Blob│
                      └────┘

  HA Required    → Consul or Raft
  No HA (prod)   → S3, DynamoDB, GCS, Azure Blob
  Testing/CI     → In-Memory (dev mode)
  Single Node    → Filesystem
```

---

## 5. Configuring Your Storage Backend

The storage backend is defined in the `storage` stanza of your Vault config file.

### Example: Consul

```hcl
storage "consul" {
  address = "127.0.0.1:8500"                          # Consul agent endpoint
  path    = "vault/"                                   # KV namespace for Vault data
  token   = "1a2b3c4d-1234-abcd-1234-1a2b3c4d5e6a"   # Consul ACL token
}
```

| Parameter | Purpose |
|-----------|---------|
| `address` | Host and port of the local Consul agent |
| `path` | Prefix within Consul's key/value store (keeps Vault data organized) |
| `token` | ACL token granting Vault read/write access to Consul |

### Example: Integrated Storage (Raft)

```hcl
storage "raft" {
  path    = "/opt/vault/data"                          # Local dir for Raft logs + snapshots
  node_id = "node-a-us-east-1.example.com"             # Unique node identifier

  retry_join {
    auto_join = "provider=aws region=us-east-1 tag_key=vault tag_value=us-east-1"
  }
}
```

| Parameter | Purpose |
|-----------|---------|
| `path` | Filesystem path where this node stores replicated data |
| `node_id` | Unique identifier for this node in the Raft cluster |
| `retry_join.auto_join` | Auto-discovers peers using cloud provider tags (AWS, Azure, GCP) |

### Example: S3 (Simple, No HA)

```hcl
storage "s3" {
  bucket     = "my-vault-data"
  region     = "us-east-1"
  access_key = "AKIA..."
  secret_key = "wJalr..."
}
```

### Example: In-Memory (Dev Mode Only)

```hcl
storage "inmem" {}
```

> No parameters needed — everything lives in RAM and vanishes on restart.

---

## 6. Security Warning

```
⚠️  NEVER commit config files with plain-text tokens or credentials!

❌ BAD:
   storage "consul" {
     token = "1a2b3c4d-real-token-here"     ← Exposed in Git!
   }

✅ GOOD: Use environment variables
   storage "consul" {
     token = env("CONSUL_TOKEN")            ← Reads from env at runtime
   }

✅ ALSO GOOD: Use a separate secrets manager or inject at deploy time
```

---

## Quick Recap

```
Storage Backend = WHERE Vault persists encrypted data
│
├── Enterprise-Supported (Production HA):
│   ├── Consul       ← External dependency, mature, proven
│   └── Raft         ← Built-in, simpler ops, recommended default
│
├── Cloud (No HA):
│   ├── S3, DynamoDB, Azure Blob, GCS
│   └── Good for single-node or non-critical workloads
│
├── Community:
│   └── etcd, MySQL, PostgreSQL
│
└── Dev/Test:
    ├── In-Memory (data lost on restart)
    └── Filesystem (single-node only)

Architecture Rules:
• ONE storage backend per cluster
• All nodes share the same backend
• Replication = Vault-to-Vault (not backend-to-backend)
• All data encrypted BEFORE reaching the backend
```

---

## Key Exam Tips

1. **One storage backend per cluster** — all nodes (active + standbys) share it.
2. **Enterprise supports only Consul and Raft** — everything else is community/open source only.
3. **Raft (Integrated Storage)** reached feature parity with Consul in **Vault 1.7** — includes auto-join, automated backups, autopilot.
4. **Raft = no external dependency** — data stored locally on each node, replicated via Raft consensus.
5. **Consul requires a separate Consul cluster** — more operational overhead but brings service mesh/DNS.
6. **Storage backend stores ENCRYPTED data** — even with direct backend access, data is unreadable without Vault's master key.
7. **Multi-cluster replication** happens between Vault nodes (Enterprise), NOT directly between storage backends.
8. **In-memory storage** = dev mode only. Data is lost on every restart.
9. **`storage` stanza is REQUIRED** in the Vault config file — you must pick a backend.
10. **Never put secrets (tokens, keys) in config files** committed to version control — use environment variables.

---

## Official References

- [Vault Storage Backends Documentation](https://www.vaultproject.io/docs/configuration/storage)
- [Consul Storage Backend](https://www.vaultproject.io/docs/configuration/storage/consul)
- [Integrated Storage (Raft)](https://www.vaultproject.io/docs/configuration/storage/raft)
- [Vault Enterprise Features](https://www.vaultproject.io/docs/enterprise)
