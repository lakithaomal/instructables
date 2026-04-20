# Step CA for SSH Certificate-Based Access

---

## Overview

**Step CA** is a lightweight Certificate Authority (CA) that automates the issuance of **SSH certificates**.

Instead of manually managing SSH keys across multiple servers, Step CA allows you to:

- Centralize trust using a CA
- Issue short-lived SSH certificates
- Avoid distributing public keys to every server
- Improve security and scalability

---

## Problem Statement

Traditional SSH setup:

```
User → copy public key → every server (authorized_keys)
```

Problems:

- Hard to manage at scale
- Requires updating every server
- No expiration of access
- Risk of key reuse

---

## Step CA Solution

```
User → requests certificate → Step CA signs → servers trust CA
```

Benefits:

- No per-server key management
- Short-lived credentials
- Centralized control
- Easy onboarding/offboarding

---

## Architecture

The following diagram illustrates the high-level interaction between components:

```
            Step CA (EC2)
                │
     ┌──────────┼──────────┐
     ▼          ▼          ▼
   Server1    Server2    Server3
 (trust CA)  (trust CA) (trust CA)

        ▲
        │
    User Laptop
  (requests certs)
```

---

## Core Concepts

### 1. SSH Key Pair

Each user has:

- Private key (kept secret)
- Public key (used for signing)

---

### 2. SSH Certificate

A signed version of a public key that includes:

- Username (principal)
- Expiration time
- CA signature

---

### 3. Certificate Authority (CA)

Step CA holds:

- CA private key (secret)
- CA public key (distributed to servers)

Servers trust any certificate signed by the CA.

---

## Setup Guide

This section provides a step-by-step procedure for deploying and configuring Step CA for SSH certificate-based authentication.

---

## Prerequisites

- SSH access to target servers
- `ssh-keygen` available on local machine
- Administrative (sudo) access on servers
- OpenSSH version supporting certificates (generally OpenSSH ≥ 5.4)

---

## Step 1 — Launch EC2 Instance

- OS: Ubuntu or Amazon Linux
- Open port: `9000` (or your chosen port)

---

## Network Deployment Options

### Public Access (Simple Setup)

- EC2 has a public IP
- Port 9000 is open (restricted by security group)
- Laptop connects directly to Step CA

### Private VPC (Recommended for Production)

- Step CA runs inside a private VPC (no public exposure)
- Access is restricted via:
  - VPN connection
  - SSH tunnel (port forwarding)

### Access via VPN (Recommended)

In most production environments, access to EC2 instances inside a private VPC is provided through a VPN.

Typical setup:

- A VPN server (e.g., OpenVPN, WireGuard, or AWS Client VPN) is deployed inside the VPC
- Your laptop connects to the VPN
- Once connected, you can directly reach private EC2 IPs (including Step CA)

Flow:

```
Laptop → VPN → VPC → Step CA
```

Benefits:

- No need to expose Step CA to the public internet
- No need for SSH tunneling per session
- Operates seamlessly for multiple services inside the VPC

High-level setup options:

1. **AWS Client VPN (managed)**
   - Fully managed by AWS
   - Integrates with IAM / certificates
   - Easiest for teams

2. **WireGuard (lightweight)**
   - Simple and fast
   - Run on a small EC2 instance (bastion/VPN node)

3. **OpenVPN**
   - Mature and widely used
   - More configuration required

Once connected to VPN, bootstrap Step CA using the private IP:

```
step ca bootstrap --ca-url https://<private-ec2-ip>:9000
```

Example SSH tunnel:

```
ssh -L 9000:localhost:9000 user@<bastion-host>
```

Then bootstrap using:

```
step ca bootstrap --ca-url https://localhost:9000
```

Benefits:

- CA is not exposed to the internet
- Reduced attack surface
- Better control over access

---

## Step 2 — Install Step CLI and Step CA

```bash
# Download and install step CLI
curl -L https://github.com/smallstep/cli/releases/latest/download/step_linux_amd64.tar.gz | tar xz
sudo mv step*/bin/* /usr/local/bin/
```

---

## Step 3 — Initialize Step CA

```bash
step ca init
```

Provide:

- CA Name: `My SSH CA`
- DNS: `<EC2-IP or domain>`
- Address: `:9000`
- Provisioner: `admin`

---

## Step 4 — Start Step CA

```bash
step-ca ~/.step/config/ca.json
```

The Certificate Authority will be accessible at the following endpoint:

```
https://<EC2-IP>:9000
```

---

## Step 5 — Configure Servers to Trust CA

### 1. Copy CA public key

From EC2:

```bash
step ssh config --roots
```

Copy output to each server:

```bash
sudo nano /etc/ssh/step_ca.pub
```

---

### 2. Update SSH config

```bash
sudo nano /etc/ssh/sshd_config
```

Add:

```
TrustedUserCAKeys /etc/ssh/step_ca.pub
```

---

### 3. Restart SSH

```bash
sudo systemctl restart ssh
```

---

## Step 6 — Install Step CLI on Laptop

```bash
brew install step
```

---

## Step 7 — Bootstrap CA on Laptop

```bash
step ca bootstrap --ca-url https://<EC2-IP>:9000
```

### What Bootstrap Does

The bootstrap step establishes trust between your laptop and the Step CA.

When you run this command:

```
step ca bootstrap --ca-url https://<EC2-IP>:9000
```

The following happens internally:

1. The Step CLI connects to the CA server
2. It downloads the CA root certificate
3. It stores the CA locally (typically under `~/.step/`)
4. It configures your system to trust this CA for future operations

This allows your laptop to securely request SSH certificates from the CA.

> ⚠️ Without this step, your system will not trust the CA and certificate requests will fail.

### Secure Bootstrap (Recommended)

For better security, use a fingerprint to verify the CA identity:

```
step ca bootstrap \
  --ca-url https://<EC2-IP>:9000 \
  --fingerprint <FINGERPRINT>
```

This prevents man-in-the-middle attacks during initial trust setup.

---

## Step 8 — Generate User Key

Each individual user should generate their own key pair on their own machine. Private keys must never be shared between users.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_user
```

---

## Step 9 — Request SSH Certificate

An administrator (holding the CA private key) takes a user's public key and signs it to generate a certificate. The user's private key remains on their device and is never shared.

```bash
step ssh certificate <username> ~/.ssh/id_user.pub
```

This generates:

```
~/.ssh/id_user-cert.pub
```

---

## Step 10 — SSH into Server

```bash
ssh -i ~/.ssh/id_user <username>@<server-ip>
```

---

## Operational Flow

```
1. User requests certificate
2. Step CA verifies identity
3. Step CA signs public key
4. Certificate returned to user
5. SSH uses:
   - private key
   - certificate
6. Server verifies:
   - signed by CA?
   - valid user?
   - not expired?
7. Access granted
```

---

## Security Model

| Component | Location | Notes |
|----------|--------|------|
| CA Private Key | EC2 | 🔴 Highly sensitive |
| CA Public Key | All servers | Safe |
| User Private Key | Laptop | 🔐 Secret |
| Certificate | Laptop | Temporary |

---

## Certificate Lifetime

Example:

```
Valid: +8h or +24h
```

Benefits:

- Limits exposure if compromised
- No need for manual revocation
- Forces periodic re-authentication

---

## Troubleshooting

### Inspect certificate details

```bash
ssh-keygen -L -f ~/.ssh/id_user-cert.pub
```

### Enable verbose SSH debugging

```bash
ssh -vvv -i ~/.ssh/id_user <username>@<server-ip>
```

### Common Issues

- "Permission denied"
  - Check username matches certificate principal
- Expired certificate
  - Re-run certificate command
- CA not trusted
  - Verify TrustedUserCAKeys path

---

## Best Practices

- Use short-lived certificates (hours to days)
- Store CA private key securely
- Separate development and production CAs
- Use meaningful identities

---

## Automated Certificate Management

While manual certificate issuance operates correctly, it does not scale. Automated solutions can manage certificate issuance and renewal.

### Script-Based Example

```bash
#!/bin/bash
CA_KEY="$HOME/.ssh/ca_user"
USER_PUBKEY="$1"
ssh-keygen -s "$CA_KEY" -I user_identity -n user -V +7d "$USER_PUBKEY"
```

---

---

## Benefits

The following summarizes the primary benefits of this approach:

| Feature | Benefit |
|--------|--------|
| Central trust | No key distribution |
| Scalable | Operates across many servers |
| Expiration | Built-in access control |
| Secure | Reduces key exposure |

---

## Summary

| Feature | Traditional SSH | Manual SSH CA | Step CA |
|--------|----------------|---------------|--------|
| Setup per Pi | Add key manually | Add CA public key | One-time CA trust |
| Adding new user | Update ALL Pis | No change needed | No change needed |
| Key compromise | Remove everywhere 😬 | Revoke / stop signing (manual) | Just stop issuing certs |
| Expiration | ❌ None | ✅ Possible (manual cert lifetime) | ✅ Built-in |
| Key reuse risk | High | Medium | Low |
| Scalability | Poor | Good | Excellent |
| Operational effort | Low initially, high over time | Medium (manual signing) | Low (automated) |
| User experience | Simple but difficult to manage at scale | Slightly complex | Smooth (can be automated) |