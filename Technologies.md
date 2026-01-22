# technologies.md — Enterprise PAM with HashiCorp Vault (SSH Certificates)

## Purpose of This File
This document explains the technologies used in this Privileged Access Management (PAM) project, what they are, how they work, and why they were selected. The goal is to show architectural intent (not just implementation steps).

---

## HashiCorp Vault (HCP Vault)

### What it is
HashiCorp Vault is a secrets and identity-based access platform that can issue, broker, and audit access to sensitive systems. In this project, Vault functions as the **PAM control plane**.

### How it works (in this project)
- Vault runs an **SSH Secrets Engine** configured as an **SSH Certificate Authority (CA)**.
- An administrator requests access by having Vault **sign their SSH public key**.
- Vault returns a **short-lived SSH certificate** with a defined TTL.
- The target Linux host trusts the Vault CA public key, so it accepts only certificates signed by Vault.

### Why it’s used
- Replaces long-lived credentials with **ephemeral access** (Zero Trust pattern).
- Centralizes policy enforcement and auditing.
- Demonstrates a modern, cloud-agnostic PAM control model comparable to enterprise PAM platforms (e.g., CyberArk/Delinea) while remaining API-driven and automation-friendly.

### Limitations / tradeoffs
- Requires careful policy design (roles, TTLs, allowed users).
- Root/admin tokens should not be used long-term; production patterns require scoped auth (OIDC/SAML) and least privilege tokens.
- Availability of Vault becomes critical; HA and break-glass paths matter in real deployments.

---

## HCP (HashiCorp Cloud Platform)

### What it is
HashiCorp Cloud Platform provides a managed control plane for HashiCorp products, including managed Vault clusters.

### How it works (in this project)
- HCP hosts the Vault cluster so Vault does not need to be installed, patched, or operated manually.
- Administrative access to Vault is done via Vault tokens in the `admin` namespace (HCP’s operational model).

### Why it’s used
- Reduces operational overhead so the project focuses on PAM outcomes (time-bound access, auditability).
- Reflects a realistic enterprise pattern: “managed control plane, secure access enforcement.”

### Limitations / tradeoffs
- Namespaces and token handling differ from self-managed Vault and must be explicitly configured (e.g., `VAULT_NAMESPACE=admin`).
- Some enterprise workflows (advanced auth, approvals) may require additional configuration or integrations.

---

## SSH (OpenSSH)

### What it is
SSH is the standard protocol for secure remote administration of Linux systems. OpenSSH is the most common implementation.

### How it works (in this project)
- The admin uses SSH with:
  - a local **private key**
  - and a **Vault-issued SSH certificate** (`id_rsa-cert.pub`)
- The Linux host validates the certificate against Vault’s CA trust.

### Why it’s used
- SSH is the default administrative control path for Linux.
- SSH certificates enable **passwordless** and **time-bound** access without sharing secrets.

### Limitations / tradeoffs
- Host SSH daemon must be configured carefully (CA trust, allowed users).
- If teams rely on shared accounts or static keys, operational change management is required.

---

## SSH Certificates (Certificate-Based Authentication)

### What it is
SSH certificates are signed assertions that a given public key is authorized to authenticate for a specific user, for a specific duration.

### How it works (in this project)
- Vault signs the admin’s SSH public key using the Vault CA.
- The certificate includes:
  - the allowed Linux user (e.g., `pamadmin`)
  - an expiration (TTL)
- After TTL expires, access fails automatically.

### Why it’s used
- Eliminates long-lived credentials and “key sprawl.”
- Enables strong access governance and automated revocation via expiration.
- Supports Zero Trust: access is not permanent; it must be re-requested and re-issued.

### Limitations / tradeoffs
- Requires a reliable certificate authority and consistent issuance policy.
- Without strong identity federation (Okta/OIDC), token-based access can become a weak link.

---

## Linux PAM Concepts (Privileged Access Context)

### What it is
In an enterprise context, “PAM” refers to privileged access control, not just Linux Pluggable Authentication Modules. This project focuses on **privileged access governance** (who gets admin access, for how long, and how it’s audited).

### How it works (in this project)
- Vault governs *authentication and issuance* of privileged access.
- Linux enforces *authorization and privilege escalation* via OS accounts and sudo.

### Why it’s used
- Demonstrates a real separation of concerns:
  - Vault as the control plane (issuance + audit)
  - Linux as the enforcement plane (user + sudo)

---

## sudo (Privilege Elevation)

### What it is
`sudo` allows approved users to run commands as another user (typically root) with auditing of command execution.

### How it works (in this project)
- The `pamadmin` account is allowed to elevate privileges using sudo.
- This demonstrates the “privileged action” portion of PAM (not just login).

### Why it’s used
- PAM is not only “getting onto the box”; it is “what can you do once you’re there.”
- sudo demonstrates least privilege potential (command allow-listing) and an audit trail on the host.

### Limitations / tradeoffs
- In production, `pamadmin` should typically be limited to approved commands via `/etc/sudoers.d/` and integrated with centralized logging.

---

## AWS EC2 (Ubuntu 22.04)

### What it is
Amazon EC2 provides compute instances in AWS. Ubuntu 22.04 LTS is a stable Linux distribution commonly used for production workloads.

### How it works (in this project)
- EC2 hosts the PAM-protected Linux resource.
- OpenSSH is configured to trust Vault’s CA via:
  - `TrustedUserCAKeys /etc/ssh/trusted-user-ca-keys/vault.pub`

### Why it’s used
- Mirrors real-world infrastructure where Linux workloads are administered via SSH.
- Provides a realistic target for demonstrating certificate-based privileged access.

### Limitations / tradeoffs
- Public IP access should be restricted (security groups, IP allow-lists, or a bastion).
- For production, prefer private subnets + managed access paths (SSM, bastion, ZTNA).

---

## WSL (Windows Subsystem for Linux)

### What it is
WSL provides a Linux userland environment on Windows. In this project it acts as the administrator workstation.

### How it works (in this project)
- Vault CLI runs in WSL to request/sign certificates.
- SSH private keys and certificates are stored locally under `~/.ssh`.
- WSL is not used as the PAM target host because it does not reliably represent a full Linux service model for sshd.

### Why it’s used
- Mirrors a real admin workstation pattern.
- Enables quick iteration without requiring a dedicated Linux desktop.

### Limitations / tradeoffs
- WSL is not a substitute for a true Linux server for service-level testing.
- Environment variables and shell context must be managed carefully across sessions.

---

## Vault CLI

### What it is
The Vault CLI is the command-line interface used to authenticate, configure, and request access from Vault.

### How it works (in this project)
- Used to enable SSH secrets engine, configure CA, create roles, and sign public keys.

### Why it’s used
- Demonstrates automation-friendly and reproducible access workflows.
- Enables clear evidence of access issuance and controls.

### Limitations / tradeoffs
- CLI access must be secured (scoped tokens, least privilege).
- For production, prefer federated auth and short-lived Vault tokens.

---

## Key Design Choices (What Was Chosen and Why)

### Choice 1: SSH Certificates vs. Password Vaulting
SSH certificates were chosen to eliminate static credentials entirely and enforce automatic revocation via TTL.

### Choice 2: Managed Vault (HCP) vs. Self-Hosted Vault
Managed Vault was chosen to reduce operational overhead and keep the project focused on security architecture outcomes.

### Choice 3: Separate Control Plane and Enforcement Plane
Vault is the control plane (issuance + audit). Linux is the enforcement plane (user + sudo). This models real enterprise PAM patterns.

---

## Next Enhancements (Planned)
- Integrate Okta (OIDC/SAML) for Vault authentication (remove admin tokens from daily operations)
- Restrict `sudo` commands for `pamadmin` via `/etc/sudoers.d/`
- Forward Linux auth logs to centralized logging (CloudWatch / SIEM)
- Add break-glass process documentation and token rotation controls
