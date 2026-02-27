# Demo: Running Vault Dev Server

> **Module 2 — Installing Vault Server**
> Lab: Launch Vault in dev mode, explore the UI, write and read secrets.

---

## Lab Overview

This lab demonstrates launching Vault in **development mode** — the fastest way to get a working Vault instance. Dev mode runs entirely in-memory, starts pre-unsealed, and gives you a root token immediately. Perfect for learning and testing.

```
┌────────────────────────────────────────────────────────┐
│                Dev Server in Action                     │
│                                                         │
│  Terminal 1 (server):     Terminal 2 (client):          │
│  vault server -dev        export VAULT_ADDR=...         │
│  │                        vault status                  │
│  │ Unseal Key: xxx        vault kv put secret/...       │
│  │ Root Token: xxx        vault kv get secret/...       │
│  │                                                      │
│  └─ Runs in foreground    └─ Interact with Vault        │
└────────────────────────────────────────────────────────┘
```

> **⚠️ WARNING:** Dev mode is NOT secure. Never use in production.

---

## Step 1: Start Vault in Dev Mode

```bash
vault server -dev
```

Output:

```
WARNING! Dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

  export VAULT_ADDR='http://127.0.0.1:8200'

Unseal Key: ZEgZgHSEEmmlnRboqtY0A00TUpleaoxo8SqqtFP2Q=
Root Token: s.d6931rVSdkpBINnnRvMHBRXR

Development mode should NOT be used in production installations!
```

> This runs in the **foreground** — open a second terminal for the next steps.

---

## Step 2: Configure Your Environment

In a **new terminal window**:

```bash
# Linux / macOS
export VAULT_ADDR='http://127.0.0.1:8200'

# Windows PowerShell
$env:VAULT_ADDR = "http://127.0.0.1:8200"

# Windows Command Prompt
set VAULT_ADDR=http://127.0.0.1:8200
```

---

## Step 3: Check Vault Status

```bash
vault status
```

```
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.7.0
Storage Type    inmem        ← In-memory! Data gone on restart
HA Enabled      false        ← No HA in dev mode
```

---

## Step 4: List Enabled Secrets Engines

```bash
vault secrets list
```

| Path | Type | Description |
|------|------|-------------|
| `cubbyhole/` | cubbyhole | Per-token private secret storage |
| `identity/` | identity | Identity store |
| `secret/` | kv | Versioned key/value (KV v2) — pre-mounted! |
| `sys/` | system | System endpoints for control and debugging |

> **Note:** The `secret/` KV v2 engine is **automatically enabled** in dev mode — in production you'd have to enable it manually.

---

## Step 5: Write and Read Secrets

### Write a Secret

```bash
vault kv put secret/vaultcourse/bryan bryan=bryan
```

```
Key            Value
---            -----
created_time   2021-05-12T12:27:09.504562727Z
deletion_time  n/a
destroyed      false
version        1
```

### Read a Secret

```bash
vault kv get secret/vaultcourse/bryan
```

```
=== Metadata ===
Key            Value
---            -----
created_time   2021-05-12T12:27:09.504562727Z
deletion_time  n/a
destroyed      false
version        1

=== Data ===
Key    Value
---    -----
bryan  bryan
```

---

## Step 6: Access the Web UI

Open your browser and go to:

```
http://127.0.0.1:8200/ui
```

Log in with the **Root Token** from the dev server output.

---

## Step 7: Clean Up

Press `Ctrl+C` in the terminal running the dev server. All in-memory data is instantly lost — every restart gives you a clean slate.

---

## Quick Recap

```
Dev Server Lab
│
├── vault server -dev           ← Start (foreground)
├── export VAULT_ADDR=...       ← Point CLI to dev server
├── vault status                ← Confirm: initialized, unsealed, inmem
├── vault secrets list          ← See pre-mounted engines
├── vault kv put/get            ← Write and read secrets
├── http://127.0.0.1:8200/ui   ← Web UI access
└── Ctrl+C                      ← Stop (all data gone)
```

---

## Official References

- [Vault CLI Documentation](https://www.vaultproject.io/docs/commands)
- [Vault Dev Server Mode](https://www.vaultproject.io/docs/commands/server#dev-server)
- [Vault Secrets Engines](https://www.vaultproject.io/docs/secrets)
