

# 🔐 HashiCorp Vault for SSH Certificate Management

## 📌 Overview

HashiCorp Vault is a centralized system for managing secrets such as SSH credentials, API keys, and certificates. In this context, Vault is used as an **SSH Certificate Authority (CA)** to securely control access to a fleet of devices (e.g., Raspberry Pis, EC2 instances, and embedded systems).

---

## 🎯 Problem Statement

Traditional SSH access management involves:

- Manually copying public keys to each device
- Maintaining `authorized_keys` files across multiple systems
- No built-in expiration for access
- Difficult revocation when users leave

This approach becomes difficult to scale and poses security risks.

---

## ✅ Solution: Vault-Based SSH Access

Vault replaces static SSH key management with **dynamic, short-lived SSH certificates**.

### Key Concept

Instead of distributing user keys to every device:

- Devices trust a **single CA public key**
- Vault signs user keys to generate temporary certificates
- Access is granted based on Vault policies

---

## 🏗️ Architecture

```
User → Vault → Signed SSH Certificate → Target Device (Pi / EC2)
                         ↓
                 Device trusts Vault CA
```

---

## ⚙️ How It Works

### 1. User Generates SSH Key (one-time)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
```

---

### 2. User Requests Certificate from Vault

```bash
vault write ssh/sign/dev-role \
    public_key=@~/.ssh/id_ed25519.pub
```

Vault returns a signed certificate.

---

### 3. Save Certificate

```
~/.ssh/id_ed25519-cert.pub
```

---

### 4. SSH Access

```bash
ssh user@device-ip
```

Access is granted if:
- The certificate is valid
- The device trusts the Vault CA

---

## 🔧 Device Configuration (One-Time Setup)

On each device (Pi / EC2), configure SSH:

```
/etc/ssh/sshd_config
```

Add:

```
TrustedUserCAKeys /etc/ssh/vault_ca.pub
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

---

## 🔥 Benefits

### 🚀 Scalability
- Add new devices without modifying SSH access
- No need to distribute user keys

### 🔐 Security
- Short-lived certificates (e.g., 4–8 hours)
- No permanent credentials stored on devices

### 👥 Centralized Access Control
- Manage users and permissions in one place
- Revoke access instantly by disabling certificate issuance

### 📜 Policy-Based Access
- Define roles (e.g., dev, admin, read-only)
- Restrict access to specific systems or environments

### 🧪 Ideal for IoT / Distributed Systems
- Works well with fleets of Pis and remote devices
- Reduces operational overhead

---

## 🔄 Comparison with Traditional SSH

| Feature | Traditional SSH | Vault SSH |
|--------|----------------|----------|
| Key distribution | Manual | Centralized |
| Expiration | None | Built-in |
| Revocation | Difficult | Immediate |
| Scalability | Poor | Excellent |
| Security | Static keys | Dynamic certificates |

---

## 🧠 Key Takeaway

Vault transforms SSH access from:

> Static, manually managed keys

into:

> Dynamic, centrally controlled, short-lived certificates

---

## 🧭 When to Use

Use Vault if you have:

- Multiple devices (Pis, EC2, gateways)
- Multiple users
- Need for secure and auditable access
- Scaling requirements

---

## 🚫 When It May Be Overkill

- Single user environments
- Very small setups (1–2 machines)

---

## 📌 Summary

HashiCorp Vault provides a secure and scalable way to manage SSH access by:

- Acting as a centralized SSH Certificate Authority
- Eliminating manual key distribution
- Enforcing short-lived, policy-driven access
- Simplifying fleet-wide authentication

---