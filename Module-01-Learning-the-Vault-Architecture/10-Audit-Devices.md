# Audit Devices

> **Module 1 — Learning the Vault Architecture**
> Topic: How Vault logs every request and response for security and compliance.

---

## Overview

Audit devices are Vault's **security cameras** — they record every single authenticated request and response in detailed JSON logs. Nothing gets past them. If someone reads a secret, writes a policy, or logs in — it's logged.

Think of it like a **bank's CCTV system**:

- Every transaction is recorded (who, what, when)
- Footage goes to multiple locations (redundancy)
- Sensitive info is blurred out in recordings (HMAC hashing)
- If ALL cameras go down, the bank shuts its doors (Vault refuses requests)

```
┌──────────────────────────────────────────────────────────────┐
│                     Vault Server                              │
│                                                               │
│  Client Request ──▶ Audit Log (BEFORE processing)            │
│         │                                                     │
│         ▼                                                     │
│  Process Request                                              │
│         │                                                     │
│         ▼                                                     │
│  Client Response ──▶ Audit Log (AFTER processing)             │
│                                                               │
│  Both request AND response are logged as JSON                │
│  Sensitive data is HMAC-hashed (not plaintext!)              │
└──────────────────────────────────────────────────────────────┘
```

---

## 1. How Audit Logging Works

### The Logging Flow

Every API call goes through this process:

```
1. Client sends request
        │
        ▼
2. Vault logs the REQUEST to audit device(s)
        │
        ▼
3. Vault processes the request
        │
        ▼
4. Vault logs the RESPONSE to audit device(s)
        │
        ▼
5. Client receives response
```

### What Gets Logged

| Logged Field | Example |
|-------------|---------|
| Timestamp | `2026-02-27T10:30:00Z` |
| Request type | `read`, `write`, `delete`, `list` |
| Path | `secret/data/myapp/db-creds` |
| Client token (hashed) | `hmac-sha256:abc123...` |
| Client IP | `10.0.1.50` |
| Auth method | `userpass`, `approle`, `token` |
| Response data (hashed) | `hmac-sha256:def456...` |
| Errors (if any) | `permission denied` |

### HMAC Hashing — Protecting Sensitive Data

Vault **never** writes plaintext secrets into audit logs. Instead, it uses HMAC-SHA256 to hash sensitive values:

```
What the secret actually is:
  password = "SuperSecret123!"

What appears in the audit log:
  password = "hmac-sha256:3b2c8f9a1d..."

You can VERIFY a value matches the hash (for forensics),
but you can NOT reverse the hash to get the original value.
```

```
┌─────────────────────────────────────────────┐
│         HMAC Hashing in Audit Logs           │
│                                              │
│  Plaintext Secret                            │
│       │                                      │
│       ▼                                      │
│  HMAC-SHA256(secret, salt)                   │
│       │                                      │
│       ▼                                      │
│  "hmac-sha256:3b2c8f9a1d4e..."              │
│                                              │
│  ✅ Can verify: "Does X match this hash?"    │
│  ❌ Cannot reverse: "What was the secret?"   │
└─────────────────────────────────────────────┘
```

> **Why this matters:** You get a complete audit trail for compliance (SOC2, PCI-DSS, HIPAA) without exposing actual secrets in your logs.

---

## 2. Audit Device Types

Vault has **three built-in** audit backends:

### Comparison Table

| Type | How It Works | Best For | Considerations |
|------|-------------|----------|----------------|
| **file** | Appends JSON logs to a local file | On-premises, standalone servers | Need external tools for rotation & shipping |
| **syslog** | Forwards to local syslog daemon | Environments with existing syslog infra | Daemon can relay to centralized server |
| **socket** | Streams over TCP, UDP, or Unix socket | Remote log aggregation, custom pipelines | Avoid UDP for critical logs (unreliable) |

### Visual Breakdown

```
                       Audit Devices
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
     ┌──────────┐    ┌──────────┐    ┌──────────┐
     │   FILE   │    │  SYSLOG  │    │  SOCKET  │
     │          │    │          │    │          │
     │ → Local  │    │ → syslog │    │ → TCP    │
     │   file   │    │   daemon │    │ → UDP    │
     │ → JSON   │    │ → Can    │    │ → Unix   │
     │   format │    │   relay  │    │   socket │
     └──────────┘    └──────────┘    └──────────┘
          │                │               │
          ▼                ▼               ▼
     Fluentd /       Centralized      Splunk /
     CloudWatch      Syslog Server    Datadog /
     Logs Agent                       Custom
```

### 2.1 File Audit Device

```bash
# Enable
vault audit enable file file_path=/var/log/vault_audit_log.log

# List enabled audit devices
vault audit list

# Disable
vault audit disable file/
```

> **Important:** The OS user running Vault (commonly `vault`) must have **write permissions** to the log file path. Without proper permissions, Vault will fail to enable the audit device.

**Log rotation & shipping** — The file backend only writes to disk. You need external tools:

| Tool | Purpose |
|------|---------|
| logrotate | Rotate log files to prevent disk fill |
| Fluentd / Filebeat | Ship logs to a central system |
| AWS CloudWatch Agent | Ship to CloudWatch Logs |

### 2.2 Syslog Audit Device

```bash
vault audit enable syslog
```

Forwards logs to the local syslog daemon (`/dev/log` or `rsyslog`), which can then relay to a centralized syslog server.

### 2.3 Socket Audit Device

```bash
# TCP (recommended for reliability)
vault audit enable socket socket_type=tcp address=logserver.example.com:9090

# UDP (fast but unreliable — avoid for critical logs)
vault audit enable socket socket_type=udp address=logserver.example.com:9090
```

> **⚠️ Warning:** UDP does not guarantee delivery. A dropped packet = a lost audit entry. Use TCP for critical logging.

---

## 3. The Critical Safety Rule

This is **the most important thing** to understand about audit devices:

```
╔══════════════════════════════════════════════════════════════╗
║                                                              ║
║  If Vault CANNOT write to ANY enabled audit device,          ║
║  it will REFUSE ALL client requests.                         ║
║                                                              ║
║  Vault prioritizes SECURITY over AVAILABILITY.               ║
║                                                              ║
║  No working audit device = Vault goes effectively offline.   ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

### Why This Matters

```
Scenario 1: One audit device, it fails
┌────────┐     ┌───────┐     ┌─────────────┐
│ Client │────▶│ Vault │────▶│ Audit (DOWN)│
│        │  ✖  │       │     └─────────────┘
│        │◀────│REFUSES│
│        │     │REQUEST│     Vault blocks ALL requests
└────────┘     └───────┘     until audit device recovers

Scenario 2: Two audit devices, one fails
┌────────┐     ┌───────┐     ┌─────────────┐
│ Client │────▶│ Vault │────▶│ Audit 1 (UP)│  ✅ Logs here
│        │  ✅ │       │     └─────────────┘
│        │◀────│WORKS! │     ┌─────────────┐
│        │     │       │──✖──│Audit 2(DOWN)│  ❌ Failed, but OK
└────────┘     └───────┘     └─────────────┘

As long as ONE audit device works, Vault keeps serving.
```

> **This is a TOP exam topic.** Remember: Vault will always choose to stop serving rather than lose audit data.

---

## 4. Best Practices

### The Must-Do List

| Practice | Why |
|----------|-----|
| **Enable at least one audit device** | Maintain a complete security trail |
| **Deploy multiple audit backends** | Prevent single point of failure |
| **Test your logging pipeline** | Verify logs reach your SIEM/retention system |
| **Monitor audit device health** | Alert on write failures before Vault blocks requests |
| **Set up log rotation** (file backend) | Prevent disk from filling up |
| **Use TCP over UDP** (socket backend) | Guarantee delivery of audit entries |

### Recommended Production Setup

```
Production Vault Cluster
     │
     ├──▶ Audit Device 1: file
     │    └─ /var/log/vault_audit.log
     │       └─ Shipped via Fluentd → Elasticsearch / Splunk
     │
     └──▶ Audit Device 2: syslog
          └─ Forwards to centralized syslog server
             └─ SIEM alerting on suspicious patterns

Two devices = if one fails, Vault keeps running!
```

---

## 5. Audit Log Format (JSON)

Audit logs are JSON. Here's a simplified view of what an entry looks like:

```json
{
  "time": "2026-02-27T10:30:00.000Z",
  "type": "request",
  "auth": {
    "client_token": "hmac-sha256:abc123...",
    "accessor": "hmac-sha256:def456...",
    "display_name": "userpass-admin",
    "policies": ["default", "admin-policy"]
  },
  "request": {
    "id": "request-id-123",
    "operation": "read",
    "path": "secret/data/myapp/db-creds",
    "remote_address": "10.0.1.50"
  },
  "response": {
    "data": {
      "username": "hmac-sha256:ghi789...",
      "password": "hmac-sha256:jkl012..."
    }
  }
}
```

Notice: tokens, accessors, and secret values are all **HMAC-hashed** — never plaintext.

---

## Quick Recap

```
Audit Devices = Security cameras for Vault
│
├── Three Types:
│   ├── file    → Local JSON file (need external shipping)
│   ├── syslog  → Forwards to syslog daemon
│   └── socket  → Streams over TCP/UDP/Unix socket
│
├── What Gets Logged:
│   ├── Every authenticated request (who, what, when, where)
│   ├── Every response (including errors)
│   └── Sensitive data = HMAC-hashed (not plaintext)
│
├── Critical Safety Rule:
│   ├── No working audit device → Vault REFUSES all requests
│   └── Security > Availability (always)
│
└── Best Practices:
    ├── Enable AT LEAST one (always)
    ├── Enable MULTIPLE (prevent SPOF)
    ├── Monitor health + set up alerts
    └── Rotate logs + ship to SIEM
```

---

## Key Exam Tips

1. **Vault logs BOTH request AND response** — two entries per API call.
2. **Sensitive data is HMAC-hashed** — secrets never appear in plaintext in audit logs.
3. **If ALL audit devices fail, Vault stops serving requests** — this is the #1 exam question on audit devices.
4. **Security over availability** — Vault would rather go offline than lose audit data.
5. **Three types: file, syslog, socket** — know all three and when to use each.
6. **`vault audit enable file file_path=<path>`** — the CLI command to enable a file audit device.
7. **File permissions matter** — the Vault process user must have write access to the log path.
8. **Multiple audit devices** for redundancy — if one fails, Vault continues as long as one is healthy.
9. **UDP is unreliable** for socket type — always prefer TCP for critical audit logs.
10. **Audit devices are enabled at runtime** (CLI/API), NOT in the config file stanza (config file only declares, runtime enables).

---

## Official References

- [Vault Audit Devices Documentation](https://www.vaultproject.io/docs/audit)
- [File Audit Device](https://www.vaultproject.io/docs/audit/file)
- [Syslog Audit Device](https://www.vaultproject.io/docs/audit/syslog)
- [Socket Audit Device](https://www.vaultproject.io/docs/audit/socket)
