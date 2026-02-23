# Seal and Unseal

> **Module 1 — Learning the Vault Architecture**
> Topic: How Vault protects data at rest using the seal/unseal mechanism.

---

## Overview

Imagine a **safe with a combination lock**. When you close the safe and spin the dial — that's **sealing**. The valuables are still inside, but nobody can access them without the combination. **Unsealing** is entering the combination to open the safe again.

Vault works the same way:

| State | What's Happening | Can You Access Secrets? |
|-------|-----------------|----------------------|
| **Sealed** | Encryption key is NOT in memory. Vault knows where data is but can't decrypt it. | No |
| **Unsealed** | Encryption key IS in memory. Vault can read/write secrets normally. | Yes |

**Every time Vault starts (or restarts), it starts SEALED.** You must unseal it before it can do anything useful.

---

## 1. The Sealed State

When Vault is sealed:

- It **knows** where data is stored (storage backend is configured).
- It **cannot decrypt** any of that data (no encryption key in memory).
- The **only things** you can do:
  - `vault status` — check if it's sealed/unsealed
  - `vault operator unseal` — provide unseal keys
- **Everything else is blocked** — no reading secrets, no writing, no generating tokens, nothing.

### What Happens During Unseal?

```
Sealed                            Unsealed
┌──────────┐                     ┌──────────┐
│          │  Provide enough     │          │
│  🔒 No   │  key shards         │  🔓 Yes  │
│  encrypt │ ──────────────────> │  encrypt │
│  key in  │  Master key is      │  key in  │
│  memory  │  reconstructed      │  memory  │
│          │                     │          │
└──────────┘                     └──────────┘
  Can't do                         Fully
  anything                       operational
```

Step by step:
1. Key shards are provided (one at a time, by different people).
2. Once enough shards are given (threshold met), the **master key** is reconstructed.
3. The master key **decrypts the encryption key** and loads it into memory.
4. Vault transitions to **unsealed** state and starts serving requests.

### What Happens During Seal?

The reverse — Vault **throws away the encryption key from memory**. Data is still on disk (encrypted), but Vault can no longer read it. A fresh unseal is required to resume.

---

## 2. Why Would You Seal Vault Manually?

Vault seals automatically on restart, but you can also **manually seal** it with:

```bash
vault operator seal
```

When would you do this? In **emergency situations**:

| Scenario | Why Seal? |
|----------|----------|
| **Unseal keys leaked** | Someone accidentally pushed keys to a public Git repo |
| **Key holder left the company** | A person with key shards is no longer trusted |
| **Network intrusion detected** | An attacker might be on your network |
| **Malware found on Vault nodes** | System integrity is compromised |

> Sealing is Vault's **panic button**. One command and everything locks down instantly.

---

## 3. Three Ways to Unseal Vault

### Method 1: Key Sharding (Shamir's Secret Sharing) — The Default

This is the **default and most common** method.

**How it works:**

When Vault is first initialized, it generates a **master key**. But instead of giving you one key, it **splits it into pieces** called **shards** using [Shamir's Secret Sharing](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing) algorithm.

```
                    Master Key
                   ┌──────────┐
                   │ XXXXXXXXX │
                   └─────┬─────┘
          Shamir's Secret Sharing
        ┌────────┬───┴───┬────────┬────────┐
        ▼        ▼       ▼        ▼        ▼
    ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
    │Shard1│ │Shard2│ │Shard3│ │Shard4│ │Shard5│
    │ Alice│ │  Bob │ │Carol │ │ Dave │ │  Eve │
    └──────┘ └──────┘ └──────┘ └──────┘ └──────┘
    
    Total Shares: 5    Threshold: 3
    
    Any 3 of these 5 people can unseal Vault.
    No single person (or even 2 people) can do it alone.
```

**Key terms:**
- **Total Shares (N)** — How many pieces the master key is split into (e.g., 5)
- **Threshold (T)** — How many pieces are needed to reconstruct it (e.g., 3)
- The order of providing shards **doesn't matter**

### Method 2: Cloud Auto Unseal

Instead of humans typing in key shards every time Vault restarts, you can delegate the unsealing to a **cloud KMS (Key Management Service)**.

| Cloud Provider | Service Used |
|---------------|-------------|
| AWS | AWS KMS |
| Azure | Azure Key Vault |
| GCP | GCP Cloud KMS |

**How it works:**

1. Vault's master key is encrypted by the cloud KMS.
2. On startup, Vault calls the cloud KMS to decrypt the master key.
3. Vault unseals itself automatically — no humans needed.

```hcl
# Example: Auto unseal with AWS KMS
seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "alias/vault-unseal-key"
}
```

**Best for:** Production environments where manual unsealing is impractical (auto-scaling, Kubernetes, etc.).

### Method 3: Transit Auto Unseal

Same idea as Cloud Auto Unseal, but instead of a cloud KMS, you use **another Vault cluster's Transit secrets engine** to protect the master key.

```
┌────────────────┐         ┌────────────────┐
│  Primary Vault │ ──────> │  Transit Vault  │
│  (needs unseal)│  asks   │  (already       │
│                │  for    │   unsealed)     │
│                │  decrypt│                 │
└────────────────┘         └────────────────┘
```

**Best for:** Organizations that want to keep everything within Vault's ecosystem (no cloud KMS dependency).

### Quick Comparison

| Method | Human Intervention? | Best For | Complexity |
|--------|-------------------|----------|------------|
| **Shamir Key Shards** | Yes — need T of N people | Small teams, high-security environments | Low |
| **Cloud Auto Unseal** | No — fully automated | Cloud-native production | Medium |
| **Transit Auto Unseal** | No — fully automated | Multi-cluster Vault setups | High |

---

## 4. Step-by-Step: Unsealing with Key Shards

### Check the status first:

```bash
$ vault status

Key                Value
---                -----
Seal Type          shamir
Sealed             true          # <-- Vault is sealed!
Total Shares       5
Threshold          3
Unseal Progress    0/3           # <-- 0 of 3 shards provided
```

### Provide shard 1 (from Alice):

```bash
$ vault operator unseal <shard-1>

Unseal Progress    1/3           # <-- 1 down, 2 to go
```

### Provide shard 2 (from Bob):

```bash
$ vault operator unseal <shard-2>

Unseal Progress    2/3           # <-- Almost there
```

### Provide shard 3 (from Carol):

```bash
$ vault operator unseal <shard-3>

Key                    Value
---                    -----
Seal Type              shamir
Sealed                 false          # <-- UNSEALED!
Total Shares           5
Threshold              3
Version                1.7.0
Storage Type           consul
Cluster Name           vault-cluster
HA Enabled             true
```

Vault is now **unsealed and fully operational**.

> The order doesn't matter — Carol could have gone first, then Alice, then Bob. Any 3 of 5 works.

---

## 5. Best Practices for Key Shards

| Practice | Why |
|----------|-----|
| **Separate custody** | Give each shard to a different person — no one person should hold enough shards to meet the threshold |
| **Encrypt shards with PGP** | During `vault operator init`, pass PGP public keys so each shard is encrypted for its intended recipient |
| **Store offline** | Keep shards out of online systems, cloud storage, and automated backups |
| **Balance the threshold** | Higher threshold = more secure but harder to unseal. Lower threshold = easier but riskier |
| **Document who holds what** | Know which shard belongs to whom, without storing the actual shard values together |

### Initializing with PGP Encryption

```bash
vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -pgp-keys="alice.asc,bob.asc,carol.asc,dave.asc,eve.asc"
```

Each person receives a shard encrypted with their PGP public key. Only they can decrypt their own shard.

---

## Quick Recap

```
Vault Starts → SEALED (always)
                  │
                  │  Provide key shards (Shamir)
                  │  OR Cloud KMS decrypts (Auto Unseal)
                  │  OR Transit Vault decrypts
                  │
                  ▼
              UNSEALED → Fully operational
                  │
                  │  vault operator seal (manual)
                  │  OR Vault restart
                  │
                  ▼
               SEALED → Back to locked state
```

---

## Key Exam Tips

1. **Vault ALWAYS starts sealed** — even after a restart.
2. **Sealed = can't do anything** except `vault status` and `vault operator unseal`.
3. **Shamir's Secret Sharing** is the DEFAULT unseal method — splits master key into N shards, needs T to reconstruct.
4. **Order of shards doesn't matter** — any T of N works.
5. **Cloud Auto Unseal** (AWS KMS, Azure Key Vault, GCP KMS) = no human intervention needed.
6. **Transit Auto Unseal** = use another Vault cluster to unseal.
7. **Sealing is an emergency operation** — immediately locks everything down.
8. **Master key ≠ Encryption key** — Master key decrypts the encryption key, which then decrypts the actual data.

---

## Official References

- [Vault Seal/Unseal Concepts](https://www.vaultproject.io/docs/concepts/seal)
- [Shamir's Secret Sharing (Wikipedia)](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing)
- [AWS KMS Auto Unseal](https://www.vaultproject.io/docs/configuration/seal/awskms)
- [Azure Key Vault Auto Unseal](https://www.vaultproject.io/docs/configuration/seal/azurekeyvault)
- [Transit Auto Unseal](https://www.vaultproject.io/docs/configuration/seal/transit)

---

> **Labs/Demos** — Coming soon! We'll initialize Vault, receive shards, and go through the full unseal process.
