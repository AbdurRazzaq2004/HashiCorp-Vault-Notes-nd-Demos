# Demo: Installing Vault Using Packer

> **Module 2 — Installing Vault Server**
> Lab: Bake a custom AMI with Vault pre-installed using Packer.

---

## Lab Overview

In this lab, you'll use **Packer** to create a reusable Amazon Machine Image (AMI) with Vault already installed. This is how real teams do it — instead of manually installing Vault on every server, you **bake it into a golden image** once, then launch as many instances as you need.

Think of it like a **frozen meal** — you prep once, freeze it, and reheat whenever you're hungry. No cooking from scratch every time.

```
┌────────────────────────────────────────────────────────────────┐
│                  Packer AMI Build Flow                          │
│                                                                 │
│  vault.pkr.hcl          Packer Build        Custom AMI         │
│  ┌──────────┐          ┌──────────┐        ┌──────────┐       │
│  │ Template │──build──▶│ Temp EC2 │──snap─▶│ AMI with │       │
│  │ + files/ │          │ Instance │        │ Vault    │       │
│  └──────────┘          └──────────┘        │ installed│       │
│                         (terminated)        └──────────┘       │
│                                                  │             │
│                                            Launch as many      │
│                                            instances as needed │
└────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- AWS account with VPC and subnet
- Packer installed locally
- Vault ZIP downloaded from [releases.hashicorp.com](https://releases.hashicorp.com/)
- SSH key pair for EC2 access

---

## Step 1: Repository Structure

Clone the repo and navigate to `vault/packer`:

```
vault/packer/
├── vault.pkr.hcl              ← Packer HCL2 template
└── files/
    ├── vault.hcl              ← Vault server configuration
    ├── vault.service          ← systemd unit file
    └── vault_int_storage.hcl  ← Integrated Storage config example
```

---

## Step 2: Download the Vault Binary

```bash
# Download Vault 1.7.1 for Linux
curl -O https://releases.hashicorp.com/vault/1.7.1/vault_1.7.1_linux_amd64.zip
```

> **Always verify** the SHA256 checksum to ensure file integrity.

---

## Step 3: Configure the Packer Template

The `vault.pkr.hcl` template defines everything Packer needs:

```hcl
# ─── VARIABLES ─────────────────────────────────────────
variable "aws_region" {
  type    = string
  default = env("AWS_REGION")
}

variable "vault_zip" {
  type    = string
  default = "/path/to/vault_1.7.1_linux_amd64.zip"
}

variable "vpc_id" {
  type    = string
  default = "vpc-xxxx"
}

variable "subnet_id" {
  type    = string
  default = "subnet-xxxx"
}

# ─── DATA SOURCE: Find latest Amazon Linux 2 AMI ──────
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# ─── SOURCE: EC2 Instance for Building ─────────────────
source "amazon-ebs" "vault-amzn2" {
  region                      = var.aws_region
  ami_name                    = "vault-amazonlinux2-{{timestamp}}"
  instance_type               = "t2.micro"
  source_ami                  = data.aws_ami.amazon_linux_2.id
  ssh_username                = "ec2-user"
  associate_public_ip_address = true
  subnet_id                   = var.subnet_id
  vpc_id                      = var.vpc_id
  tags = {
    Name = "HashiCorp Vault"
    OS   = "Amazon Linux 2"
  }
}

# ─── BUILD: Upload files + Install Vault ───────────────
build {
  sources = ["source.amazon-ebs.vault-amzn2"]

  # Upload Vault ZIP to temp instance
  provisioner "file" {
    source      = var.vault_zip
    destination = "/tmp/vault.zip"
  }

  # Upload config files
  provisioner "file" {
    source      = "files/"
    destination = "/tmp"
  }

  # Install Vault and configure systemd
  provisioner "shell" {
    inline = [
      "unzip /tmp/vault.zip -d /usr/local/bin",
      "chmod +x /usr/local/bin/vault",
      "mkdir -p /etc/vault.d",
      "mv /tmp/vault.hcl /etc/vault.d/",
      "mv /tmp/vault_int_storage.hcl /etc/vault.d/",
      "mv /tmp/vault.service /etc/systemd/system/",
      "systemctl daemon-reload",
      "systemctl enable vault"
    ]
  }
}
```

### What Each Provisioner Does

```
provisioner "file" (vault.zip)     → Uploads Vault binary to /tmp
provisioner "file" (files/)        → Uploads config + systemd files to /tmp
provisioner "shell"                → Unzips, moves to PATH, configures systemd
```

---

## Step 4: Set AWS Variables

Either update defaults in the template or pass at build time:

```bash
packer build \
  -var aws_region=us-east-1 \
  -var vpc_id=vpc-123456 \
  -var subnet_id=subnet-abcdef \
  vault.pkr.hcl
```

---

## Step 5: Validate & Build the AMI

```bash
# Validate template syntax
packer validate vault.pkr.hcl

# Build the AMI
packer build vault.pkr.hcl
```

What happens behind the scenes:

```
1. Packer launches a temporary EC2 instance
2. Uploads Vault ZIP + config files
3. Runs shell provisioner (install + systemd setup)
4. Creates an AMI snapshot from the instance
5. Terminates the temporary instance
6. AMI is now available in EC2 → AMIs
```

---

## Step 6: Launch & Validate

1. Go to **EC2 → AMIs** → find `vault-amazonlinux2-*`
2. Launch a new instance from the AMI (T2 Micro, public IP)
3. Open SSH (port 22) in security group
4. SSH in and verify:

```bash
ssh -i key.pem ec2-user@<public-ip>

# Check Vault is installed and enabled
sudo systemctl status vault
vault version
# Vault v1.7.1 (abcd1234)
```

---

## Quick Recap

```
Packer + Vault AMI Build
│
├── Template (vault.pkr.hcl)
│   ├── Variables: region, VPC, subnet, vault ZIP path
│   ├── Data source: Latest Amazon Linux 2 AMI
│   ├── Source: amazon-ebs builder (t2.micro)
│   └── Build: file + shell provisioners
│
├── Files baked into AMI:
│   ├── /usr/local/bin/vault        (binary)
│   ├── /etc/vault.d/vault.hcl     (config)
│   └── /etc/systemd/system/vault.service (systemd)
│
└── Result: Reusable AMI → launch unlimited Vault instances
```

---

## Official References

- [Packer Documentation](https://www.packer.io/docs)
- [Vault Downloads](https://releases.hashicorp.com/vault/)
- [AWS EC2 User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/)
