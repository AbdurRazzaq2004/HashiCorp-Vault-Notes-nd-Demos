# Lab 01 — Unsealing Vault with Key Shards

> **Module 1 — Learning the Vault Architecture**
> Lab: Hands-on walkthrough of initializing and unsealing Vault using Shamir's Secret Sharing.
> 
> **Related Notes:** [04-Unsealing-with-Key-Shards](../Module-01-Learning-the-Vault-Architecture/04-Unsealing-with-Key-Shards.md)

---

## What You'll Do in This Lab

1. Check Vault's initial status (not initialized, sealed)
2. Review the Vault configuration file
3. Initialize Vault (generate key shards + root token)
4. Unseal Vault using 3 of 5 key shards
5. Authenticate and verify Vault is working

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| **Vault installed** | Binary available in `$PATH` (`vault version` should work) |
| **Config file** | Vault HCL config at `/etc/vault.d/vault.hcl` (or your custom path) |
| **Permissions** | Write access to Vault's data and config directories |

---

## Step 1: Check Vault Status

First, let's see what state Vault is in:

```bash
vault status
```

**Expected output** (fresh Vault — never initialized):

```
Key              Value
----             -----
Seal Type        shamir
Initialized      false        ← Not yet initialized
Sealed           true         ← Sealed (obviously, since not initialized)
Total Shares     0
Threshold        0
Unseal Progress  0/0
Unseal Nonce     n/a
Version          1.7.1
Storage Type     raft
HA Enabled       true
```

**Key things to notice:**
- `Initialized: false` — Vault has never been set up
- `Sealed: true` — No encryption key in memory
- `Total Shares: 0` / `Threshold: 0` — No shards exist yet

---

## Step 2: Review Vault Configuration

Let's look at the config file to understand what we're working with:

```bash
cat /etc/vault.d/vault.hcl
```

**Example configuration (Raft storage + TCP listener):**

```hcl
# Storage Backend — Using Integrated Raft
storage "raft" {
  path      = "/opt/vault/data"
  node_id   = "node-a-us-east-1"
  retry_join {
    auto_join = "provider=aws region=us-east-1 tag_key=vault tag_value=us-east-1"
  }
}

# Listener — How clients connect to Vault
listener "tcp" {
  address         = "0.0.0.0:8200"     # Listen on all interfaces, port 8200
  cluster_address = "0.0.0.0:8201"     # Cluster communication port
  tls_disable     = 1                  # TLS disabled (for lab only!)
}

# API and Cluster addresses
api_addr     = "http://10.1.0.37:8200"
cluster_addr = "http://10.1.0.37:8201"
cluster_name = "vault-prod-us-east-1"

# UI and Logging
ui           = true                    # Enable Web UI
log_level    = "INFO"
```

**What to notice:**
- **No `seal` stanza** → Vault uses Shamir's Secret Sharing by default
- **`storage "raft"`** → Using Integrated Raft (built-in HA storage)
- **`tls_disable = 1`** → Only acceptable in labs. NEVER in production!
- **`ui = true`** → Web UI available at `http://<ip>:8200/ui`

---

## Step 3: Initialize Vault

This is the **one-time setup** that creates the master key, splits it into shards, and generates the root token.

```bash
vault operator init
```

**Output:**

```
Unseal Key 1: MxKr/oY8RKMd19gV75hNUK0ExE7JmZjeufCxTNCts+8W9
Unseal Key 2: zy1sDEWUYqLAm8v9F1ukM0Mfs4AIdR3E3FhIZ
Unseal Key 3: 78eRyYcIndlyP2hmOF5pfnAXD6g6d0Phwqxtbgi6
Unseal Key 4: BbTvQb68JE1OlwIgfKFa1wsqRRIxZIlot5I838IzS
Unseal Key 5: tMSPooLeVPBzxfbyMN1CvExInIcbshFJDUN06XnnC8b

Initial Root Token: s.EPAXM61G2egrqULVd61Stphx

Vault initialized with 5 key shares and a key threshold of 3.
```

### ⚠️ SAVE THESE IMMEDIATELY!

```
┌─────────────────────────────────────────────────────────────┐
│  This is the ONLY TIME you will ever see these keys.        │
│  Vault does NOT store them. If you lose them, they're gone. │
│                                                             │
│  In a real environment:                                     │
│  • Give each shard to a DIFFERENT trusted person            │
│  • Store the root token separately                          │
│  • Use PGP encryption for extra security                    │
└─────────────────────────────────────────────────────────────┘
```

**Verify — Vault is initialized but still sealed:**

```bash
vault status
```

```
Key             Value
----            -----
Seal Type       shamir
Initialized     true        ← Now initialized!
Sealed          true        ← But still sealed
Total Shares    5           ← 5 shards created
Threshold       3           ← Need 3 to unseal
Unseal Progress 0/3         ← Waiting for shards
Version         1.7.1
Storage Type    raft
HA Enabled      true
```

---

## Step 4: Unseal Vault

Now we provide 3 of the 5 shards. Run `vault operator unseal` three times:

### Shard 1 of 3:

```bash
vault operator unseal
# Prompt: Unseal Key (will be hidden):
# Paste your first key shard
```

```
Unseal Progress    1/3       ← 1 down, 2 to go
```

### Shard 2 of 3:

```bash
vault operator unseal
# Paste your second key shard (different from the first!)
```

```
Unseal Progress    2/3       ← Almost there
```

### Shard 3 of 3:

```bash
vault operator unseal
# Paste your third key shard
```

```
Key                     Value
---                     -----
Seal Type               shamir
Sealed                  false              ← UNSEALED!
Total Shares            5
Threshold               3
Version                 1.7.1
Storage Type            raft
Cluster Name            vault-prod-us-east-1
Cluster ID              xxx-xxx-xxx-xxx
HA Enabled              true
HA Cluster              n/a
HA Mode                 standby
Raft Committed Index    24
Raft Applied Index      24
```

### What just happened:

```
Shard 4 (Key 4) ──┐
                   ├──→ Reconstructed Master Key ──→ Decrypted Encryption Key ──→ UNSEALED
Shard 1 (Key 1) ──┤
                   │
Shard 5 (Key 5) ──┘

(We used keys 4, 1, and 5 — order and which 3 you pick doesn't matter!)
```

---

## Step 5: Authenticate & Verify

Now let's log in with the root token and make sure everything works:

### Login:

```bash
vault login s.EPAXM61G2egrqULVd61Stphx
```

```
Success! You are now authenticated.
Token policies: ["root"]
```

### List secrets engines:

```bash
vault secrets list
```

```
Path        Type        Accessor                  Description
----        ----        --------                  -----------
cubbyhole/  cubbyhole   cubbyhole_8ab2d9b8        per-token private secret storage
identity/   identity    identity_7e99b119          identity store
sys/        system      system_2ab43a59            system endpoints
```

**What you see:**
- `cubbyhole/` — Token-scoped private storage (always present)
- `identity/` — Identity management (always present)
- `sys/` — System backend (always present)

These are the **reserved paths** we learned about in the notes. No KV engine yet — you'd need to enable one in production.

---

## Lab Complete! Recap

Here's everything we did:

```
Step 1: vault status              → Confirmed: not initialized, sealed
Step 2: cat vault.hcl             → Reviewed config: Raft storage, no seal stanza
Step 3: vault operator init       → Got 5 key shards + root token
Step 4: vault operator unseal ×3  → Provided 3 shards → Vault unsealed
Step 5: vault login + secrets list → Authenticated, verified Vault is working
```

---

## Common Mistakes to Avoid

| Mistake | What Happens | Fix |
|---------|-------------|-----|
| Not saving keys after init | Keys are lost forever | Always save immediately — use a password manager or secure vault |
| Using the same shard twice | Only counts once, progress doesn't advance | Use different shards each time |
| Restarting Vault after unseal | Goes back to sealed state | Unseal again (or use Auto Unseal) |
| Using root token for daily work | Security risk — root has unlimited power | Create proper auth methods and policies, then revoke root token |
| `tls_disable = 1` in production | Traffic is unencrypted | Always use TLS in production |

---

## Try It Yourself — Extra Challenges

1. **Seal Vault manually** and unseal again using 3 different shards than before
   ```bash
   vault operator seal
   vault status
   # Now unseal with a different combination of 3 keys
   ```

2. **Initialize with custom shares** — Try 7 shares with a threshold of 4:
   ```bash
   vault operator init -key-shares=7 -key-threshold=4
   ```

3. **Enable a KV engine** and store your first secret:
   ```bash
   vault secrets enable -path=secret kv-v2
   vault kv put secret/hello target=world
   vault kv get secret/hello
   ```

---

## Official References

- [Vault Getting Started](https://www.vaultproject.io/docs)
- [Vault Operator Init](https://www.vaultproject.io/docs/commands/operator/init)
- [Vault Operator Unseal](https://www.vaultproject.io/docs/commands/operator/unseal)
- [Shamir's Secret Sharing](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing)
