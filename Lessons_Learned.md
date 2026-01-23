# lessons_learned.md — Enterprise PAM with HashiCorp Vault (SSH Certificates)

## Purpose
This document captures key takeaways from implementing a Vault-backed PAM architecture using SSH certificates on AWS EC2. It focuses on what worked, what broke, and what I would improve to make the design production-ready.

---

## What Went Well

### 1) SSH certificates are a clean PAM primitive
Replacing static SSH keys/passwords with Vault-signed SSH certificates delivered the core PAM outcomes:
- **No standing credentials on the host**
- **Time-bound access (TTL)**
- **Central policy enforcement**
- **Clear audit trail for access issuance**

### 2) Separation of control plane vs enforcement plane is real and explainable
This architecture cleanly separates responsibilities:
- **Vault** = issuance + policy + auditing (control plane)
- **Linux/sshd + sudo** = authentication validation + privilege execution (enforcement plane)

This separation made it easier to reason about risk and troubleshoot issues.

### 3) Host trust configuration is the “point of control”
Once the EC2 host trusts the Vault CA (`TrustedUserCAKeys`), the access model becomes enforceable and repeatable. That single SSHD directive is the key design anchor.

---

## What Broke or Slowed Things Down

### 1) Environment drift between WSL and EC2 is easy to underestimate
WSL is great as an admin workstation, but it introduced friction:
- `chmod` behavior on Windows-mounted paths (`/mnt/c/...`)
- different shells / sessions losing environment variables
- key files not in expected locations across sessions/users

**Fix:** Keep SSH keys inside Linux home (`~/.ssh`) and document environment setup explicitly.

### 2) Vault namespace configuration (HCP) is non-obvious
HCP Vault commonly uses a namespace (e.g., `admin/`). If `VAULT_NAMESPACE` isn’t set correctly, valid tokens can appear “invalid” or “permission denied.”

**Fix:** Document and standardize:
- `VAULT_ADDR`
- `VAULT_NAMESPACE`
- `VAULT_TOKEN` (temporary for setup only)

### 3) “Unknown role” is a predictable failure mode
Vault will not sign certificates unless a signing role exists and is correctly configured. Creating the SSH role required explicitly enabling user cert signing (`allow_user_certificates=true`).

**Fix:** Include a checklist section:
- SSH secrets engine enabled
- CA generated
- role created (with allow_user_certificates)
- policy attached
- sign endpoint validated

### 4) Path handling can break automation
Using `~/.ssh/id_rsa.pub` in the Vault CLI caused issues in some contexts. Absolute paths were more reliable.

**Fix:** Use absolute paths in reproducible docs and scripts, or ensure shell expansion is explicitly handled.

---

## Security Design Improvements (Production-Grade Enhancements)

### 1) Remove admin/root tokens from day-to-day use
During setup, admin tokens are required, but production should use:
- OIDC/SAML auth (Okta, Entra, etc.)
- short-lived Vault tokens
- least-privilege policies per role

### 2) Add stronger host-side restrictions
To harden the enforcement plane:
- disable password SSH authentication
- restrict allowed users explicitly
- tighten `sudo` permissions (command allow-listing via `/etc/sudoers.d/`)
- optionally require MFA upstream (via IdP → Vault)

### 3) Centralize log forwarding
Vault logs prove issuance; host logs prove execution. For enterprise readiness:
- forward `/var/log/auth.log` to CloudWatch/SIEM
- correlate certificate issuance events with host logins and sudo events

### 4) Add break-glass procedures
A production PAM program needs a documented break-glass process:
- who can use it
- when it’s allowed
- enhanced monitoring
- emergency token rotation

---

## Biggest Takeaway
Certificate-based PAM is a strong modern pattern because it converts privileged access into a **policy-driven, time-bound capability** rather than a static entitlement. The design is simple enough to explain to leadership, but deep enough to satisfy technical reviewers and audit requirements.

---
