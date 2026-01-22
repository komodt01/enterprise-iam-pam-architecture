# Enterprise Privileged Access Management (PAM) Architecture

## Project Overview

This project implements **enterprise-grade Privileged Access Management (PAM)** using **HashiCorp Vault** and **SSH certificate-based authentication** to secure administrative access to cloud infrastructure. The design eliminates long-lived credentials, enforces time-bound access, and provides centralized auditability aligned with Zero Trust principles.

The implementation targets an **AWS EC2 (Ubuntu)** workload and demonstrates how Vault can function as a modern PAM control plane comparable to commercial solutions such as CyberArk or Delinea, while remaining cloud-agnostic.

---

## Why This Project Exists (Business Context)

Organizations routinely struggle with:

* Shared administrative accounts
* Long-lived SSH keys and passwords
* Limited auditability of privileged actions
* Manual access revocation during incidents or offboarding

This project addresses those risks by replacing static credentials with **ephemeral, centrally issued SSH certificates** that automatically expire and are fully auditable.

---

## Architecture Summary

### Control Plane

* **HashiCorp Vault (HCP)**

  * Acts as the PAM authority
  * Issues time-bound SSH certificates
  * Enforces role-based access
  * Maintains audit logs

### Data Plane

* **AWS EC2 (Ubuntu Linux)**

  * Trusts Vault as an SSH Certificate Authority
  * No stored private keys or passwords
  * Enforces least-privilege via sudo

### Admin Workstation

* **WSL (Linux on Windows)**

  * Vault CLI for access requests
  * SSH private key remains local

---

## Access Flow (High Level)

1. Administrator authenticates to Vault
2. Vault issues a short-lived SSH certificate
3. Administrator connects to EC2 using SSH certificate
4. Privileged commands are executed via sudo
5. Access expires automatically based on TTL

---

## Security Controls Implemented

* Passwordless SSH authentication
* Time-bound privileged access (TTL)
* Centralized role enforcement
* Server-side trust of Vault CA
* Elimination of long-lived credentials
* Full auditability of access issuance

---

## Threats Mitigated

* Credential theft
* Key sprawl
* Orphaned admin access
* Insider misuse
* Lack of forensic visibility

---

## Technologies Used

* HashiCorp Vault (HCP)
* AWS EC2 (Ubuntu Linux)
* OpenSSH
* Linux sudo
* WSL (Admin workstation)

Detailed explanations are provided in `technologies.md`.

---

## Repository Structure

```
enterprise-iam-pam-architecture/
├── README.md
├── business_requirements.md
├── architecture/
│   ├── level0.png
│   └── level1.png
├── implementation/
│   └── vault_pam_steps.md
├── compliance_mapping.md
├── technologies.md
└── lessons_learned.md
```

---

## Why HashiCorp Vault (Design Decision)

Vault was selected because it:

* Supports native SSH certificate issuance
* Enables Zero Trust, ephemeral access models
* Is cloud-agnostic
* Provides strong audit logging
* Mirrors the control patterns of enterprise PAM tools

This choice demonstrates architectural depth beyond simple credential management.

---

## Status

✅ Fully implemented and validated

---


