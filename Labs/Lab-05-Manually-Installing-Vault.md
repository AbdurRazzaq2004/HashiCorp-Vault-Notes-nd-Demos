# Demo: Manually Installing Vault

> **Module 2 — Installing Vault Server**
> Lab: Download, install, and verify the Vault binary on Amazon Linux 2.

---

## Lab Overview

This lab walks through the **most fundamental** Vault install — downloading the binary, placing it in your PATH, and verifying it works. No package managers, no automation — just you and the binary.

This is important to understand even if you'll always use package managers or Packer, because it's what everything else automates under the hood.

```
┌───────────────────────────────────────────────────────┐
│              Manual Install Flow                       │
│                                                        │
│  Download ZIP ──▶ Unzip ──▶ Move to PATH ──▶ Verify   │
│                                                        │
│  curl ...zip     unzip     mv vault          vault     │
│                  vault.zip /usr/local/bin/    --version │
└───────────────────────────────────────────────────────┘
```

---

## Prerequisites

- AWS EC2 instance running Amazon Linux 2 (or any Linux distro)
- SSH access to the instance
- Root or sudo privileges

---

## Method 1: YUM Repository (Package Manager)

The **easiest** method on RHEL-based systems:

```bash
# Install yum-utils for repo management
sudo yum install -y yum-utils

# Add the official HashiCorp repo
sudo yum-config-manager --add-repo \
  https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo

# Install Vault
sudo yum -y install vault
```

---

## Method 2: Manual Binary Download

### Step 1 — Verify Vault Is NOT Installed

```bash
[root@ip-10-0-1-160 /]# vault
bash: vault: command not found
```

### Step 2 — Download the ZIP

```bash
curl --silent -Lo /tmp/vault.zip \
  https://releases.hashicorp.com/vault/1.7.1/vault_1.7.1_linux_amd64.zip
```

### Step 3 — Unzip and Move to PATH

```bash
cd /tmp
unzip vault.zip
sudo mv vault /usr/local/bin/
```

### Step 4 — Verify Installation

```bash
$ vault
Usage: vault <command> [args]

Common commands:
    read    Read data and retrieve secrets
    write   Write data, configuration, and secrets
    delete  Delete secrets and configuration
    list    List data or secrets
    login   Authenticate locally
    agent   Start a Vault agent
    server  Start a Vault server
    unwrap  Unwrap a wrapped secret

$ vault version
Vault v1.7.1 (abcd1234)
```

### Step 5 — Quick Dev Server Test

```bash
# Start dev mode to confirm everything works
vault server -dev

# Output includes:
# Unseal Key: ZEgZgHSEEmmlnRboqtY0A00TUpleaoxo8SqqtFP2Q=
# Root Token: s.d6931rVSdkpBINnnRvMHBRXR
```

In another terminal:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
vault status
```

---

## YUM vs Manual — Comparison

| | YUM Repository | Manual Download |
|--|---------------|-----------------|
| **Commands** | 3 commands | 3 commands |
| **Auto-updates** | `yum update vault` | Must re-download |
| **PATH setup** | Automatic | Manual (`mv` to `/usr/local/bin/`) |
| **Checksum verification** | Automatic (GPG) | Manual (compare SHA256) |
| **Best for** | Servers with internet | Air-gapped environments |

---

## Quick Recap

```
Manual Vault Install
│
├── Method 1: YUM (easy)
│   └── yum install vault
│
├── Method 2: Binary (manual)
│   ├── curl → download ZIP
│   ├── unzip → extract binary
│   ├── mv → place in /usr/local/bin/
│   └── vault --version → verify
│
└── Quick test: vault server -dev
```

---

## Official References

- [Vault Downloads](https://www.vaultproject.io/download)
- [HashiCorp Releases](https://releases.hashicorp.com/vault/)
