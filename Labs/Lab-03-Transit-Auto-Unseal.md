# Lab 03 — Transit Auto Unseal (Vault Unseals Vault)

> **Module 1 — Learning the Vault Architecture**
> Lab: Configure one Vault cluster to automatically unseal another using the Transit Secrets Engine.
>
> **Related Notes:** [06-Unsealing-with-Transit-Auto-Unseal](../Module-01-Learning-the-Vault-Architecture/06-Unsealing-with-Transit-Auto-Unseal.md)

---

## What You'll Do in This Lab

```
Part 1: Set up the Transit Cluster  (the "unsealer")
Part 2: Configure the Target Cluster (the one that auto-unseals)
Part 3: Initialize & verify auto unseal
Part 4: Test it — restart and confirm it stays unsealed
```

---

## Environment Overview

We're working with **two separate Vault clusters**:

```
┌──────────────────────────────┐     ┌──────────────────────────────┐
│   TRANSIT CLUSTER            │     │   TARGET CLUSTER             │
│   (The Unsealer)             │     │   (The one being unsealed)   │
│                              │     │                              │
│   IP: 10.0.1.209             │     │   IP: 10.0.1.37             │
│   Role: Runs Transit Engine  │     │   Role: Raft-backed Vault   │
│   Status: Already unsealed   │     │   Status: Not yet initialized│
└──────────────────────────────┘     └──────────────────────────────┘
         │                                       ▲
         │      "Here's your decrypted key"      │
         └───────────────────────────────────────┘
```

### Prerequisites

| Requirement | Details |
|-------------|---------|
| **Two Vault nodes** | SSH access to both (Transit at `10.0.1.209`, Target at `10.0.1.37`) |
| **Network** | Port `8200` open between both nodes |
| **Vault CLI** | Installed on both nodes (`vault version` works) |
| **Transit Cluster** | Already initialized and unsealed |

### Open two terminal sessions:

```bash
# Terminal 1 — Transit Cluster
ssh ec2-user@10.0.1.209

# Terminal 2 — Target Cluster
ssh ec2-user@10.0.1.37
```

---

## Part 1: Configure the Transit Cluster (The Unsealer)

> **All commands in this section run on the Transit Cluster (10.0.1.209)**

### Step 1.1 — Enable the Transit Secrets Engine

```bash
# Check what's currently enabled
vault secrets list

# Enable Transit
vault secrets enable transit
```

```
Success! Enabled the transit secrets engine at: transit/
```

### Step 1.2 — Create an Encryption Key

This is the key that will encrypt/decrypt the target cluster's master key:

```bash
# Create the key
vault write -f transit/keys/unseal-key

# Verify it was created
vault list transit/keys
```

```
Keys
----
unseal-key
```

### Step 1.3 — Create an Unseal Policy

The target cluster needs **minimal permissions** — only encrypt and decrypt using this specific key.

Create a policy file:

```bash
cat > policy.hcl <<EOF
path "transit/encrypt/unseal-key" {
  capabilities = ["update"]
}

path "transit/decrypt/unseal-key" {
  capabilities = ["update"]
}
EOF
```

Upload it to Vault:

```bash
vault policy write unseal policy.hcl
```

```
Success! Uploaded policy: unseal
```

### Step 1.4 — Create a Token for the Target Cluster

```bash
vault token create -policy=unseal
```

```
Key                  Value
---                  -----
token                s.v9hDNIycSM8ZL7wsFo9vD0i    ← SAVE THIS!
token_accessor       ...
token_duration       768h
token_renewable      true
token_policies       ["default" "unseal"]
```

### Save that token! The target cluster needs it.

```
┌─────────────────────────────────────────────────────────────────┐
│  Token: s.v9hDNIycSM8ZL7wsFo9vD0i                              │
│                                                                 │
│  This token can ONLY encrypt/decrypt with the unseal-key.       │
│  It CANNOT read secrets, manage policies, or do anything else.  │
│  (Least-privilege principle in action!)                          │
└─────────────────────────────────────────────────────────────────┘
```

### Part 1 Summary — What we set up on Transit:

```
Transit Cluster (10.0.1.209)
├── Transit Secrets Engine: enabled at transit/
├── Encryption Key: unseal-key
├── Policy: unseal (encrypt + decrypt only)
└── Token: s.v9hDNIycSM8ZL7wsFo9vD0i (scoped to unseal policy)
```

---

## Part 2: Configure the Target Cluster

> **All commands in this section run on the Target Cluster (10.0.1.37)**

### Step 2.1 — Check Current Status

```bash
vault status
```

```
Key              Value
----             -----
Seal Type        shamir          ← Default, we'll change this
Initialized      false
Sealed           true
```

### Step 2.2 — Update Vault Configuration

Edit the config:

```bash
sudo vi /etc/vault.d/vault.hcl
```

Add the `seal "transit"` stanza. Your complete config should look like:

```hcl
# Storage — Raft (integrated)
storage "raft" {
  path    = "/opt/vault3/data"
  node_id = "node-us-east-1"

  retry_join {
    auto_join = "provider=aws region=us-east-1 tag_key=vault tag_value=us-east-1"
  }
}

# ─── TRANSIT AUTO UNSEAL — THE KEY ADDITION ───
seal "transit" {
  address    = "http://10.0.1.209:8200"          # Transit cluster address
  token      = "s.v9hDNIycSM8ZL7wsFo9vD0i"       # Token from Step 1.4
  key_name   = "unseal-key"                       # Key from Step 1.2
  mount_path = "transit"                          # Where Transit engine is mounted
}
# ───────────────────────────────────────────────

# Listener
listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_disable     = true
}

# Addresses
api_addr     = "http://10.0.1.37:8200"
cluster_addr = "http://10.0.1.37:8201"
cluster_name = "vault-prod-us-east-1"
ui           = true
log_level    = "INFO"
```

### What each `seal "transit"` field connects to:

```
seal "transit" {
  address    = "http://10.0.1.209:8200"     ──→ Transit cluster IP
  token      = "s.v9hDNIycSM8ZL7w..."       ──→ Token from Step 1.4
  key_name   = "unseal-key"                  ──→ Key from Step 1.2
  mount_path = "transit"                     ──→ Engine path from Step 1.1
}
```

### Step 2.3 — Restart Vault

```bash
sudo systemctl restart vault
```

Verify the seal type changed:

```bash
vault status
```

```
Key                  Value
----                 -----
Seal Type            transit        ← Changed from "shamir" to "transit"!
Initialized          false
Sealed               true
Recovery Seal Type   n/a
```

The seal type is now `transit` — Vault knows to call the Transit cluster for unsealing.

---

## Part 3: Initialize and Verify Auto Unseal

### Initialize the target cluster:

```bash
vault operator init
```

```
Recovery Key 1:  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Recovery Key 2:  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Recovery Key 3:  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Recovery Key 4:  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Recovery Key 5:  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Initial Root Token: s.XXXXXXXXXXXXXXXXXXXXXXXX

Success! Vault is initialized with 5 recovery shares and a threshold of 3.
```

### Notice: Recovery Keys, NOT Unseal Keys!

Same as with Cloud Auto Unseal — recovery keys **cannot** unseal Vault. They're for admin operations like rekeying.

### Verify — Vault should already be unsealed:

```bash
vault status
```

```
Key                      Value
---                      -----
Seal Type                transit
Recovery Seal Type       shamir
Initialized              true
Sealed                   false           ← AUTO-UNSEALED!
Total Recovery Shares    5
Threshold                3
Version                  1.7.1
Storage Type             raft
Cluster Name             vault-prod-us-east-1
Cluster ID               xxxx-xxxx-xxxx
HA Enabled               true
HA Mode                  active
```

### What just happened behind the scenes:

```
Target Cluster                  Transit Cluster
──────────────                  ───────────────
1. Generated master key
2. Sent to Transit ──────────── 3. Encrypted with unseal-key
4. Stored encrypted        ◄── 5. Returned encrypted blob
   master key in Raft
6. Asked Transit: decrypt ───── 7. Decrypted master key
8. Received plaintext      ◄── 9. Returned plaintext
   master key
10. Unlocked encryption key
11. UNSEALED ✅
```

---

## Part 4: Post-Unseal — Use Vault & Test Restart

### Login with root token:

```bash
vault login <initial-root-token>
```

```
Success! You are now authenticated. Token policies: ["root"]
```

### Enable some engines and store data:

```bash
# Enable Azure secrets engine
vault secrets enable azure

# Enable a KV engine at a custom path
vault secrets enable -path=vaultcourse kv

# Store a secret
vault kv put vaultcourse/bryan bryan=bryan

# Read it back
vault kv get vaultcourse/bryan
```

```
=== Data ===
Key      Value
---      -----
bryan    bryan
```

### The Real Test — Restart Vault:

```bash
sudo systemctl restart vault
```

Wait a moment, then:

```bash
vault status
```

```
Sealed    false          ← Still unsealed after restart!
```

**Transit Auto Unseal is working!** Every time this cluster restarts, it calls the Transit cluster to decrypt its master key automatically.

---

## Lab Complete! Full Recap

```
TRANSIT CLUSTER (10.0.1.209)          TARGET CLUSTER (10.0.1.37)
────────────────────────────          ──────────────────────────

1. Enable transit engine              
2. Create unseal-key                  
3. Create unseal policy               
4. Create token                       
                                      5. Add seal "transit" to config
                                      6. Restart vault
                                      7. vault operator init
                                         → Recovery keys (not unseal keys)
                                         → Auto-unsealed immediately!
                                      8. vault login → use normally
                                      9. Restart → still unsealed ✅
```

---

## Comparison: All Three Labs

| | Lab 01 (Shamir) | Lab 02 (AWS KMS) | Lab 03 (Transit) |
|---|-----------------|-------------------|-------------------|
| **Config change** | None (default) | Add `seal "awskms"` | Add `seal "transit"` |
| **External dependency** | Humans | AWS KMS service | Another Vault cluster |
| **`vault operator init` gives** | Unseal Keys | Recovery Keys | Recovery Keys |
| **After init** | Still sealed | Auto-unsealed | Auto-unsealed |
| **After restart** | Sealed again | Auto-unsealed | Auto-unsealed |
| **Setup steps** | 1 (just init) | 2 (KMS + config) | 4 (engine, key, policy, token + config) |

---

## Common Issues & Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `error connecting to transit cluster` | Network issue | Check port 8200 is open between the two nodes |
| `permission denied` on Transit | Token doesn't have the right policy | Verify token was created with `-policy=unseal` |
| `key not found` | Wrong `key_name` in config | Must match the key created in Step 1.2 (`unseal-key`) |
| `token expired` | Token TTL ran out | Create a new token or use `-period=` for renewable tokens |
| Seal Type still `shamir` | Vault not restarted after config change | `sudo systemctl restart vault` |
| Target won't unseal after restart | Transit cluster is down | Transit cluster **must** be running and unsealed |

---

## Official References

- [Vault Transit Secrets Engine](https://www.vaultproject.io/docs/secrets/transit)
- [Vault Transit Seal Configuration](https://www.vaultproject.io/docs/configuration/seal/transit)
- [Vault Raft Storage Backend](https://www.vaultproject.io/docs/configuration/storage/raft)
