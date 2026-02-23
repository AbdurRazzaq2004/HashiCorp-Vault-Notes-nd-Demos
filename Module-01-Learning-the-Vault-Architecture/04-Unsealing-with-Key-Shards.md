# Unsealing with Key Shards

> **Module 1 — Learning the Vault Architecture**
> Topic: Deep dive into Shamir's Secret Sharing and the hands-on unseal process.

---

## Overview

In topic 03 we learned **what** sealing/unsealing is and the three methods available. This topic goes deeper into the **default method** — Shamir's Secret Sharing — and walks through exactly how it works in practice.

---

## 1. How Shamir's Secret Sharing Works

### The Problem It Solves

You have ONE master key. If one person holds it:
- They could go rogue.
- They could lose it.
- They become a single point of failure.

### The Solution

**Split the key into pieces (shards) and distribute them to different people.** No single person can reconstruct the key alone — you need a minimum number of people (threshold) to come together.

```
                       vault operator init
                              │
                              ▼
                     ┌─────────────────┐
                     │   Master Key    │
                     │  (generated)    │
                     └────────┬────────┘
                              │
                    Shamir's Algorithm splits it
                              │
             ┌────────┬───────┼───────┬────────┐
             ▼        ▼       ▼       ▼        ▼
          ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
          │Key 1 │ │Key 2 │ │Key 3 │ │Key 4 │ │Key 5 │
          │      │ │      │ │      │ │      │ │      │
          │Alice │ │ Bob  │ │Carol │ │Dave  │ │ Eve  │
          └──────┘ └──────┘ └──────┘ └──────┘ └──────┘

          Total Shares: 5       Threshold: 3

          ✅ Alice + Bob + Carol = Can unseal
          ✅ Bob + Dave + Eve   = Can unseal
          ✅ Any 3 of 5         = Can unseal
          ❌ Alice + Bob        = NOT enough (only 2)
          ❌ Any single person  = NOT enough
```

### Default Values

| Setting | Default | Meaning |
|---------|---------|---------|
| **Key Shares** | 5 | Master key is split into 5 pieces |
| **Key Threshold** | 3 | Need at least 3 pieces to reconstruct |

You can customize these during initialization:

```bash
vault operator init -key-shares=7 -key-threshold=4
```

---

## 2. Distributing Key Shards

After running `vault operator init`, Vault outputs:

```
Unseal Key 1: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Unseal Key 2: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Unseal Key 3: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Unseal Key 4: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Unseal Key 5: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Initial Root Token: hvs.xxxxxxxxxxxxxxxxxxxx
```

**This is the ONLY time you'll see these keys.** Vault does NOT store them.

### What to do next:

```
Shard 1 ──→ Give to Alice   (e.g., Security Lead)
Shard 2 ──→ Give to Bob     (e.g., VP Engineering)
Shard 3 ──→ Give to Carol   (e.g., DevOps Manager)
Shard 4 ──→ Give to Dave    (e.g., CTO)
Shard 5 ──→ Give to Eve     (e.g., Compliance Officer)

Root Token ──→ Use for INITIAL setup only, then REVOKE it!
```

Each person stores their shard securely. **No one person should hold multiple shards** (defeats the purpose).

---

## 3. The Unsealing Process — Step by Step

### Step 1: Check Vault Status (Sealed)

```bash
$ vault status

Key                     Value
---                     -----
Seal Type               shamir
Sealed                  true        # ← Locked!
Total Shares            5
Threshold               3
Unseal Progress         0/3         # ← No shards submitted yet
```

### Step 2: Submit Shard 1 (Alice provides her key)

```bash
$ vault operator unseal <alice-shard>

Key                     Value
---                     -----
Seal Type               shamir
Sealed                  true        # ← Still sealed
Unseal Progress         1/3         # ← 1 of 3 done
```

### Step 3: Submit Shard 2 (Bob provides his key)

```bash
$ vault operator unseal <bob-shard>

Key                     Value
---                     -----
Seal Type               shamir
Sealed                  true        # ← Still sealed
Unseal Progress         2/3         # ← 2 of 3 done
```

### Step 4: Submit Shard 3 (Carol provides her key)

```bash
$ vault operator unseal <carol-shard>

Key                     Value
---                     -----
Seal Type               shamir
Sealed                  false       # ← UNSEALED! 🎉
Total Shares            5
Threshold               3
Version                 1.7.0
Storage Type            consul
Cluster Name            vault-cluster
Cluster ID              xxx-xxx-xxx-xxx
HA Enabled              true
```

### What Just Happened?

```
Alice's Shard ──┐
                ├──→ Shamir combines them ──→ Master Key ──→ Decrypts Encryption Key
Bob's Shard   ──┤                                               │
                │                                               ▼
Carol's Shard ──┘                                    Vault is UNSEALED
                                                  (encryption key now in memory)
```

### Important Notes

- **Order doesn't matter** — Carol could go first, then Alice, then Bob. Same result.
- **Shards are NEVER logged** — Vault tracks unseal progress but never records the actual shard values.
- **Dave and Eve weren't needed** — Any 3 of 5 is enough. The other 2 are backup in case someone is unavailable.

---

## 4. What If Something Goes Wrong?

| Scenario | What Happens |
|----------|-------------|
| Wrong shard submitted | Vault rejects it; progress stays the same |
| Vault restarts mid-unseal | Progress resets to 0. Start over. |
| Lost shards (but still have threshold) | No problem — e.g., lost 2 of 5, still have 3 ≥ threshold |
| Lost shards (below threshold) | **Disaster.** Cannot unseal. Data is permanently inaccessible. |
| Someone provides the same shard twice | Only counts once. |

> This is why **backup and shard management** are critical.

---

## 5. Key Shard Best Practices

| Practice | Details |
|----------|---------|
| **PGP Encrypt Shards** | During init, provide each custodian's PGP public key. Each shard is encrypted so only the intended person can read it. |
| **Offline Storage** | Store shards on encrypted USB drives, hardware security modules, or in physical safes. NOT in cloud storage or email. |
| **Access Controls** | Only the assigned custodian should have access to their shard — physically and digitally. |
| **Custodian Roster** | Maintain a current list of who holds which shard number. Update when people leave the company. |
| **Reachability** | Ensure at least `threshold` number of custodians are reachable at any given time (consider time zones, vacations). |
| **Rekeying** | If a custodian leaves, use `vault operator rekey` to generate new shards and distribute to new people. |

### PGP-Encrypted Initialization

```bash
vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -pgp-keys="alice.asc,bob.asc,carol.asc,dave.asc,eve.asc"
```

Each person decrypts their shard with their private PGP key:

```bash
echo "<encrypted-shard>" | base64 -d | gpg --decrypt
```

---

## 6. Visualizing the Full Lifecycle

```
┌──────────────────────────────────────────────────────────┐
│                    INITIALIZATION                         │
│  vault operator init                                      │
│  → Master key generated                                   │
│  → Split into 5 shards (Shamir)                          │
│  → Root token created                                     │
│  → Vault starts SEALED                                    │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                    UNSEALING                               │
│  vault operator unseal (3 times with different shards)    │
│  → Master key reconstructed                               │
│  → Encryption key decrypted into memory                   │
│  → Vault becomes UNSEALED                                 │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                   OPERATIONAL                              │
│  Vault serves requests normally                           │
│  → Read/write secrets                                     │
│  → Authenticate users                                     │
│  → Generate dynamic credentials                           │
└──────────────────────┬───────────────────────────────────┘
                       │
            vault operator seal  OR  restart
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                    SEALED AGAIN                            │
│  Encryption key removed from memory                       │
│  → Must unseal again to resume operations                 │
└──────────────────────────────────────────────────────────┘
```

---

## Key Exam Tips

1. **Default: 5 shares, threshold of 3** — know these numbers.
2. **`vault operator init`** creates the shards — this is a **one-time operation**.
3. **`vault operator unseal`** is run multiple times (once per shard) until threshold is met.
4. **Shards are never logged** by Vault — confidentiality preserved.
5. **Order of shard submission doesn't matter.**
6. **Losing shards below threshold = data loss** — there's no recovery.
7. **`vault operator rekey`** — use this when you need to regenerate shards (e.g., custodian leaves).
8. **PGP encryption** of shards during init is a security best practice.

---

## Official References

- [Vault Operator Init](https://www.vaultproject.io/docs/commands/operator/init)
- [Vault Operator Unseal](https://www.vaultproject.io/docs/commands/operator/unseal)
- [Vault Operator Rekey](https://www.vaultproject.io/docs/commands/operator/rekey)
- [Shamir's Secret Sharing (Wikipedia)](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing)

---

> **Labs/Demos** — Coming soon! We'll initialize a Vault server and go through the full shard distribution and unseal process.
