# Installing and Running Vault Server

> **Module 2 — Installing Vault Server**
> Topic: How to install Vault on any platform and bring it to life.

---

## Overview

Installing Vault is like setting up a **restaurant kitchen** — before you can start cooking (managing secrets), you need to:

1. **Buy the equipment** (install the binary)
2. **Design the layout** (create a config file)
3. **Open the doors** (start the server)
4. **Set the combination on the safe** (initialize)
5. **Unlock the safe** (unseal)

Vault is intentionally **platform-agnostic** — it runs everywhere from your laptop to a Kubernetes cluster. It ships as a single binary with zero dependencies.

```
┌──────────────────────────────────────────────────────────────┐
│                   Vault Installation Flow                      │
│                                                               │
│  Step 1         Step 2          Step 3        Step 4   Step 5 │
│ ┌──────┐     ┌──────────┐    ┌────────┐    ┌──────┐  ┌─────┐│
│ │Install│────▶│Config    │────▶│ Start  │────▶│ Init │──▶│Unseal││
│ │Binary │    │File(.hcl)│    │Server  │    │      │  │     ││
│ └──────┘     └──────────┘    └────────┘    └──────┘  └─────┘│
│                                                               │
│ vault         vault.hcl       vault server   vault     vault  │
│ --version                     -config=...    operator  operator│
│                                              init      unseal │
└──────────────────────────────────────────────────────────────┘
```

---

## 1. Supported Platforms

Vault runs **anywhere** — it doesn't care about your infrastructure:

| Platform | Examples | Use Case |
|----------|----------|----------|
| **Kubernetes** | AKS, EKS, GKE, self-hosted | Cloud-native, microservices |
| **Cloud VMs** | AWS EC2, Azure VM, GCE | Traditional cloud deployments |
| **VMware** | vSphere, ESXi | Private cloud / data center |
| **Physical Servers** | Bare metal | Isolated crypto operations, high security |
| **Local Workstations** | Laptops, desktops | Development and learning |

```
                     Vault Runs On...

  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │Kubernetes│  │ Cloud VM │  │  VMware  │  │  Bare    │
  │ AKS/EKS/ │  │ EC2/Azure│  │ vSphere  │  │  Metal   │
  │ GKE      │  │ GCE      │  │          │  │          │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘
       │              │             │              │
       └──────────────┴─────────────┴──────────────┘
                          │
                    Single binary
                    Zero dependencies
```

> **Security note:** Some teams choose physical servers to completely isolate Vault's cryptographic operations from noisy neighbors on shared hardware.

---

## 2. Supported Operating Systems

| OS | Typical Use |
|----|------------|
| **Linux** | Production servers (Ubuntu, RHEL, Amazon Linux, CentOS) |
| **macOS** | Local development on Apple hardware |
| **Windows** | Development or Windows-based production servers |
| **FreeBSD / NetBSD / OpenBSD** | Specialized or legacy environments |
| **Solaris** | Legacy enterprise environments |

> **In practice:** The vast majority of production Vault deployments run on **Linux** (Ubuntu or RHEL/Amazon Linux).

---

## 3. Installation Methods

### Method 1: Package Manager — APT (Debian/Ubuntu)

The **easiest** way to install on Linux:

```bash
# Add HashiCorp GPG key
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -

# Add the official HashiCorp repository
sudo apt-add-repository \
  "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"

# Update and install
sudo apt-get update
sudo apt-get install vault
```

**What this does:**
1. Adds HashiCorp's package signing key (verifies authenticity)
2. Adds the APT repository (so `apt` knows where to find Vault)
3. Installs the `vault` binary into your system PATH

### Method 2: Package Manager — YUM/DNF (RHEL/CentOS/Fedora)

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install vault
```

### Method 3: Homebrew (macOS)

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/vault
```

### Method 4: Helm Chart (Kubernetes)

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace
```

Deploys Vault as a Kubernetes Deployment with a Service and StatefulSet for storage.

### Method 5: Manual Download (Any Platform)

```bash
# 1. Download the ZIP archive
curl -O https://releases.hashicorp.com/vault/1.12.3/vault_1.12.3_linux_amd64.zip

# 2. Unzip
unzip vault_1.12.3_linux_amd64.zip

# 3. Move to a PATH location
sudo mv vault /usr/local/bin/

# 4. Verify
vault --version
```

### Installation Methods Comparison

| Method | Command | Best For |
|--------|---------|----------|
| APT (Debian/Ubuntu) | `apt-get install vault` | Linux production servers |
| YUM/DNF (RHEL) | `yum install vault` | RHEL-based production servers |
| Homebrew (macOS) | `brew install hashicorp/tap/vault` | Mac development |
| Helm (Kubernetes) | `helm install vault hashicorp/vault` | K8s deployments |
| Manual download | `curl` + `unzip` + `mv` | Any platform, air-gapped envs |

---

## 4. Verifying the Installation

After installing, confirm it works:

```bash
$ vault --version
Vault v1.12.3 (209b3dd99fe8ca320340e95116aa33cb09263eef)
```

You can also check available commands:

```bash
$ vault -h
```

And check if autocomplete is enabled:

```bash
$ vault -autocomplete-install
```

---

## 5. The Installation Workflow (End-to-End)

After installing the binary, here's the full sequence to get Vault running:

```
┌─────────────────────────────────────────────────────────────┐
│ STEP 1: Install Vault                                        │
│   apt-get install vault  (or other method)                  │
│   vault --version                                            │
│                                                              │
│ STEP 2: Create Configuration File                            │
│   /etc/vault.d/vault.hcl                                    │
│   (storage, listener, seal, top-level params)               │
│                                                              │
│ STEP 3: Start the Server                                     │
│   vault server -config=/etc/vault.d/vault.hcl              │
│   (or via systemd: systemctl start vault)                   │
│                                                              │
│ STEP 4: Initialize Vault (one-time)                          │
│   vault operator init                                        │
│   (generates master key shards + root token)                │
│                                                              │
│ STEP 5: Unseal Vault                                         │
│   vault operator unseal  (repeat for threshold)             │
│   (or auto-unseal via KMS)                                  │
│                                                              │
│ ✅ Vault is READY — store secrets, issue creds              │
└─────────────────────────────────────────────────────────────┘
```

### Automation Tools

Don't do this manually in production. Use automation:

| Tool | Purpose |
|------|---------|
| **Terraform** | Provision infrastructure + deploy Vault |
| **Ansible** | Configure and install Vault on servers |
| **Helm** | Deploy Vault on Kubernetes |
| **Packer** | Bake Vault into machine images (AMIs, etc.) |
| **systemd** | Manage Vault as a system service (auto-start, logging) |

---

## 6. Dev Mode vs Production Mode

| | Dev Mode | Production Mode |
|--|----------|----------------|
| **Command** | `vault server -dev` | `vault server -config=vault.hcl` |
| **Config file** | Not needed | Required |
| **Storage** | In-memory (lost on restart) | Persistent (Consul, Raft, etc.) |
| **TLS** | Disabled | Must be configured |
| **Seal state** | Auto-unsealed | Sealed (must unseal manually or auto) |
| **Root token** | Printed to terminal | Generated during `vault operator init` |
| **Init required?** | No (auto) | Yes |
| **Use case** | Learning, dev, testing | Staging, production |

```
Dev Mode:                          Production Mode:
┌──────────────┐                  ┌──────────────┐
│ vault server │                  │ vault server │
│ -dev         │                  │ -config=...  │
│              │                  │              │
│ • In-memory  │                  │ • Persistent │
│ • No TLS    │                  │ • TLS on     │
│ • Unsealed  │                  │ • Sealed     │
│ • Root token│                  │ • Must init  │
│   printed   │                  │   + unseal   │
│              │                  │              │
│ NEVER use   │                  │ Always use   │
│ in prod!    │                  │ in prod!     │
└──────────────┘                  └──────────────┘
```

---

## Quick Recap

```
Installing Vault
│
├── Single binary, zero dependencies, platform-agnostic
│
├── Installation Methods:
│   ├── Package manager (APT, YUM, Homebrew)
│   ├── Helm chart (Kubernetes)
│   └── Manual download (curl + unzip)
│
├── Post-Install Workflow:
│   1. Install binary
│   2. Create config file (vault.hcl)
│   3. Start server (vault server -config=...)
│   4. Initialize (vault operator init) — one-time
│   5. Unseal (vault operator unseal)
│
├── Runs on: Linux, macOS, Windows, *BSD, Solaris
├── Runs on: K8s, Cloud VMs, VMware, Bare Metal, Laptops
│
└── Automate with: Terraform, Ansible, Helm, Packer
```

---

## Key Exam Tips

1. **Vault is a single binary** — no additional dependencies needed.
2. **Platform-agnostic** — runs on Kubernetes, cloud VMs, VMware, bare metal, and workstations.
3. **`vault server -dev`** = dev mode (in-memory, no TLS, auto-unsealed). **Never use in production.**
4. **`vault server -config=<path>`** = production mode (persistent storage, TLS, must init + unseal).
5. **Installation workflow:** Install → Config → Start → Init → Unseal (5 steps).
6. **`vault operator init`** is a one-time operation. **`vault operator unseal`** is needed on every restart (unless auto-unseal).
7. **Verify installation** with `vault --version`.
8. **Use systemd** (or equivalent) to manage Vault as a service in production.
9. **Most production deployments** run on Linux (Ubuntu, RHEL, Amazon Linux).
10. **Automate installs** with Terraform, Ansible, or Helm — never manual in production.

---

## Official References

- [Vault Installation Guide](https://www.vaultproject.io/docs/install)
- [HashiCorp Releases](https://releases.hashicorp.com/vault/)
- [Vault Helm Charts](https://github.com/hashicorp/vault-helm)
- [APT Package Repository](https://apt.releases.hashicorp.com/)
