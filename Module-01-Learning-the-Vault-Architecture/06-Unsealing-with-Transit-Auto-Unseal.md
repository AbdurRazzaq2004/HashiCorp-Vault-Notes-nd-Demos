# Unsealing with Transit Auto Unseal

> **Module 1 — Learning the Vault Architecture**
> Topic: Using another Vault cluster's Transit engine to auto-unseal — keeping everything in the Vault ecosystem.

---

## Overview

We've now seen two unseal methods:

| Method | Who handles the master key? |
|--------|---------------------------|
| **Shamir** (topic 03-04) | Humans hold key shards |
| **Cloud Auto Unseal** (topic 05) | Cloud KMS (AWS/Azure/GCP) |

But what if you **don't want to depend on a cloud provider** for your unsealing? What if you want to keep everything **within Vault's own ecosystem**?

That's where **Transit Auto Unseal** comes in — one Vault cluster unseals another.

---

## 1. The Concept

You run **two (or more) Vault clusters**:

```
┌─────────────────────────────────────────────────────────────┐
│                    TRANSIT VAULT CLUSTER                      │
│              (The "Unsealer" / "Key Holder")                 │
│                                                              │
│   ┌─────────────────────────────────────┐                    │
│   │    Transit Secrets Engine           │                    │
│   │    (holds the encryption key)       │                    │
│   └──────────────┬──────────────────────┘                    │
│                  │                                            │
└──────────────────┼───────────────────────────────────────────┘
                   │
         Encrypts/Decrypts master keys
                   │
      ┌────────────┼────────────┐
      │            │            │
      ▼            ▼            ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Vault    │ │ Vault    │ │ Vault    │
│ Cluster  │ │ Cluster  │ │ Cluster  │
│ A        │ │ B        │ │ C        │
│ (auto-   │ │ (auto-   │ │ (auto-   │
│  unseals)│ │  unseals)│ │  unseals)│
└──────────┘ └──────────┘ └──────────┘
     Dependent / Child Clusters
```

### How it works:

1. The **Transit Vault cluster** runs the Transit Secrets Engine with an encryption key.
2. Each **dependent cluster** stores its master key encrypted by that Transit key.
3. On startup, each dependent cluster calls the Transit cluster: *"Please decrypt my master key."*
4. Transit cluster decrypts it and sends it back.
5. Dependent cluster uses the master key to unlock — **auto unsealed!**

### Real-World Analogy

Think of it like a **corporate headquarters** that holds the master keycard:

- HQ (Transit Vault) has the master keycard reader.
- Branch offices (dependent clusters) need to call HQ every morning to get their doors unlocked.
- If HQ goes down, no branch office can open.

---

## 2. Transit Auto Unseal vs Cloud Auto Unseal

| Aspect | Cloud Auto Unseal | Transit Auto Unseal |
|--------|-------------------|---------------------|
| **External dependency** | Cloud provider (AWS/Azure/GCP) | Another Vault cluster |
| **Cloud lock-in** | Yes | No — pure Vault |
| **Setup complexity** | Lower (just add KMS ARN) | Higher (need a whole separate Vault cluster) |
| **Key rotation** | Managed by cloud | Managed by you (Transit engine supports it) |
| **Best for** | Cloud-native environments | Multi-cluster Vault, on-prem, no cloud dependency |
| **Open source?** | Yes (since v1.0) | Yes (since v1.0) |

---

## 3. Key Features

| Feature | Details |
|---------|---------|
| **Dedicated Transit Engine** | Uses a separate Vault cluster's Transit Secrets Engine to protect the master key |
| **Key Rotation Support** | Rotate the unseal encryption key regularly for compliance |
| **Open Source & Enterprise** | Works in both editions — no extra plugins needed |
| **Chaining** | You can chain: Cluster A unseals Cluster B, Cluster B unseals Cluster C |
| **HA Requirement** | The Transit cluster **MUST** be highly available — if it goes down, nothing can unseal |

### The Critical Risk

```
⚠️  If the Transit Vault cluster goes down:
    → All dependent clusters CANNOT unseal on restart
    → This is a single point of failure
    → ALWAYS run the Transit cluster in HA mode with a resilient storage backend
```

---

## 4. Configuration

Add a `seal "transit"` stanza to the **dependent cluster's** config file:

```hcl
seal "transit" {
  # ─── Connection to the Transit Vault cluster ───
  address         = "https://vault-transit.example.com:8200"
  token           = "s.Qf1s5zigZ4OX6akYjQXJC1jY"
  disable_renewal = "false"

  # ─── Which Transit key to use ───
  key_name        = "unseal-key"
  mount_path      = "transit/"
  namespace       = "ns1/"          # If using namespaces (Enterprise)

  # ─── TLS for secure communication ───
  tls_ca_cert     = "/etc/vault/ca_cert.pem"
  tls_client_cert = "/etc/vault/client_cert.pem"
  tls_client_key  = "/etc/vault/client_key.pem"
  tls_server_name = "vault"
  tls_skip_verify = "false"
}
```

### Breaking Down Each Field

| Field | What It Does |
|-------|-------------|
| `address` | URL of the Transit Vault cluster |
| `token` | Vault token with permission to use the Transit engine |
| `disable_renewal` | Whether to auto-renew the token (keep `false` to auto-renew) |
| `key_name` | Name of the Transit encryption key used for sealing |
| `mount_path` | Where the Transit engine is mounted (default: `transit/`) |
| `namespace` | Vault namespace (Enterprise feature, omit for OSS) |
| `tls_*` | TLS certificates for secure cluster-to-cluster communication |

---

## 5. Full Architecture — What Goes Where

### On the Transit Vault Cluster (the unsealer):

```bash
# 1. Enable Transit secrets engine
vault secrets enable transit

# 2. Create an encryption key for unsealing
vault write -f transit/keys/unseal-key

# 3. Create a policy that allows encrypt/decrypt on this key
vault policy write unseal-policy - <<EOF
path "transit/encrypt/unseal-key" {
  capabilities = ["update"]
}
path "transit/decrypt/unseal-key" {
  capabilities = ["update"]
}
EOF

# 4. Create a token with this policy (for the dependent clusters to use)
vault token create -orphan -policy="unseal-policy" -period=24h
```

### On each Dependent Cluster:

```hcl
# vault.hcl — add the seal stanza pointing to Transit cluster
seal "transit" {
  address    = "https://vault-transit.example.com:8200"
  token      = "<token-from-step-4-above>"
  key_name   = "unseal-key"
  mount_path = "transit/"
}
```

Then initialize and the dependent cluster will auto-unseal through Transit.

---

## 6. Chaining Clusters

You can chain auto-unseal across multiple clusters:

```
Transit Cluster (Shamir — manually unsealed)
       │
       ├──→ Cluster A (auto-unseals from Transit)
       │         │
       │         ├──→ Cluster D (auto-unseals from A)
       │         └──→ Cluster E (auto-unseals from A)
       │
       ├──→ Cluster B (auto-unseals from Transit)
       │
       └──→ Cluster C (auto-unseals from Transit)
```

> The "root" Transit cluster is typically unsealed manually with Shamir or via Cloud KMS. Everything downstream auto-unseals from it.

---

## 7. Comparison: All Three Unseal Methods

| | Shamir | Cloud Auto Unseal | Transit Auto Unseal |
|---|--------|-------------------|---------------------|
| **Master key held by** | Human shards | Cloud KMS | Another Vault cluster |
| **Human intervention** | Every restart | None | None |
| **External dependency** | None (humans) | Cloud provider | Transit Vault cluster |
| **Cloud lock-in** | No | Yes | No |
| **Setup effort** | Low | Medium | High |
| **Key rotation** | `vault operator rekey` | Cloud-managed | Transit engine rotation |
| **Best for** | Dev, small teams | Cloud production | Multi-cluster, on-prem |
| **Single point of failure** | Humans unavailable | Cloud KMS outage | Transit cluster down |
| **Open source?** | Yes | Yes (v1.0+) | Yes (v1.0+) |

---

## Quick Recap

```
Transit Auto Unseal = Use Vault to unseal Vault

┌─────────────────┐                    ┌──────────────────┐
│  Dependent Vault │ ── "decrypt my ──>│  Transit Vault   │
│  (needs unseal)  │     master key"   │  (unsealer)      │
│                  │ <── here you go ──│  Transit Engine   │
│  Now unsealed!   │                   │  + Encryption Key │
└─────────────────┘                    └──────────────────┘

No cloud KMS. No humans. Pure Vault-to-Vault.
```

---

## Key Exam Tips

1. **Transit Auto Unseal uses a SEPARATE Vault cluster's Transit Secrets Engine** — not the same cluster.
2. **The Transit cluster is a single point of failure** — MUST be highly available.
3. **Available in open source since v1.0** — not Enterprise-only.
4. **Key rotation is supported** via the Transit engine's built-in rotation.
5. **Chaining is possible** — Cluster A unseals B, B unseals C, etc.
6. **`seal "transit"` stanza** in the dependent cluster's config — needs `address`, `token`, `key_name`.
7. **If Transit cluster is down**, dependent clusters **cannot unseal on restart**.
8. **Recovery keys** are generated (not unseal keys), same as Cloud Auto Unseal.

---

## Official References

- [Vault Transit Secrets Engine](https://www.vaultproject.io/docs/secrets/transit)
- [Vault Seal/Unseal Concepts](https://www.vaultproject.io/docs/concepts/seal)
- [Vault Transit Seal Configuration](https://www.vaultproject.io/docs/configuration/seal/transit)

---

> **Labs/Demos** — Coming next! We'll set up a Transit cluster and configure a dependent cluster to auto-unseal from it.
