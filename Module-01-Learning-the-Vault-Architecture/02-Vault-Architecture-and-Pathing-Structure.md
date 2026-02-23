# Vault Architecture & Pathing Structure

> **Module 1 — Learning the Vault Architecture**
> Topic: How Vault is built internally and how it routes every request using paths.

---

## Overview

In the previous topic, we learned the **4 components** of Vault. Now let's zoom in and understand:

1. **How Vault is structured internally** (the architecture)
2. **How every request finds the right component** (path-based routing)

Think of it this way — if the 4 components are the **departments** in a bank, the architecture is the **building blueprint** and the pathing system is the **hallway navigation** that directs you to the right department.

---

## 1. Vault Architecture

### The Big Picture

Vault talks to the outside world through **one door only** — its **HTTPS API**.

Whether you use the CLI, the Web UI, or call it from your application — underneath, everything hits the same API over a secure TLS connection.

### Internal Layers (Outside → Inside)

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENTS                               │
│         (CLI, UI, Apps, Terraform, etc.)                 │
└────────────────────┬────────────────────────────────────┘
                     │ HTTPS / TLS
                     ▼
┌─────────────────────────────────────────────────────────┐
│                   HTTP API LAYER                         │
│            (The only entry point)                        │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│             ╔═══════════════════════╗                    │
│             ║  CRYPTOGRAPHIC BARRIER ║  ◄── The "wall"  │
│             ╚═══════════════════════╝                    │
│  ┌──────────────────────────────────────────────────┐   │
│  │               VAULT CORE                         │   │
│  │                                                  │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐     │   │
│  │  │ Policy   │ │ Token    │ │ Path Router  │     │   │
│  │  │ Store    │ │ Store    │ │              │     │   │
│  │  └──────────┘ └──────────┘ └──────────────┘     │   │
│  │                                                  │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐     │   │
│  │  │ Secrets  │ │ Auth     │ │ Audit        │     │   │
│  │  │ Engines  │ │ Methods  │ │ Devices      │     │   │
│  │  └──────────┘ └──────────┘ └──────────────┘     │   │
│  │                                                  │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │         System Backend (/sys)            │    │   │
│  │  └──────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────┘   │
│             ╔═══════════════════════╗                    │
│             ║  CRYPTOGRAPHIC BARRIER ║                   │
│             ╚═══════════════════════╝                    │
└────────────────────┬────────────────────────────────────┘
                     │ Always Encrypted
                     ▼
┌─────────────────────────────────────────────────────────┐
│                 STORAGE BACKEND                          │
│        (Consul, Raft, DynamoDB, S3, etc.)               │
│        Vault NEVER stores unencrypted data here         │
└─────────────────────────────────────────────────────────┘
```

### The Three Layers Explained

#### Layer 1: HTTP API
- The **only way** to communicate with Vault.
- Even the `vault` CLI is just a wrapper that makes API calls.
- Always runs over **TLS** (HTTPS).

#### Layer 2: The Cryptographic Barrier (The Security Wall)

This is the **most important concept** in Vault's architecture.

Think of it as an **unbreakable glass wall** around Vault's core:

- **Data going OUT** to storage → gets **encrypted** before crossing the barrier.
- **Data coming IN** from storage → gets **decrypted** only after authentication.
- **Nothing passes through** unless it's encrypted or the caller has a valid token.

> The barrier is what makes Vault trustworthy even when the storage backend is untrusted.

You could literally store Vault's data on a public S3 bucket and it would still be secure — because the storage backend only ever sees encrypted blobs.

#### Layer 3: Vault Core

Inside the barrier, you find everything:

| Component | Role |
|-----------|------|
| **Path Router** | Directs requests to the right engine/method based on the URL path |
| **Token Store** | Manages all Vault tokens (creation, lookup, renewal, revocation) |
| **Policy Store** | Stores and enforces access control policies |
| **Secrets Engines** | Handle secret operations (KV, Database, Transit, etc.) |
| **Auth Methods** | Verify identities and issue tokens |
| **Audit Devices** | Log every request and response |
| **System Backend** | Internal management mounted at `/sys` |

---

## 2. Vault Paths & Routing

### Everything in Vault is a Path

This is a **fundamental concept**: every single operation in Vault is mapped to a **URL-like path**.

```
vault read secret/data/myapp/config
             └──────────────────────┘
                  This is the PATH
```

When Vault receives a request, the **Path Router** looks at the path prefix and sends it to the correct component:

```
Request Path                    →  Routed To
─────────────────────────────────────────────────
secret/data/myapp/config        →  KV Secrets Engine (mounted at secret/)
database/creds/my-role          →  Database Secrets Engine (mounted at database/)
auth/ldap/login/john            →  LDAP Auth Method (mounted at auth/ldap/)
sys/policies/acl/my-policy      →  System Backend (mounted at sys/)
```

### Mounting = Giving a Component its Address

When you **enable** a secrets engine or auth method, you're **mounting** it at a path — like assigning a room number in a building.

```bash
# Default path (uses the engine name)
vault secrets enable database
# ↑ Mounts at: database/

# Custom path (you choose the name)
vault secrets enable -path=my-postgres database
# ↑ Mounts at: my-postgres/
```

Same for auth methods:

```bash
# Default path
vault auth enable userpass
# ↑ Mounts at: auth/userpass/

# Custom path
vault auth enable -path=my-azure azure
# ↑ Mounts at: auth/my-azure/
```

### Why Custom Paths?

You might want **multiple instances** of the same engine for different teams or environments:

```bash
vault secrets enable -path=team-a/secrets kv
vault secrets enable -path=team-b/secrets kv
# Now each team has their own isolated KV store!
```

---

## 3. System-Reserved Paths

Vault has certain paths that are **always present** and **cannot be removed**. These are built into Vault's core:

| Reserved Path | Purpose | Can You Disable It? |
|---------------|---------|-------------------|
| `sys/` | System operations — manage policies, mounts, audit, health checks, leader status | No |
| `auth/token/` | Token management — create, renew, revoke tokens (default auth method) | No |
| `cubbyhole/` | Token-scoped private storage — each token gets its own isolated cubbyhole | No |
| `identity/` | Identity management — entities, groups, aliases | No |
| `secret/` | Default KV v2 engine (**dev mode only!**) | N/A |

### Important Gotcha

> In **dev mode**, Vault automatically enables a KV v2 engine at `secret/`. In **production**, it does NOT. You must explicitly enable it yourself.

```bash
# In production, you need to do this yourself:
vault secrets enable -path=secret kv-v2
```

---

## How It All Flows Together

Here's a complete request lifecycle:

```
1. User runs:  vault kv get secret/data/myapp/password
                           │
2. CLI converts to:  HTTP GET https://vault:8200/v1/secret/data/myapp/password
                           │
3. API Layer receives the request
                           │
4. Auth check: Is the token valid? Does it have the right policy?
                           │
5. Path Router: "secret/" prefix → route to KV Secrets Engine
                           │
6. KV Engine: Fetch data from storage
                           │
7. Cryptographic Barrier: Decrypt the data from storage
                           │
8. Audit Device: Log the request + response (with hashed sensitive fields)
                           │
9. Return the secret to the user
```

---

## Quick Recap

| Concept | One-Liner |
|---------|-----------|
| **API** | The only door into Vault — always HTTPS |
| **Cryptographic Barrier** | Encrypts everything going to storage, decrypts on the way back |
| **Path Router** | Reads the URL path and sends the request to the right component |
| **Mounting** | Assigning a path address to an engine or auth method |
| **Reserved Paths** | `sys/`, `auth/token/`, `cubbyhole/`, `identity/` — always there |
| **Dev vs Prod** | Dev auto-enables `secret/` KV v2; production does NOT |

---

## Key Exam Tips

1. **Vault ONLY communicates via HTTPS API** — CLI and UI are just wrappers around it.
2. **Cryptographic Barrier** = the trust boundary. Storage backend is UNTRUSTED. Everything is encrypted before leaving the barrier.
3. **Path-based routing** — every request goes to a component based on its path prefix.
4. **Custom mount paths** using `-path=` flag. Default path = component name.
5. **`secret/` path exists by default in dev mode ONLY** — not in production!
6. **Reserved paths** (`sys/`, `cubbyhole/`, `identity/`, `auth/token/`) cannot be disabled.

---

## Official References

- [Vault Architecture Overview](https://www.vaultproject.io/docs/architecture)
- [Vault HTTP API](https://www.vaultproject.io/api-docs)
- [Secrets Engines](https://www.vaultproject.io/docs/secrets)
- [Authentication Methods](https://www.vaultproject.io/docs/auth)

---

> **Labs/Demos** — Coming soon! We'll explore paths and mounts hands-on.
