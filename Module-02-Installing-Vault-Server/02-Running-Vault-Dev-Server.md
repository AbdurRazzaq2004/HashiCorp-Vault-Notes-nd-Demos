# Running Vault Dev Server

> **Module 2 — Installing Vault Server | Topic 06**
> Dev mode: instant Vault for learning, testing, and CI — never for production.

---

## What Is Dev Mode?

Think of **dev mode** as Vault's **sandbox** — a one-command playground where everything is pre-configured so you can start experimenting immediately.

```
┌─────────────────────────────────────────────────────────────┐
│                    Dev Server Analogy                        │
│                                                              │
│  Production Vault = A bank with vaults, guards, cameras     │
│  Dev Vault         = A toy safe in your bedroom              │
│                                                              │
│  Both store things, but only one is protected.              │
└─────────────────────────────────────────────────────────────┘
```

---

## Dev Server Feature Summary

| Feature | Dev Server Behavior |
|---------|-------------------|
| **Initialization** | Automatically initialized |
| **Seal State** | Automatically unsealed |
| **Storage** | In-memory only (data lost on stop) |
| **TLS** | Disabled (HTTP only) |
| **Root Token** | Pre-set (you choose it) |
| **KV Engine** | v2 pre-mounted at `secret/` |
| **Unseal Keys** | Single key, printed to stdout |
| **UI** | Available at `http://127.0.0.1:8200/ui` |

---

## When To Use Dev Mode

| Use Case | Example |
|----------|---------|
| **Learning & Exploration** | Following tutorials, testing CLI commands |
| **Application Development** | App integration testing with secrets |
| **CI/CD Pipelines** | Ephemeral Vault for automated tests |
| **Proof of Concept** | Demonstrating Vault features to a team |
| **Policy Testing** | Validating ACL policies before production |

> **⚠️ Never use dev mode in production.** No TLS + in-memory storage + no auth = zero security.

---

## Starting the Dev Server

### Basic Launch

```bash
vault server -dev
```

Output includes:

```
==> Vault server configuration:
             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Go Version: go1.17.5
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster: "127.0.0.1:8201", tls: "disabled")
               Log Level: info
                   Mlock: supported: false, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.9.2

WARNING! dev mode is enabled!
You may need to set the following environment variables:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

Unseal Key: O9LjuQxW...
Root Token: hvs.mE0Y...
```

### Launch with a Custom Root Token

```bash
vault server -dev -dev-root-token-id="my-root-token"
```

This lets you set a predictable token for scripts and automation.

### Custom Listen Address

```bash
vault server -dev -dev-listen-address="0.0.0.0:8200"
```

Useful if you need to access the dev server from other machines.

---

## Connecting to the Dev Server

Open a **new terminal** (the server occupies the first one):

```bash
# Point CLI to the dev server
export VAULT_ADDR='http://127.0.0.1:8200'

# Authenticate with the root token
export VAULT_TOKEN='my-root-token'
# OR
vault login my-root-token
```

---

## Checking Status

```bash
vault status
```

```
Key             Value
---             -----
Seal Type       shamir
Initialized     true        ← Auto-initialized
Sealed          false       ← Auto-unsealed
Total Shares    1           ← Single unseal key
Threshold       1
Version         1.9.2
Storage Type    inmem       ← In-memory (no persistence)
Cluster Name    vault-cluster-abc123
Cluster ID      4e7c2c4e-...
HA Enabled      false       ← No HA in dev mode
```

---

## Quick Test: Write and Read a Secret

```bash
# Write a secret (KV v2 is pre-mounted at secret/)
vault kv put secret/myapp username="admin" password="s3cret"

# Read it back
vault kv get secret/myapp

# Read a specific field
vault kv get -field=password secret/myapp
```

---

## Dev vs Production Comparison

```
┌──────────────────┬───────────────────┬─────────────────────┐
│   Aspect         │   Dev Server      │   Production        │
├──────────────────┼───────────────────┼─────────────────────┤
│ Start command    │ vault server -dev │ vault server -config│
│ Init needed?     │ No (auto)         │ Yes (manual)        │
│ Unsealed?        │ Yes (auto)        │ No (manual/auto)    │
│ Storage          │ In-memory         │ Raft / Consul       │
│ Data persists?   │ No                │ Yes                 │
│ TLS              │ Off               │ Required            │
│ Root token       │ Pre-set           │ Generated at init   │
│ UI               │ http://...        │ https://...         │
│ Mlock            │ Disabled          │ Enabled             │
│ HA               │ Not available     │ Available           │
└──────────────────┴───────────────────┴─────────────────────┘
```

---

## Stopping the Dev Server

```bash
# In the terminal where it's running:
Ctrl+C

# All data is gone — in-memory storage vanishes
```

---

## Quick Recap

```
Dev Server
│
├── Start:   vault server -dev [-dev-root-token-id="token"]
├── Config:  export VAULT_ADDR='http://127.0.0.1:8200'
├── Auth:    export VAULT_TOKEN='my-root-token'
├── Status:  vault status → Initialized=true, Sealed=false
├── Test:    vault kv put secret/app key=value
├── UI:      http://127.0.0.1:8200/ui
│
└── Remember:
    ├── In-memory ONLY (data lost on restart)
    ├── No TLS (HTTP only)
    ├── Single unseal key
    └── NEVER use in production
```

---

## Key Exam Tips

1. Dev mode auto-initializes AND auto-unseals — no `vault operator init` or `vault operator unseal` needed
2. Storage type is `inmem` — all data is lost when the server stops
3. TLS is **disabled** by default in dev mode
4. KV v2 secrets engine is pre-mounted at `secret/`
5. Root token can be set with `-dev-root-token-id` flag
6. Dev server listens on `127.0.0.1:8200` by default (loopback only)
7. `-dev-listen-address="0.0.0.0:8200"` makes it accessible from other hosts

---

## Official References

- [Vault Dev Server Mode](https://developer.hashicorp.com/vault/docs/concepts/dev-server)
- [Vault CLI: server Command](https://developer.hashicorp.com/vault/docs/commands/server)
- [KodeKloud: HashiCorp Vault](https://kodekloud.com)
