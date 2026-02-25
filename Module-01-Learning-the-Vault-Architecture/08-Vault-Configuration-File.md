# Vault Configuration File

> **Module 1 — Learning the Vault Architecture**
> Topic: How to configure Vault for reliable, production-grade operation.

---

## Overview

The Vault configuration file is the **blueprint** for your Vault server — it tells Vault where to store data, how to listen for connections, how to unseal, and what to monitor. Think of it like a **restaurant floor plan** before opening day:

- **Storage** = the kitchen (where everything is kept)
- **Listener** = the front door (how customers get in)
- **Seal** = the safe in the back office (how valuables are protected)
- **Telemetry** = the security cameras (monitoring everything)

The file is written in **HCL** (HashiCorp Configuration Language) or **JSON**, and is loaded once at startup.

```
┌─────────────────────────────────────────────────────────┐
│                  vault-config.hcl                        │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │  storage     │  │  listener   │  │    seal      │     │
│  │  "consul"    │  │  "tcp"      │  │  "awskms"   │     │
│  │  { ... }     │  │  { ... }    │  │  { ... }     │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐                       │
│  │ telemetry   │  │   audit     │   Top-level params:   │
│  │ { ... }     │  │  "file"     │   api_addr = "..."    │
│  │             │  │  { ... }    │   ui = true            │
│  └─────────────┘  └─────────────┘   cluster_name = "."  │
│                                      log_level = "INFO"  │
└─────────────────────────────────────────────────────────┘
```

---

## 1. Running Vault with a Config File

```bash
vault server -config /etc/vault.d/vault.hcl
```

That's it — one command, one config file. Compare this to dev mode:

| | Dev Mode | Config File Mode |
|--|----------|-----------------|
| **Command** | `vault server -dev` | `vault server -config <path>` |
| **Storage** | In-memory (gone on restart) | Persistent (Consul, Raft, etc.) |
| **TLS** | Disabled | You configure it |
| **Seal** | Auto-unsealed | Sealed on start |
| **Root Token** | Printed to terminal | Generated during init |
| **Use Case** | Learning / testing | Staging / Production |

> **Production tip:** Use a service manager like **systemd** (Linux) or **Windows Service Manager** to run Vault, so it starts automatically and logs properly.

---

## 2. Configuration Structure

A Vault config file has two types of settings:

### Named Stanzas (blocks with braces)

```hcl
stanza_type "label" {
  parameter1 = "value1"
  parameter2 = "value2"
}
```

### Top-Level Parameters (standalone key-value pairs)

```hcl
api_addr     = "https://vault.example.com:8200"
cluster_addr = "https://vault-node.example.com:8201"
cluster_name = "vault-prod"
ui           = true
log_level    = "INFO"
```

### Skeleton / Template

```hcl
# ─── STORAGE ───────────────────────────────────────────
storage "backend_type" {
  # Where Vault persists ALL data
}

# ─── LISTENER ──────────────────────────────────────────
listener "tcp" {
  # How Vault accepts connections (API + cluster)
}

# ─── SEAL (optional) ──────────────────────────────────
seal "provider" {
  # Auto-unseal configuration
}

# ─── TELEMETRY (optional) ─────────────────────────────
telemetry {
  # Metrics export settings
}

# ─── TOP-LEVEL PARAMETERS ────────────────────────────
api_addr     = "<address>"
cluster_addr = "<address>"
cluster_name = "<name>"
ui           = true
log_level    = "INFO"
```

---

## 3. Key Configuration Components

### Quick Reference Table

| Stanza | What It Does | Required? | Example |
|--------|-------------|-----------|---------|
| `storage` | Persistent data backend | **Yes** | `storage "consul" { ... }` |
| `listener` | Network interface, ports, TLS | **Yes** | `listener "tcp" { address = "0.0.0.0:8200" }` |
| `seal` | Auto-unseal mechanism | No* | `seal "awskms" { region = "us-east-1" }` |
| `telemetry` | Metrics collection & export | No | `telemetry { prometheus_retention_time = "24h" }` |
| `audit` | Audit logging declarations | No | `audit "file" { path = "/var/log/vault_audit.log" }` |

> *Without a `seal` stanza, Vault defaults to **Shamir** — manual unseal on every restart.

### 3.1 Listener Stanza

```hcl
listener "tcp" {
  address                  = "0.0.0.0:8200"    # API port
  cluster_address          = "0.0.0.0:8201"    # Cluster port (HA)
  tls_disable              = false              # NEVER true in production
  tls_cert_file            = "/etc/vault.d/client.pem"
  tls_key_file             = "/etc/vault.d/cert.key"
  tls_disable_client_certs = true
}
```

```
Port Breakdown:

Client/App ──HTTPS──▶ :8200 (API)       ◀── listener.address
                                              (all API calls, UI, CLI)

Vault Node ──HTTPS──▶ :8201 (Cluster)   ◀── listener.cluster_address
                                              (node-to-node HA replication)
```

> **⚠️ Warning:** `tls_disable = true` is insecure. Always set to `false` in production and provide valid TLS certificates.

### 3.2 Storage Stanza

```hcl
storage "consul" {
  address = "127.0.0.1:8500"
  path    = "vault/"
  token   = "1a2b3c4d-1234-abdc-1234-1a2b3c4d5e6a"
}
```

Common backends: `consul`, `raft` (Integrated Storage), `s3`, `dynamodb`, `file`.
(Covered in depth in the next topic — Storage Backends.)

### 3.3 Seal Stanza

```hcl
seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "12345678-abcd-1234-abcd-123456789101"
  endpoint   = "example.kms.us-east-1.vpce.amazonaws.com"
}
```

Common types: `awskms`, `azurekeyvault`, `gcpckms`, `transit`, `pkcs11`.

### 3.4 Telemetry Stanza

```hcl
telemetry {
  prometheus_retention_time = "24h"
  statsd_address            = "statsd.example.com:8125"
  disable_hostname          = true
}
```

### 3.5 Top-Level Parameters

| Parameter | Purpose | Example |
|-----------|---------|---------|
| `api_addr` | Full URL for API redirects (HA) | `"https://vault.example.com:8200"` |
| `cluster_addr` | Full URL for cluster communication | `"https://node1.example.com:8201"` |
| `cluster_name` | Human-readable cluster identifier | `"vault-prod-us-east-1"` |
| `ui` | Enable/disable the web UI | `true` |
| `log_level` | Logging verbosity | `"INFO"`, `"DEBUG"`, `"WARN"`, `"ERROR"`, `"TRACE"` |

---

## 4. Production-Ready Configuration Example

A complete, copy-paste-ready config for a real cluster:

```hcl
# ═══════════════════════════════════════════════════════════
# PRODUCTION VAULT CONFIGURATION
# ═══════════════════════════════════════════════════════════

# ─── STORAGE: Consul backend ──────────────────────────────
storage "consul" {
  address = "127.0.0.1:8500"
  path    = "vault/"
  token   = "1a2b3c4d-1234-abdc-1234-1a2b3c4d5e6a"
}

# ─── LISTENER: TCP with TLS ──────────────────────────────
listener "tcp" {
  address                  = "0.0.0.0:8200"
  cluster_address          = "0.0.0.0:8201"
  tls_disable              = false
  tls_cert_file            = "/etc/vault.d/client.pem"
  tls_key_file             = "/etc/vault.d/cert.key"
  tls_disable_client_certs = true
}

# ─── SEAL: AWS KMS Auto-Unseal ───────────────────────────
seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "12345678-abcd-1234-abcd-123456789101"
  endpoint   = "example.kms.us-east-1.vpce.amazonaws.com"
}

# ─── TOP-LEVEL PARAMETERS ────────────────────────────────
api_addr     = "https://vault-us-east-1.example.com:8200"
cluster_addr = "https://node-us-east-1.example.com:8201"
cluster_name = "vault-prod-us-east-1"
ui           = true
log_level    = "INFO"
```

### What Each Section Does

```
┌──────────────────────────────────────────────────────────────┐
│ storage "consul"                                              │
│   └─ Persists all encrypted data to local Consul agent       │
│                                                               │
│ listener "tcp"                                                │
│   ├─ :8200 → API traffic (CLI, UI, apps)                     │
│   ├─ :8201 → Cluster traffic (HA replication)                │
│   └─ TLS enforced with cert + key files                      │
│                                                               │
│ seal "awskms"                                                 │
│   ├─ Auto-unseal via AWS KMS                                 │
│   └─ VPC endpoint for private network access                 │
│                                                               │
│ api_addr     → Where clients should redirect (HA standby)    │
│ cluster_addr → Where nodes talk to each other                │
│ cluster_name → Friendly identifier for this cluster          │
│ ui = true    → Web UI enabled at /ui                         │
│ log_level    → INFO (balanced verbosity)                     │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. What the Config File Does NOT Manage

This is a critical distinction. The config file defines **how Vault runs**, not **what's inside Vault**.

| Managed by Config File | Managed Inside Vault (CLI/API) |
|------------------------|-------------------------------|
| Storage backend | Secrets Engines |
| Listener / TLS | Auth Methods |
| Seal mechanism | Policies |
| Telemetry | Entities & Groups |
| API/cluster addresses | Tokens |
| UI toggle | Audit device **configuration** |
| Log level | Namespaces (Enterprise) |

```
Config File (BEFORE init)          Vault Runtime (AFTER init + unseal)
┌──────────────────────┐           ┌──────────────────────┐
│ storage "consul"     │           │ vault secrets enable  │
│ listener "tcp"       │           │ vault auth enable     │
│ seal "awskms"        │  ──init──▶│ vault policy write    │
│ ui = true            │  unseal   │ vault audit enable    │
│ log_level = "INFO"   │           │ vault write secret/.. │
└──────────────────────┘           └──────────────────────┘
          ▲                                   ▲
    Loaded at startup              Created at runtime via CLI/API
```

---

## 6. Summary of All Stanzas

| Stanza | Required | Purpose |
|--------|----------|---------|
| `listener` | **Yes** | API and cluster bindings, TLS settings |
| `storage` | **Yes** | Backend for storing Vault data |
| `seal` | No* | Auto-unseal provider (AWS KMS, Azure, GCP, Transit) |
| `telemetry` | No | Metrics publishing settings |
| `audit` | No | Audit device declarations |

> *Without `seal`, Vault uses Shamir's Secret Sharing — manual unseal required at every startup.

### HCL vs JSON

Both formats are supported. HCL is preferred for readability:

```
HCL (preferred):                       JSON (equivalent):
┌────────────────────────┐             ┌──────────────────────────────┐
│ listener "tcp" {       │             │ {                            │
│   address = "0.0.0.0"  │      =      │   "listener": {             │
│   tls_disable = true   │             │     "tcp": {                │
│ }                      │             │       "address": "0.0.0.0", │
│                        │             │       "tls_disable": true   │
└────────────────────────┘             │     }                       │
                                       │   }                         │
                                       │ }                           │
                                       └──────────────────────────────┘
```

---

## Quick Recap

```
Vault Config File (.hcl or .json)
│
├── listener "tcp" { }       ← REQUIRED: ports, TLS, addresses
├── storage  "type" { }      ← REQUIRED: where data is persisted
├── seal     "provider" { }  ← Optional: auto-unseal (no stanza = Shamir)
├── telemetry { }            ← Optional: metrics export
├── audit    "type" { }      ← Optional: audit logging
│
├── api_addr                 ← Top-level: client redirect URL
├── cluster_addr             ← Top-level: node-to-node URL
├── cluster_name             ← Top-level: friendly name
├── ui                       ← Top-level: web UI on/off
└── log_level                ← Top-level: logging verbosity

Start with:  vault server -config /etc/vault.d/vault.hcl
```

---

## Key Exam Tips

1. **Config file format** — HCL or JSON. HCL is the standard for exams and docs.
2. **Two required stanzas** — `listener` and `storage`. Everything else is optional.
3. **No seal stanza = Shamir** — Vault falls back to manual unseal if no seal block is present.
4. **`tls_disable = true` is insecure** — Never use in production. Always provide certs.
5. **Port 8200 = API** (clients, CLI, UI). **Port 8201 = Cluster** (node-to-node HA).
6. **Config file ≠ Vault contents** — Secrets, policies, auth methods, and entities are managed **inside** Vault via CLI/API, not in the config file.
7. **`vault server -config <path>`** is how you start a real Vault server (not `-dev`).
8. **Top-level params to know**: `api_addr`, `cluster_addr`, `cluster_name`, `ui`, `log_level`.
9. **`api_addr`** is used for client redirection in HA setups — standby nodes redirect clients to the active node.

---

## Official References

- [Vault Configuration Docs](https://www.vaultproject.io/docs/configuration)
- [Consul Storage Backend](https://www.vaultproject.io/docs/configuration/storage/consul)
- [AWS KMS Auto-Unseal](https://www.vaultproject.io/docs/configuration/seal/awskms)
- [HashiCorp Vault Getting Started](https://learn.hashicorp.com/vault)
