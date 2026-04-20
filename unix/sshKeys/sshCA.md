# SSH Certificate Authority (CA)

---

## Overview

An **SSH Certificate Authority (CA)** is a centrally trusted entity that **signs user public keys**, granting access to servers without the need to distribute individual keys to each host.

Instead of managing access like this:

```
Each server → stores many user public keys (authorized_keys)
```

You move to a model where:

```
Each server → trusts ONE CA
CA → signs user keys → access granted
```

This approach simplifies access management, improves scalability, and reduces the risk of key sprawl.

---

## Core Concepts

### 1. SSH Key Pair

Each user maintains an asymmetric key pair:

```
Private Key   (kept secret, stored on the user's device)
Public Key    (shareable, submitted to the CA for signing)
```

---

### 2. SSH Certificate

A **certificate** is a user's public key signed by the CA, embedded with metadata:

- **Identity** — a human-readable label
- **Principals** — the permitted login usernames
- **Validity period** — when the certificate expires
- **Restrictions** — optional usage constraints

---

### 3. Certificate Authority (CA)

The CA is composed of a key pair with distinct roles:

```
CA Private Key   ← Must be strictly protected
CA Public Key    ← Distributed to all target servers
```

The CA signs user public keys. Any server configured to trust the CA will accept connections from users holding a valid, CA-signed certificate.

---

## Architecture

```
                CA (Signer)
                    │
                    ▼
           Signed User Key (Certificate)
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
     server1     server2     server3
    (trust CA)  (trust CA)  (trust CA)
```

---

## Setup Steps

### Step 1 — Create the CA Key Pair

Run the following on a secure, dedicated machine:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ca_user -C "ssh-ca"
```

This produces:

```
~/.ssh/ca_user        ← CA private key (must remain secret)
~/.ssh/ca_user.pub    ← CA public key (distribute to servers)
```

---

### Step 2 — Configure Servers to Trust the CA

Copy the CA public key to each target server:

```bash
scp ~/.ssh/ca_user.pub user@<server-ip>:/tmp/
```

On each server, move the key to a permanent location and set appropriate permissions:

```bash
sudo mv /tmp/ca_user.pub /etc/ssh/ca_user.pub
sudo chmod 644 /etc/ssh/ca_user.pub
```

Edit the SSH server configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

Add the following directive:

```
TrustedUserCAKeys /etc/ssh/ca_user.pub
```

Restart the SSH service to apply the change:

```bash
sudo systemctl restart ssh
```

---

### Step 3 — Generate a User Key Pair

Each user generates their own key pair on their own machine. Private keys must never be shared between users.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_user -C "user-device"
```

---

### Step 4 — Sign the User's Public Key

An administrator (holding the CA private key) signs the user's public key to produce a certificate. The user's private key never leaves their device.

```bash
ssh-keygen -s ~/.ssh/ca_user \
    -I user_identity \
    -n user \
    -V +7d \
    ~/.ssh/id_user.pub
```

This produces:

```
id_user-cert.pub
```

---

### Step 5 — Connect to a Server

```bash
ssh -i ~/.ssh/id_user user@<server-ip>
```

SSH will automatically present the certificate (`id_user-cert.pub`) alongside the private key during authentication.

---

## Option Reference

| Option | Description                              |
|--------|------------------------------------------|
| `-s`   | CA private key used to sign the certificate |
| `-I`   | Certificate identity (human-readable label) |
| `-n`   | Comma-separated list of permitted login usernames |
| `-V`   | Validity period (e.g. `+7d`, `+8h`)     |

---

## Security Model

| Component       | Location        | Notes                        |
|-----------------|-----------------|------------------------------|
| CA Private Key  | Secure machine  | Must never be shared or exposed |
| CA Public Key   | All servers     | Safe to distribute           |
| User Private Key| User device     | Must remain confidential     |
| Certificate     | User device     | Short-lived; replace on expiry |

---

## Security Considerations

### CA Key Compromise

If the CA private key is compromised, an attacker can sign arbitrary keys and gain access to any server trusting that CA — representing a full system compromise.

Mitigations:
- Store the CA private key on a dedicated, air-gapped or hardware-secured machine
- Use separate CAs for development and production environments
- Rotate the CA key and re-issue certificates if a compromise is suspected

---

### Short-Lived Certificates

Issuing short-lived certificates significantly reduces exposure in the event of key theft or misuse:

```bash
-V +1d   # Valid for 1 day
-V +8h   # Valid for 8 hours
```

Benefits:
- Limits the window of opportunity for a compromised certificate
- Reduces the need for manual certificate revocation

---

## What to Avoid

- Sharing private keys between users
- Reusing the same key pair across multiple users
- Distributing the CA private key outside of a controlled environment
- Issuing long-lived certificates without a clear operational need

---

## Benefits

| Feature           | Benefit                                         |
|-------------------|-------------------------------------------------|
| Centralised trust | No need to copy keys to each individual server  |
| Scalability       | Supports large, distributed infrastructure      |
| Expiration        | Built-in, time-bound access control             |
| Clean management  | Eliminates `authorized_keys` sprawl             |

---

## Limitations

A basic SSH CA alone does **not** provide:

- Automated certificate issuance or renewal
- Identity verification (SSO / MFA)
- Centralised user lifecycle management (onboarding / offboarding)

---

## Automated Certificate Management

Manual certificate signing does not scale beyond small teams. Automated solutions handle issuance, distribution, and expiration without requiring users to manage certificate files directly.

### Typical Automation Flow

```
User authenticates (SSO / MFA)
         ↓
System verifies identity
         ↓
Short-lived SSH certificate is issued
         ↓
User connects to server
```

---

### Common Approaches

#### 1. Script-Based Automation (Lightweight)

Suitable for small teams with simple requirements. A script wraps the `ssh-keygen` signing command and can be integrated with internal tooling or a CLI.

```bash
#!/bin/bash
# Usage: ./sign-key.sh user.pub

CA_KEY=~/.ssh/ca_user
USER_PUBKEY="$1"

ssh-keygen -s "$CA_KEY" -I user_identity -n user -V +7d "$USER_PUBKEY"
```

```bash
./sign-key.sh user.pub
```

---

#### 2. API-Based CA Systems

Services that expose APIs for on-demand certificate issuance with centralised policy control:

- **Step CA** — Lightweight, open-source CA with REST API
- **HashiCorp Vault** — Enterprise-grade secrets management with SSH secrets engine

---

#### 3. Full Access Management Platforms

Platforms that manage the full SSH access lifecycle:

- **Teleport** — Provides SSO/MFA integration, role-based access control, automatic certificate issuance, and audit logging

---

### Benefits of Automation

- Eliminates manual certificate distribution
- Enforces short-lived credentials (hours rather than days)
- Simplifies revocation by centralising access control
- Improves compliance posture with audit trails

---

### When to Consider Automation

Automation is recommended when:

- Multiple users require access to shared infrastructure
- The team experiences frequent onboarding or offboarding
- Security or compliance requirements demand audit trails
- Manual signing introduces operational bottlenecks

---

## Summary

```
Manual CA       → suitable for small, low-complexity setups
Automated CA    → required for scalable, secure environments
```
