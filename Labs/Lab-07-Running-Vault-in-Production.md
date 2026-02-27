# Demo: Running Vault in Production

> **Module 2 — Installing Vault Server**
> Lab: Set up a production-grade Vault node on EC2 with Raft storage, AWS KMS auto-unseal, and systemd.

---

## Lab Overview

This is the **real deal** — setting up Vault the way it runs in production. You'll create a dedicated system user, write a systemd service file, configure Raft storage with AWS KMS auto-unseal, and start Vault as a managed service.

```
┌──────────────────────────────────────────────────────────────┐
│              Production Setup Steps                           │
│                                                               │
│  1. Install binary    → /usr/local/bin/vault                 │
│  2. Create vault user → Non-root, no shell                   │
│  3. Create dirs       → /etc/vault.d + /opt/vault/data1     │
│  4. Write systemd     → /etc/systemd/system/vault.service   │
│  5. Write config      → /etc/vault.d/vault.hcl              │
│  6. Start + verify    → systemctl start vault                │
└──────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- EC2 instance with Vault binary in `/tmp` (e.g., from Packer AMI)
- AWS IAM role with KMS permissions attached to the instance
- AWS KMS key created for auto-unseal

---

## Step 1: Install the Vault Binary

```bash
cd /tmp && ls                          # Confirm vault.zip is here
sudo unzip vault.zip
sudo mv vault /usr/local/bin/vault
vault --version                        # Vault v1.7.1
```

---

## Step 2: Create Vault System User and Directories

```bash
# Create a dedicated system user (no shell, no login)
sudo useradd --system --home /var/lib/vault --shell /sbin/nologin vault

# Create config and data directories
sudo mkdir -p /etc/vault.d /opt/vault/data1

# Set ownership
sudo chown -R vault:vault /etc/vault.d /opt/vault
```

| Directory | Purpose | Owner |
|-----------|---------|-------|
| `/etc/vault.d` | Configuration files | vault:vault |
| `/opt/vault/data1` | Raft storage data | vault:vault |
| `/var/lib/vault` | Vault home directory | vault:vault |

> **Why a system user?** Never run Vault as root. A dedicated user with no shell login minimizes attack surface.

---

## Step 3: Define the systemd Service

Create `/etc/systemd/system/vault.service`:

```ini
[Unit]
Description="HashiCorp Vault - Secrets Management"
Documentation=https://www.vaultproject.io/docs/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault.d/vault.hcl
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/local/bin/vault server --config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5s
TimeoutStopSec=30s

[Install]
WantedBy=multi-user.target
```

### Key systemd Directives Explained

| Directive | What It Does |
|-----------|-------------|
| `User=vault` | Runs as the vault user (not root) |
| `CAP_IPC_LOCK` | Allows Vault to lock memory (prevent swap) |
| `ProtectSystem=full` | Makes `/usr`, `/boot`, `/etc` read-only |
| `PrivateTmp=yes` | Isolates Vault's `/tmp` from other processes |
| `Restart=on-failure` | Auto-restart if Vault crashes |
| `ConditionFileNotEmpty` | Won't start without a config file |

Enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable vault
```

---

## Step 4: Vault Configuration (vault.hcl)

Create `/etc/vault.d/vault.hcl`:

```hcl
storage "raft" {
  path    = "/opt/vault/data1"
  node_id = "node-a-us-east-1"

  retry_join {
    auto_join = [
      "provider=aws",
      "region=us-east-1",
      "tag_key=vault",
      "tag_value=us-east-1"
    ]
  }
}

seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "arn:aws:kms:us-east-1:003674902126:key/8bc6b2ab-..."
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_disable     = 1    # Enable TLS in real production!
}

api_addr     = "http://10.0.1.37:8200"
cluster_addr = "http://10.0.1.37:8201"
cluster_name = "vault-prod-us-east-1"

ui        = true
log_level = "INFO"
```

> **⚠️ Production:** Always enable TLS (`tls_disable = false`) with real certificates.

---

## Step 5: Start and Verify

```bash
# Start Vault
sudo systemctl start vault

# Check status
vault status
```

Expected output:

```
Key             Value
---             -----
Seal Type       awskms
Initialized     false        ← Needs initialization
Sealed          false        ← Auto-unsealed via KMS
Total Shares    0
Version         1.7.1
Storage Type    raft
HA Enabled      true
```

### Monitor Logs

```bash
# Service status
sudo systemctl status vault

# Live log streaming
sudo journalctl -u vault -f
```

You should see AWS auto-join discovery:

```
[INFO]  core: [DEBUG] discover-aws: Creating session...
[INFO]  core: [DEBUG] discover-aws: Filter instances with vault=us-east-1
```

---

## Step 6: Next Steps

After this lab, you would:

1. `vault operator init` — initialize (one time)
2. Distribute recovery keys securely
3. `vault login <root-token>` — authenticate
4. Configure auth methods, policies, secrets engines
5. Revoke the root token

---

## Quick Recap

```
Production Vault Setup
│
├── Binary:   /usr/local/bin/vault
├── User:     vault (system, no shell)
├── Config:   /etc/vault.d/vault.hcl
├── Data:     /opt/vault/data1 (Raft)
├── Service:  /etc/systemd/system/vault.service
├── Seal:     AWS KMS (auto-unseal)
│
└── Commands:
    systemctl start vault    ← Start
    systemctl status vault   ← Check
    journalctl -u vault -f   ← Logs
    vault status             ← Seal/init status
```

---

## Official References

- [Vault Documentation](https://www.vaultproject.io/docs/)
- [AWS KMS Auto-Unseal](https://www.vaultproject.io/docs/configuration/seal/awskms)
- [Packer by HashiCorp](https://www.packer.io/)
