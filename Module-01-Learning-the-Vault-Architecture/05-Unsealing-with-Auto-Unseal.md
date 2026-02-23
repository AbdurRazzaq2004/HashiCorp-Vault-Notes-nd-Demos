# Unsealing with Auto Unseal

> **Module 1 — Learning the Vault Architecture**
> Topic: How to eliminate manual unsealing by using an external KMS or HSM.

---

## Overview

In topics 03 and 04 we learned about **Shamir's Secret Sharing** — the default unseal method where humans manually provide key shards every time Vault starts. That's fine for small setups, but imagine:

- Vault running in **Kubernetes** with auto-scaling pods
- A cluster of **5 Vault nodes** that restart after a rolling update
- **3 AM incident** — Vault restarted and nobody is awake to unseal it

Manually typing key shards every time? **Not practical.**

That's where **Auto Unseal** comes in.

---

## 1. What is Auto Unseal?

Auto Unseal delegates the master key protection to an **external service** instead of splitting it into human-held shards.

### Shamir (Manual) vs Auto Unseal

```
SHAMIR (Manual)                          AUTO UNSEAL
────────────────                         ────────────────

Master Key                               Master Key
    │                                        │
    ▼                                        ▼
Split into 5 shards                      Encrypted by Cloud KMS
    │                                        │
    ▼                                        ▼
Given to 5 humans                        Stored in storage backend
    │                                        │
    ▼                                        ▼
On restart: 3 humans                     On restart: Vault calls
must type their shards                   KMS to decrypt automatically
    │                                        │
    ▼                                        ▼
Vault unsealed                           Vault unsealed
(minutes/hours later)                    (seconds — no humans needed)
```

### The Simple Version

| Aspect | Shamir | Auto Unseal |
|--------|--------|-------------|
| Master key protected by | Human-held shards | External KMS/HSM |
| Human intervention on restart? | Yes (need T of N people) | No (fully automatic) |
| Speed of unseal | Minutes to hours | Seconds |
| Best for | Small teams, air-gapped environments | Production, cloud-native, auto-scaling |

---

## 2. How Auto Unseal Works (Step by Step)

Here's what happens under the hood:

### During Initialization

```
┌──────────────────────────────────────────────────────────┐
│  vault operator init                                      │
│                                                          │
│  1. Vault generates the Master Key                       │
│  2. Instead of splitting into shards, Vault sends        │
│     the Master Key to the external KMS                   │
│  3. KMS encrypts the Master Key and returns it           │
│  4. Vault stores the ENCRYPTED Master Key in the         │
│     storage backend (Consul, Raft, etc.)                 │
│  5. Vault generates Recovery Keys (similar to shards     │
│     but CANNOT unseal — used for rekeying & recovery)    │
└──────────────────────────────────────────────────────────┘
```

### During Every Startup / Restart

```
┌──────────┐       ┌───────────────┐       ┌─────────────┐
│  Vault   │──1──> │Storage Backend│       │             │
│ (starts) │<──2── │ (Consul/Raft) │       │  Cloud KMS  │
│          │       └───────────────┘       │(AWS/Azure/  │
│          │                               │  GCP)       │
│          │──3── Request: Decrypt this ──>│             │
│          │<──4── Response: Here you go ──│             │
│          │       └───────────────────────┘             │
│          │                                             │
│  5. Master Key decrypted                               │
│  6. Encryption Key unlocked                            │
│  7. Vault is UNSEALED — automatically!                 │
└──────────┘
```

**Step by step:**
1. Vault reads the **encrypted** master key from storage backend
2. Storage returns the encrypted blob
3. Vault sends the encrypted master key to the cloud KMS: *"Please decrypt this"*
4. KMS decrypts it and returns the plaintext master key
5. Vault uses the master key to decrypt the **data encryption key (DEK)**
6. DEK is loaded into memory
7. Vault is unsealed — **zero human involvement**

---

## 3. Supported Auto Unseal Providers

| Provider | Config Stanza | Use When |
|----------|--------------|----------|
| **AWS KMS** | `seal "awskms"` | Running Vault on AWS |
| **Azure Key Vault** | `seal "azurekeyvault"` | Running Vault on Azure |
| **Google Cloud KMS** | `seal "gcpckms"` | Running Vault on GCP |
| **AliCloud KMS** | `seal "alicloudkms"` | Running Vault on Alibaba Cloud |
| **On-prem HSM (PKCS#11)** | `seal "pkcs11"` | Air-gapped / on-premise environments (Enterprise only) |
| **Transit (another Vault)** | `seal "transit"` | Using a separate Vault cluster as your KMS |

> Auto Unseal has been available in **open-source Vault since v1.0**. You don't need Enterprise for cloud KMS options.

---

## 4. Configuration Example: AWS KMS

Here's a complete Vault config with Auto Unseal enabled:

```hcl
# Storage Backend
storage "consul" {
  address = "127.0.0.1:8500"
  path    = "vault/"
}

# Listener
listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
}

# ────────────────────────────────────────
# Auto Unseal — This is the magic stanza
# ────────────────────────────────────────
seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/abcd-1234"
}

api_addr = "https://<VAULT_IP>:8200"
ui       = true
```

### What each field means:

| Field | Description |
|-------|-------------|
| `seal "awskms"` | Tells Vault to use AWS KMS for auto unseal |
| `region` | AWS region where your KMS key lives |
| `kms_key_id` | The ARN or key ID of the KMS key used to encrypt/decrypt the master key |

### For other providers:

**Azure Key Vault:**
```hcl
seal "azurekeyvault" {
  tenant_id      = "your-tenant-id"
  client_id      = "your-client-id"
  client_secret  = "your-client-secret"
  vault_name     = "my-vault-kv"
  key_name       = "vault-unseal-key"
}
```

**GCP Cloud KMS:**
```hcl
seal "gcpckms" {
  credentials = "/path/to/service-account.json"
  project     = "my-project"
  region      = "global"
  key_ring    = "vault-keyring"
  crypto_key  = "vault-unseal-key"
}
```

---

## 5. Recovery Keys (Important Concept!)

When using Auto Unseal, Vault no longer generates **unseal keys**. Instead, it generates **recovery keys**.

### What's the difference?

| | Unseal Keys (Shamir) | Recovery Keys (Auto Unseal) |
|--|---------------------|---------------------------|
| **Can unseal Vault?** | Yes | No |
| **Generated during** | `vault operator init` (Shamir mode) | `vault operator init` (Auto Unseal mode) |
| **Used for** | Unsealing Vault | Rekeying, generating root tokens, certain admin operations |
| **Split with Shamir?** | Yes (N shares, T threshold) | Yes (same N/T concept) |

> Recovery keys are like **emergency admin keys** — they can't open the vault, but they can change the locks.

---

## 6. Shamir vs Auto Unseal — When to Use What

| Scenario | Recommendation |
|----------|---------------|
| Development / learning | Shamir (simpler to set up) |
| Small team, few restarts | Shamir (manageable) |
| Production on cloud | **Auto Unseal with cloud KMS** |
| Kubernetes / auto-scaling | **Auto Unseal** (pods restart frequently) |
| Air-gapped / on-prem | Auto Unseal with HSM (PKCS#11) or Shamir |
| Multi-cluster Vault | Transit Auto Unseal |

---

## 7. Security Considerations

| Concern | Mitigation |
|---------|-----------|
| KMS key gets deleted | **Vault can NEVER unseal again.** Protect the KMS key with deletion protection and access policies. |
| KMS credentials leaked | Attacker could decrypt the master key. Use IAM roles instead of static credentials. |
| Cloud provider outage | Vault can't unseal until KMS is back. Consider multi-region KMS keys. |
| `kms_key_id` in source control | Never commit credentials or key IDs. Use environment variables or a secrets workflow. |

```bash
# Better approach — use environment variables
export VAULT_AWSKMS_SEAL_KEY_ID="arn:aws:kms:us-east-1:123456789012:key/abcd-1234"
```

---

## Quick Recap

```
Auto Unseal = Let a cloud KMS handle the master key instead of humans

┌─────────────┐     ┌──────────┐     ┌────────────┐
│ Vault starts │────>│ Storage  │────>│ Cloud KMS  │
│ (sealed)    │     │ Backend  │     │ decrypts   │
│             │<────│ returns  │<────│ master key │
│ (unsealed!) │     │ enc. key │     │            │
└─────────────┘     └──────────┘     └────────────┘

No humans needed. Happens in seconds.
```

---

## Key Exam Tips

1. **Auto Unseal uses an external KMS/HSM** — not human-held shards.
2. **Available in open-source Vault since v1.0** — not just Enterprise.
3. **`seal` stanza in vault.hcl** — this is how you configure it (awskms, azurekeyvault, gcpckms, etc.).
4. **Recovery keys ≠ Unseal keys** — recovery keys CANNOT unseal Vault; they're for admin operations only.
5. **If the KMS key is deleted, Vault is permanently locked** — protect it with deletion policies.
6. **No `seal` stanza = Shamir mode** (the default).
7. **Transit Auto Unseal** = using another Vault cluster as the KMS.

---

## Official References

- [Vault Auto Unseal Overview](https://www.vaultproject.io/docs/concepts/seal)
- [AWS KMS Seal](https://www.vaultproject.io/docs/configuration/seal/awskms)
- [Azure Key Vault Seal](https://www.vaultproject.io/docs/configuration/seal/azurekeyvault)
- [GCP Cloud KMS Seal](https://www.vaultproject.io/docs/configuration/seal/gcpckms)
- [PKCS#11 HSM Seal](https://www.vaultproject.io/docs/configuration/seal/pkcs11)

---

> **Labs/Demos** — Coming soon! We'll configure Auto Unseal with AWS KMS and watch Vault unseal itself.
