# Vault Initialization

> **Module 1 — Learning the Vault Architecture**
> Topic: The one-time setup process that brings Vault to life.

---

## Overview

Initialization is Vault's **birth moment** — it's a **one-time operation** that happens only once per cluster, ever. Think of it like setting up a brand-new safe for the first time:

1. The safe **generates its own combination** (master key)
2. It **creates the internal lock mechanism** (data encryption key)
3. It **splits the combination into pieces** for trusted people (key shards)
4. It **gives you a temporary admin badge** (root token)

After this, Vault is ready — but still sealed. You need to unseal before using it.

```
   Brand new Vault            vault operator init              Initialized
   ┌──────────┐              ─────────────────>               ┌──────────┐
   │Not init'd│                                               │Init'd but│
   │Not sealed│              Generates:                       │  SEALED  │
   │(nothing  │              • Master key                     │          │
   │ exists)  │              • Encryption key                 │ Ready to │
   └──────────┘              • Key shards / Recovery keys     │ unseal   │
                             • Root token                     └──────────┘
```

---

## 1. What Happens During Initialization

When you run `vault operator init`, four things happen in order:

### Step-by-Step Breakdown

```
┌─────────────────────────────────────────────────────────────────┐
│  Step 1: MASTER KEY generated                                    │
│  └─ This is the "key to all keys"                               │
│                                                                  │
│  Step 2: DATA ENCRYPTION KEY (DEK) generated                    │
│  └─ This is the key that actually encrypts/decrypts your data   │
│  └─ The Master Key encrypts the DEK                             │
│                                                                  │
│  Step 3: MASTER KEY is split/protected                          │
│  └─ Shamir mode → split into N shards (default: 5)             │
│  └─ Auto Unseal → encrypted by KMS, recovery keys generated    │
│                                                                  │
│  Step 4: ROOT TOKEN issued                                       │
│  └─ Your first (and most powerful) authentication credential    │
└─────────────────────────────────────────────────────────────────┘
```

### The Key Hierarchy

```
Master Key
    │
    ├── Protects → Data Encryption Key (DEK)
    │                   │
    │                   ├── Encrypts → All secrets
    │                   ├── Encrypts → Policies
    │                   ├── Encrypts → Configuration
    │                   └── Encrypts → Everything in storage
    │
    └── Is protected by:
        ├── Shamir shards (manual unseal)
        ├── Cloud KMS (auto unseal)
        └── Transit Vault (transit auto unseal)
```

> **Key point:** Vault uses TWO levels of encryption. The master key doesn't directly encrypt your data — it encrypts the DEK, which encrypts the data. This layered approach is called **envelope encryption**.

---

## 2. Key Shares, Thresholds, and Recovery Keys

### Default Values

| Setting | Default | What It Means |
|---------|---------|--------------|
| `-key-shares` | 5 | Master key is split into 5 pieces |
| `-key-threshold` | 3 | Need any 3 of those 5 to unseal |

### Customizing During Init

```bash
# Default: 5 shares, threshold 3
vault operator init

# Custom: 10 shares, threshold 6
vault operator init \
  -key-shares=10 \
  -key-threshold=6

# Minimal: 1 share, threshold 1 (dev/testing only!)
vault operator init \
  -key-shares=1 \
  -key-threshold=1
```

### Shamir Mode vs Auto Unseal Mode

| | Shamir Mode | Auto Unseal Mode |
|--|-------------|-----------------|
| **What init generates** | Unseal Key Shards | Recovery Keys |
| **Can these unseal Vault?** | Yes | No |
| **How many?** | N shards, T threshold | N recovery keys, T threshold |
| **Used for** | Unsealing on every restart | Rekeying, generating root tokens, recovery |

```bash
# Shamir init output:
Unseal Key 1: xxxx
Unseal Key 2: xxxx
...
Initial Root Token: hvs.xxxx

# Auto Unseal init output:
Recovery Key 1: xxxx
Recovery Key 2: xxxx
...
Initial Root Token: hvs.xxxx
```

---

## 3. Encrypting Keys with PGP

For extra security, you can encrypt each shard/recovery key so only the intended person can read it.

### How It Works

```
During Init:
                     PGP Public Keys
                     ┌──────────────┐
  Shard 1 ──encrypt──│ alice.pem    │──→ Encrypted Shard 1 (only Alice can decrypt)
  Shard 2 ──encrypt──│ bob.pem      │──→ Encrypted Shard 2 (only Bob can decrypt)
  Shard 3 ──encrypt──│ carol.pem    │──→ Encrypted Shard 3 (only Carol can decrypt)
                     └──────────────┘
```

### Command

```bash
vault operator init \
  -key-shares=3 \
  -key-threshold=2 \
  -pgp-keys="alice.pem,bob.pem,carol.pem"
```

Each person decrypts their shard with their private PGP key:

```bash
echo "<encrypted-shard>" | base64 -d | gpg --decrypt
```

> You can also encrypt the root token with `-root-token-pgp-key="admin.pem"`.

---

## 4. Three Ways to Initialize

### Method 1: CLI (Most Common)

```bash
# Simple init
vault operator init

# Custom init
vault operator init \
  -key-shares=7 \
  -key-threshold=4 \
  -pgp-keys="team1.pem,team2.pem,team3.pem,team4.pem,team5.pem,team6.pem,team7.pem"
```

**Best for:** Manual setup, learning, quick cluster bootstrapping.

### Method 2: API (For Automation)

```bash
curl \
  --request PUT \
  --data '{"secret_shares": 5, "secret_threshold": 3}' \
  https://vault.example.com:8200/v1/sys/init
```

**Best for:** CI/CD pipelines, infrastructure-as-code, automated deployments.

### Method 3: Web UI

1. Open `https://<vault-ip>:8200/ui` in your browser
2. You'll see the initialization screen
3. Enter key shares and threshold
4. Click "Initialize"
5. Download/save your keys and root token

**Best for:** Visual setup, demos, teams unfamiliar with CLI.

---

## 5. Post-Initialization Steps

After init, Vault is **initialized but sealed**. Here's your checklist:

```
✅ Step 1: Initialization (vault operator init)
      │
      ▼
✅ Step 2: Securely store/distribute keys
      │    • Give each shard to a different person
      │    • Store root token separately
      │
      ▼
✅ Step 3: Unseal Vault
      │    • Shamir: vault operator unseal (x3)
      │    • Auto Unseal: happens automatically
      │
      ▼
✅ Step 4: Authenticate
      │    vault login <root-token>
      │
      ▼
✅ Step 5: Initial Configuration
      │    • Enable auth methods
      │    • Create policies
      │    • Enable secrets engines
      │    • Enable audit devices
      │
      ▼
✅ Step 6: Revoke Root Token
         vault token revoke <root-token>
```

---

## 6. Important Rules About Initialization

| Rule | Details |
|------|---------|
| **One-time only** | Init happens ONCE per cluster. NEVER re-initialize. |
| **Cannot undo** | Once initialized, you can't "un-initialize." You'd have to wipe storage and start fresh. |
| **After backup restore** | Do NOT re-initialize. Just unseal with the original keys. |
| **After node failure** | New nodes joining the cluster do NOT need initialization. Only the cluster does. |
| **After migration** | If you migrate storage backends, you don't re-initialize. |
| **Keys shown once** | The unseal keys and root token are displayed ONLY during init. Save them immediately. |

### Common Mistake

```
❌ WRONG: Vault crashed → restore from backup → vault operator init
   This would create NEW keys and wipe existing data!

✅ RIGHT: Vault crashed → restore from backup → vault operator unseal
   Use the ORIGINAL keys from the first initialization.
```

---

## Quick Recap

```
vault operator init
        │
        ├── Generates Master Key
        │     └── Encrypts the Data Encryption Key (DEK)
        │
        ├── Splits Master Key (Shamir) or Encrypts it (Auto Unseal)
        │     ├── Shamir → N unseal key shards
        │     └── Auto Unseal → N recovery keys
        │
        ├── Issues Root Token
        │     └── All-powerful, use sparingly, revoke after setup
        │
        └── Writes to storage backend (ONE TIME ONLY)
```

---

## Key Exam Tips

1. **Initialization is a ONE-TIME operation** — never re-initialize after restore or failure.
2. **`vault operator init`** generates: master key, DEK, key shards (or recovery keys), and root token.
3. **Default: 5 shares, threshold 3** — customizable with `-key-shares` and `-key-threshold`.
4. **Envelope encryption** — Master key encrypts DEK, DEK encrypts data (two layers).
5. **PGP encryption** for shards — pass `-pgp-keys` during init.
6. **Three init methods** — CLI, API (`PUT /v1/sys/init`), and Web UI.
7. **After init, Vault is still SEALED** — unsealing is a separate step (unless auto unseal).
8. **Keys are shown ONLY ONCE** — if you lose them and don't have enough shards, data is gone.
9. **Root token should be revoked** after initial setup — use proper auth methods instead.

---

## Official References

- [Vault Initialization API](https://www.vaultproject.io/api-docs/system/init)
- [Vault Operator Init Command](https://www.vaultproject.io/docs/commands/operator/init)
- [Shamir's Secret Sharing](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing)
- [Auto Unseal with AWS KMS](https://www.vaultproject.io/docs/configuration/seal/awskms)

---

> **Labs/Demos** — Already covered in Lab 01 (Shamir init) and Lab 02 (Auto Unseal init). Review those for hands-on practice!
