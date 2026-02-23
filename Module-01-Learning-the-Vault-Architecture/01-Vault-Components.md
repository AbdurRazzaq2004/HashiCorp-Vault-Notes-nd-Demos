# Vault Components

> **Module 1 — Learning the Vault Architecture**
> Topic: The four core building blocks of HashiCorp Vault.

---

## Overview

Think of Vault like a **high-security bank**. To run that bank, you need four things:

| Component | Bank Analogy | What It Does in Vault |
|-----------|-------------|----------------------|
| **Storage Backends** | The vault room (where gold is kept) | Where Vault stores all its data |
| **Secrets Engines** | The different lockers inside | Manage, generate, or encrypt secrets |
| **Auth Methods** | The ID verification at the entrance | Verify who you are before giving access |
| **Audit Devices** | The security cameras | Log every single action for accountability |

Let's understand each one in detail.

---

## 1. Storage Backends

### What is it?

A Storage Backend is simply **where Vault keeps all its data** — encryption keys, secrets, configurations, everything.

Think of it as the **hard drive** of Vault.

### Key Points

- Vault uses **exactly ONE** storage backend per cluster — no mixing and matching.
- All data is **encrypted before** it hits the storage backend:
  - **In Transit** — secured with TLS (like HTTPS).
  - **At Rest** — encrypted with AES-256 (military-grade encryption).
- The storage backend itself **never sees unencrypted data**. It just stores blobs of encrypted bytes.

### Popular Storage Backends

| Backend | High Availability? | Best For |
|---------|-------------------|----------|
| **Consul** | Yes | Production — native HA, leader election, snapshots |
| **Integrated Raft** | Yes | Production — built-in, no external dependencies |
| **DynamoDB** | Yes | AWS-native setups, horizontal scaling |
| **File System** | No | Dev/testing only — simple but no HA |
| **S3** | No | Backup/archival scenarios |

### How to Configure?

You define the storage backend in the Vault config file (`vault.hcl`):

```hcl
storage "consul" {
  address = "127.0.0.1:8500"
  path    = "vault/"
}
```

Or with Integrated Raft (recommended for simplicity):

```hcl
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "node1"
}
```

### Remember This

> One cluster = One storage backend. Want HA or geo-replication? Run multiple clusters with separate backends.

---

## 2. Secrets Engines

### What is it?

Secrets Engines are **plugins** that store, generate, or encrypt data. They are the **core reason Vault exists** — to manage secrets.

Think of them as **different departments** inside the bank — one handles cash, another handles gold, another handles crypto.

### Key Points

- Secrets Engines are **mounted at specific paths** (like file system paths).
- Each engine is **isolated** — what's at `secret/` cannot see what's at `database/`.
- You can have **multiple instances** of the same engine on different paths.
- Interaction happens via **CLI, API, or UI**.

### Common Secrets Engines

| Engine | What It Does | Example Use Case |
|--------|-------------|-----------------|
| **KV (Key/Value)** | Stores static secrets (like a password safe) | Store API keys, passwords, config values |
| **Database** | Generates **dynamic** DB credentials on the fly | App needs a MySQL password → Vault creates one, auto-revokes later |
| **AWS / GCP / Azure** | Creates **dynamic** cloud IAM credentials | CI/CD pipeline needs temporary AWS access |
| **Transit** | Encryption-as-a-Service (encrypt/decrypt without storing) | App sends data → Vault encrypts it → App stores encrypted data |
| **PKI** | Generates TLS/SSL certificates | Auto-generate certs for your microservices |

### How to Enable?

```bash
# Enable the KV secrets engine at the default path
vault secrets enable kv

# Enable the database secrets engine at a custom path
vault secrets enable -path=mydb database
```

### The Big Idea

> Secrets Engines don't just **store** secrets — some **generate** them dynamically. Dynamic secrets are more secure because they're short-lived and unique.

---

## 3. Auth Methods

### What is it?

Auth Methods are **how Vault verifies your identity**. Before you can read or write any secret, Vault needs to know **who you are**.

Think of it as the **login system** — you prove your identity, and Vault gives you a **token** (like a visitor badge) with specific permissions.

### Key Points

- The **end goal** of every auth method is to get a **Vault token**.
- Vault comes with the **Token auth method** enabled by default (that's how the root token works).
- Auth methods are split into two categories:

| Type | For Whom? | Examples |
|------|----------|---------|
| **Human-centric** | People / Users | LDAP, OIDC (SSO), Username & Password, GitHub |
| **Machine-centric** | Apps / Services | AppRole, Kubernetes, TLS Certificates, AWS IAM |

### How Does It Work?

```
You (human/app) ──authenticate──> Auth Method ──verified──> Vault Token ──access──> Secrets
```

1. You authenticate using your method (e.g., LDAP credentials).
2. Vault verifies with the external system (e.g., your LDAP server).
3. If valid, Vault issues a **token** with attached **policies** (permissions).
4. You use that token for all future requests.

### How to Enable?

```bash
# Enable LDAP auth
vault auth enable ldap

# Enable Kubernetes auth
vault auth enable kubernetes

# Enable AppRole (for applications)
vault auth enable approle
```

### Important

> The **root token** from initialization is all-powerful. Use it **only** for initial setup, then **revoke it** and use safer auth methods for daily operations.

---

## 4. Audit Devices

### What is it?

Audit Devices are **security cameras** for Vault. They record **every single request and response** in JSON format.

No action goes unlogged. Period.

### Key Points

- **Every request** to Vault is logged — who asked, what they asked, and what Vault responded.
- Logs are in **JSON format** — easy to ship to SIEM tools (Splunk, ELK, Datadog, etc.).
- **Sensitive data is hashed** (using HMAC-SHA256) — so secrets don't appear in plain text in logs.
- You can enable **multiple audit devices** simultaneously (file + syslog, for example).

### The Safety Rule (This is IMPORTANT!)

> If **all** audit devices become unavailable (e.g., disk full), Vault will **STOP processing requests entirely**.

Why? Because Vault prioritizes **security over availability**. It would rather go down than operate without an audit trail.

### Supported Audit Devices

| Device | Where Logs Go |
|--------|--------------|
| **File** | Local file on disk |
| **Syslog** | System logging daemon |
| **Socket** | TCP/UDP remote endpoint |

### How to Enable?

```bash
# Enable file-based audit logging
vault audit enable file file_path=/var/log/vault/audit.log

# Enable syslog audit
vault audit enable syslog

# Enable socket audit (send to remote)
vault audit enable socket address=127.0.0.1:9090 socket_type=tcp
```

---

## Quick Recap — The 4 Pillars

```
┌─────────────────────────────────────────────────────┐
│                   HashiCorp Vault                    │
├─────────────┬──────────────┬───────────┬────────────┤
│   Storage   │   Secrets    │   Auth    │   Audit    │
│  Backends   │   Engines    │  Methods  │  Devices   │
├─────────────┼──────────────┼───────────┼────────────┤
│ WHERE data  │ WHAT secrets │ WHO gets  │ LOGGING    │
│ is stored   │ are managed  │ access    │ everything │
├─────────────┼──────────────┼───────────┼────────────┤
│ Consul,Raft │ KV,Database  │ LDAP,K8s  │ File,      │
│ DynamoDB,S3 │ AWS,Transit  │ AppRole   │ Syslog     │
└─────────────┴──────────────┴───────────┴────────────┘
```

---

## Key Exam Tips

1. **Storage Backend** — Only ONE per cluster. Data is always encrypted before storage.
2. **Secrets Engines** — Isolated by path. Can be static (KV) or dynamic (Database, AWS).
3. **Auth Methods** — Everything ends with a TOKEN. Root token should be revoked after setup.
4. **Audit Devices** — If all audit devices fail, Vault STOPS. Security > Availability.

---

## Official References

- [Vault Storage Backends](https://www.vaultproject.io/docs/configuration/storage)
- [Vault Secrets Engines](https://www.vaultproject.io/docs/secrets)
- [Vault Auth Methods](https://www.vaultproject.io/docs/auth)
- [Vault Audit Devices](https://www.vaultproject.io/docs/audit)

---

> **Labs/Demos** — Coming soon! We'll set up each component hands-on.
