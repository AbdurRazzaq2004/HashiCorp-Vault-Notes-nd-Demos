# Lab 02 — Auto Unseal with AWS KMS

> **Module 1 — Learning the Vault Architecture**
> Lab: Configure Vault to automatically unseal using AWS KMS — no more manual key shards!
>
> **Related Notes:** [05-Unsealing-with-Auto-Unseal](../Module-01-Learning-the-Vault-Architecture/05-Unsealing-with-Auto-Unseal.md)

---

## What You'll Do in This Lab

1. Check Vault's initial status (default Shamir seal)
2. Review the existing Vault configuration
3. Add the AWS KMS `seal` stanza to the config
4. Restart Vault and verify the seal type changed
5. Initialize Vault (get recovery keys instead of unseal keys)
6. Authenticate and use Vault
7. Restart Vault and confirm it auto-unseals

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| **EC2 Instance** | With Vault installed and running |
| **AWS KMS Key** | A Customer Managed Key (CMK) created in AWS KMS |
| **IAM Permissions** | The EC2 instance's IAM role must have `kms:Encrypt`, `kms:Decrypt`, `kms:DescribeKey` on the KMS key |

### Quick: Create a KMS Key in AWS Console

```
AWS Console → KMS → Customer managed keys → Create key
  → Symmetric → Encrypt and decrypt
  → Alias: "vault-unseal-key"
  → Copy the ARN (you'll need it in Step 3)
```

---

## Step 1: Check Current Vault Status

```bash
vault status
```

**Output:**

```
Key              Value
----             -----
Seal Type        shamir          ← Currently using Shamir (default)
Initialized      false
Sealed           true
Total Shares     0
Threshold        0
Unseal Progress  0/0
Unseal Nonce     n/a
Version          1.7.1
Storage Type     raft
HA Enabled       true
```

Vault is fresh — not initialized, using the default Shamir seal. We're going to change this to AWS KMS.

---

## Step 2: Review the Vault Configuration

```bash
cat /etc/vault.d/vault.hcl
```

**Current config (no `seal` stanza):**

```hcl
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "node-a-us-east-1"
  retry_join {
    auto_join = "provider=aws region=us-east-1 tag_key=vault tag_value=us-east-1"
  }
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_disable     = 1
}

api_addr     = "http://10.0.1.37:8200"
cluster_addr = "http://10.0.1.37:8201"
cluster_name = "vault-prod-us-east-1"
ui           = true
log_level    = "INFO"
```

**Notice:** No `seal` stanza → Vault defaults to Shamir. We need to add one.

---

## Step 3: Add the AWS KMS Seal Stanza

### 3a. Get your KMS key ARN from AWS Console

```
arn:aws:kms:us-east-1:123456789012:key/abcd1234-5678-90ab-cdef-EXAMPLEKEY
```

### 3b. Edit the config file

```bash
sudo vi /etc/vault.d/vault.hcl
```

### 3c. Add the `seal "awskms"` stanza

Insert this block anywhere in the file:

```hcl
seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/abcd1234-5678-90ab-cdef-EXAMPLEKEY"
}
```

### 3d. Complete config should now look like:

```hcl
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "node-a-us-east-1"
  retry_join {
    auto_join = "provider=aws region=us-east-1 tag_key=vault tag_value=us-east-1"
  }
}

# ─── THIS IS THE NEW PART ───
seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/abcd1234-5678-90ab-cdef-EXAMPLEKEY"
}
# ─────────────────────────────

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_disable     = 1
}

api_addr     = "http://10.0.1.37:8200"
cluster_addr = "http://10.0.1.37:8201"
cluster_name = "vault-prod-us-east-1"
ui           = true
log_level    = "INFO"
```

> **Security Warning:** Never commit `kms_key_id` or credentials to a public repo. Use environment variables in production.

---

## Step 4: Restart Vault and Verify Seal Type Changed

```bash
sudo systemctl restart vault
```

Now check the status:

```bash
vault status
```

**Output:**

```
Key                  Value
----                 -----
Seal Type            awskms         ← Changed from "shamir" to "awskms"!
Initialized          false
Sealed               true
Total Recovery Shares 0             ← Notice: "Recovery Shares" not "Shares"
Threshold            0
Version              1.7.1
Storage Type         raft
HA Enabled           true
```

### What changed?

```
BEFORE                              AFTER
──────                              ─────
Seal Type: shamir          →        Seal Type: awskms
Total Shares: 0            →        Total Recovery Shares: 0
```

Two key differences:
1. **Seal Type** flipped from `shamir` to `awskms`
2. **"Shares"** became **"Recovery Shares"** — because with auto unseal, you get recovery keys, not unseal keys

---

## Step 5: Initialize Vault

```bash
vault operator init
```

**Output:**

```
Recovery Key 1:  qDfLTvJNhT3Dgj8UWaep9o2qgQZcVq/+w6QXQ4Tq+
Recovery Key 2:  ...
Recovery Key 3:  ...
Recovery Key 4:  ...
Recovery Key 5:  2/FshgVlCzLhqhG+C0M0azU3ry82c2KhmKSUpelv

Initial Root Token: s.7gu7dshRlK1KNoq8B9dFme

Success! Vault is initialized with 5 recovery shares and a threshold of 3.
```

### Notice the difference from Lab 01:

```
LAB 01 (Shamir)                     LAB 02 (Auto Unseal)
───────────────                     ────────────────────
"Unseal Key 1: ..."          →      "Recovery Key 1: ..."
"5 key shares"               →      "5 recovery shares"
Vault remains SEALED         →      Vault auto-UNSEALS immediately!
```

**Verify — Vault should already be unsealed:**

```bash
vault status
```

```
Key                      Value
---                      -----
Seal Type                awskms
Recovery Seal Type       shamir
Initialized              true
Sealed                   false           ← Already unsealed! No manual steps!
Total Recovery Shares    5
Threshold                3
Version                  1.7.1
Storage Type             raft
Cluster Name             vault-prod-us-east-1
Cluster ID               6245bbfd-8db5-b507-f689-ba48628ad2a5
HA Enabled               true
HA Cluster               http://10.0.1.37:8201
HA Mode                  active
```

Vault **automatically unsealed** by calling AWS KMS to decrypt the master key. No shards needed!

---

## Step 6: Authenticate and Use Vault

### Login with root token:

```bash
vault login s.7gu7dshRlK1KNoq8B9dFme
```

```
Success! You are now authenticated. Token policies: ["root"]
```

### Enable a secrets engine (let's try Azure):

```bash
vault secrets enable azure
```

### List all engines:

```bash
vault secrets list
```

```
Path          Type        Accessor                Description
----          ----        --------                -----------
azure/        azure       azure_6d868445          n/a
cubbyhole/    cubbyhole   cubbyhole_2e79ae0c      per-token private secret storage
identity/     identity    identity_65b04cae        identity store
sys/          system      system_9d391d96          system endpoints
```

Everything works — just like with Shamir, but without any manual unsealing!

---

## Step 7: The Magic Test — Restart and Auto-Unseal

This is where auto unseal proves its value. Let's restart Vault and see what happens:

```bash
sudo systemctl restart vault
```

Wait a few seconds, then:

```bash
vault status
```

```
Key                      Value
---                      -----
Seal Type                awskms
Sealed                   false           ← Still unsealed after restart!
```

### Compare with Lab 01:

```
Lab 01 (Shamir):
  Restart → Sealed → Call 3 people → Type shards → Unsealed (minutes/hours)

Lab 02 (Auto Unseal):
  Restart → Auto-unsealed in seconds → Ready to serve (no humans needed)
```

---

## Lab Complete! What We Did

```
Step 1: vault status               → Confirmed Shamir seal (default)
Step 2: cat vault.hcl              → No seal stanza present
Step 3: Added seal "awskms" block  → Pointed to our KMS key ARN
Step 4: systemctl restart vault    → Seal Type changed to "awskms"
Step 5: vault operator init        → Got RECOVERY keys (not unseal keys)
                                      Vault auto-unsealed immediately!
Step 6: vault login + secrets      → Used Vault normally
Step 7: Restarted Vault            → Still unsealed — auto unseal works!
```

---

## Side-by-Side: Lab 01 vs Lab 02

| Aspect | Lab 01 (Shamir) | Lab 02 (Auto Unseal) |
|--------|-----------------|---------------------|
| **Seal stanza in config** | None (default) | `seal "awskms" { ... }` |
| **`vault operator init` gives** | Unseal Keys | Recovery Keys |
| **After init** | Still sealed | Auto-unsealed! |
| **After restart** | Sealed → need 3 humans | Auto-unsealed in seconds |
| **Seal Type in status** | `shamir` | `awskms` |
| **Human intervention** | Every restart | Never |

---

## Common Issues & Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `Error initializing seal: error assuming role` | EC2 instance doesn't have KMS permissions | Attach IAM role with `kms:Encrypt`, `kms:Decrypt`, `kms:DescribeKey` |
| `Seal Type still shows shamir` | Vault not restarted after config change | Run `sudo systemctl restart vault` |
| `AccessDeniedException` from KMS | Wrong KMS key ARN or wrong region | Double-check the ARN and region in your config |
| `KMS key not found` | KMS key was deleted or in a different account | Verify key exists in AWS Console |
| Vault stays sealed after restart | KMS/network issue — Vault can't reach AWS | Check network, security groups, VPC endpoints |

---

## Official References

- [HashiCorp Vault Docs](https://www.vaultproject.io/docs)
- [AWS KMS — Create a Customer Managed Key](https://docs.aws.amazon.com/kms/latest/developerguide/create-keys.html)
- [Vault AWS KMS Seal Config](https://www.vaultproject.io/docs/configuration/seal/awskms)
