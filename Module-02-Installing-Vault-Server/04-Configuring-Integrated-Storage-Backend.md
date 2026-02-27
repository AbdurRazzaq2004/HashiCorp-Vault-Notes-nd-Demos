# Configuring Integrated Storage Backend (Raft)

> **Module 2 — Installing Vault Server | Topic 08**
> Raft: Vault's built-in storage — no external dependencies, built-in HA, and simple operations.

---

## What Is Integrated Storage?

**Integrated Storage** uses the **Raft consensus protocol** to provide a built-in, distributed storage backend directly inside Vault — no Consul, no etcd, no external database.

```
┌───────────────────────────────────────────────────────────────┐
│                   Integrated Storage Analogy                   │
│                                                                │
│  Imagine 5 librarians (nodes) who each keep a copy of the     │
│  catalog. When a book is added, they vote:                     │
│                                                                │
│  "Do we all agree this book exists?"                           │
│                                                                │
│  If the majority (3 of 5) agree → the entry is committed.     │
│  If 2 librarians are sick → the library still works.           │
│  That's Raft consensus.                                        │
└───────────────────────────────────────────────────────────────┘
```

---

## Why Raft? Key Advantages

| Advantage | Description |
|-----------|-------------|
| **No external dependency** | Storage is embedded in Vault itself |
| **Built-in HA** | Leader election + automatic failover |
| **Data replication** | All nodes have a full copy of the data |
| **Simpler operations** | One cluster to manage, not two |
| **Integrated snapshots** | `vault operator raft snapshot save` |
| **HashiCorp recommended** | The preferred backend for new deployments |

---

## Raft Cluster Topology

### 5-Node Production Cluster

```
                    ┌─────────────┐
                    │   Node A     │
                    │  (Leader)    │
                    │  :8200 API   │
                    │  :8201 Raft  │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼─────┐ ┌───▼───────┐ ┌─▼───────────┐
        │  Node B    │ │  Node C    │ │  Node D      │
        │ (Follower) │ │ (Follower) │ │ (Follower)   │
        │  :8200     │ │  :8200     │ │  :8200       │
        │  :8201     │ │  :8201     │ │  :8201       │
        └────────────┘ └───────────┘ └──────────────┘
                                            │
                                      ┌─────▼──────┐
                                      │  Node E     │
                                      │ (Follower)  │
                                      │  :8200      │
                                      │  :8201      │
                                      └─────────────┘
```

| Port | Purpose |
|------|---------|
| **8200** | Client API requests (read/write secrets) |
| **8201** | Cluster communication (Raft consensus, log replication) |

---

## Quorum and Fault Tolerance

| Cluster Size | Quorum Needed | Failures Tolerated |
|:------------:|:-------------:|:-----------------:|
| 3 nodes | 2 | 1 |
| 5 nodes | 3 | 2 |
| 7 nodes | 4 | 3 |

> **Rule of thumb:** Use **odd numbers** (3, 5, 7). Even numbers waste a node without improving fault tolerance.

---

## Configuration: vault.hcl

### Node A (First Node / Bootstrap Leader)

```hcl
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "node-a-us-east-1a"

  retry_join {
    leader_api_addr = "http://10.0.1.10:8200"
  }
  retry_join {
    leader_api_addr = "http://10.0.1.20:8200"
  }
  retry_join {
    leader_api_addr = "http://10.0.1.30:8200"
  }
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
cluster_name = "vault-prod-us-east-1"
ui           = true
log_level    = "INFO"
```

### Configuration Breakdown

| Stanza / Parameter | Purpose |
|-------------------|---------|
| `storage "raft"` | Use integrated storage |
| `path` | Local directory for Raft data files |
| `node_id` | Unique identifier for this node in the cluster |
| `retry_join` | Addresses of other nodes to join on startup |
| `leader_api_addr` | API address of a potential leader |
| `listener "tcp"` | Network listener for client API |
| `cluster_address` | Address for node-to-node Raft traffic |
| `seal "awskms"` | Auto-unseal using AWS KMS |
| `api_addr` | This node's advertised API address |
| `cluster_addr` | This node's advertised cluster address |

---

## Auto-Join with Cloud Provider Tags

Instead of hardcoding IPs, use **auto-join** to discover nodes via cloud provider tags:

```hcl
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "node-a-us-east-1a"

  retry_join {
    auto_join             = "provider=aws region=us-east-1 tag_key=vault tag_value=us-east-1"
    auto_join_scheme      = "http"
    auto_join_port        = 8200
  }
}
```

| Provider | Auto-Join Config Example |
|----------|------------------------|
| AWS | `provider=aws region=us-east-1 tag_key=vault tag_value=prod` |
| Azure | `provider=azure tag_name=vault tag_value=prod` |
| GCP | `provider=gce tag_value=vault-server` |

> **Auto-join** queries the cloud API for instances with matching tags — no need to know IPs in advance.

---

## Cluster Operations

### Initialize the First Node

```bash
export VAULT_ADDR='http://10.0.1.10:8200'
vault operator init
```

### Join Additional Nodes

If not using `retry_join`, manually join nodes:

```bash
# On Node B
export VAULT_ADDR='http://10.0.1.20:8200'
vault operator raft join http://10.0.1.10:8200
```

With `retry_join` configured, nodes join automatically on startup — no manual step needed.

### List Raft Peers

```bash
vault operator raft list-peers
```

```
Node                   Address            State       Voter
----                   -------            -----       -----
node-a-us-east-1a     10.0.1.10:8201     leader      true
node-b-us-east-1b     10.0.1.20:8201     follower    true
node-c-us-east-1c     10.0.1.30:8201     follower    true
```

### Remove a Peer

```bash
vault operator raft remove-peer node-c-us-east-1c
```

---

## Snapshots (Backup & Restore)

```bash
# Save a snapshot
vault operator raft snapshot save backup-2024-01-15.snap

# Restore from a snapshot
vault operator raft snapshot restore backup-2024-01-15.snap
```

> **Best practice:** Schedule automated snapshots and store them in S3 / GCS / Azure Blob.

---

## Data Flow: Write Operation

```
Client writes secret
       │
       ▼
┌──────────────┐
│  Leader Node  │ ── Appends to Raft log
│  (Node A)     │
└──────┬───────┘
       │  Replicates via port 8201
       ├──────────────────────┐
       ▼                      ▼
┌──────────────┐     ┌──────────────┐
│  Follower B   │     │  Follower C   │
│  Writes log   │     │  Writes log   │
└──────────────┘     └──────────────┘
       │                      │
       ▼                      ▼
    ACK to Leader          ACK to Leader
       │
       ▼
  Quorum reached (2 of 3) → Commit confirmed
       │
       ▼
  Client gets success response
```

---

## Quick Recap

```
Integrated Storage (Raft)
│
├── What: Built-in distributed storage inside Vault
├── Port 8200: API (client traffic)
├── Port 8201: Cluster (Raft replication)
│
├── Config (vault.hcl):
│   storage "raft" {
│     path    = "/opt/vault/data"
│     node_id = "unique-name"
│     retry_join { leader_api_addr = "http://..." }
│   }
│
├── Discovery:
│   ├── retry_join with IPs (static)
│   └── auto_join with cloud tags (dynamic)
│
├── Operations:
│   ├── vault operator raft join <addr>
│   ├── vault operator raft list-peers
│   ├── vault operator raft remove-peer <id>
│   └── vault operator raft snapshot save/restore
│
└── Sizing:
    ├── 3 nodes → tolerates 1 failure
    └── 5 nodes → tolerates 2 failures (recommended)
```

---

## Key Exam Tips

1. **Integrated Storage = Raft** — they are the same thing
2. Port **8200** for API, Port **8201** for cluster/Raft communication
3. `retry_join` enables automatic cluster formation on startup
4. `auto_join` uses cloud provider APIs (AWS tags, Azure tags, etc.) for discovery
5. **Quorum** = majority of nodes must agree for writes to commit
6. 5-node cluster needs **3 nodes** for quorum (tolerates 2 failures)
7. `vault operator raft list-peers` shows cluster membership and leader
8. Snapshots are the **backup mechanism** for Raft storage
9. Each node needs a **unique `node_id`**
10. Raft is HashiCorp's **recommended** storage backend for new deployments

---

## Official References

- [Vault Integrated Storage](https://developer.hashicorp.com/vault/docs/configuration/storage/raft)
- [Raft Reference Architecture](https://developer.hashicorp.com/vault/tutorials/day-one-raft/raft-reference-architecture)
- [Cloud Auto-Join](https://developer.hashicorp.com/vault/docs/configuration/storage/raft#retry_join-stanza)
- [KodeKloud: HashiCorp Vault](https://kodekloud.com)
