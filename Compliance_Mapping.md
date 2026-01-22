# compliance_mapping.md — Enterprise Privileged Access Management (PAM)

## Purpose

This document maps the implemented PAM architecture to common security and risk frameworks. The intent is to demonstrate how architectural design decisions translate into compliance-aligned security controls, rather than treating compliance as a checklist exercise.

This mapping focuses on **intent, coverage, and evidence**, which is how auditors and security leadership evaluate real-world controls.

---

## Scope

* Privileged access to Linux infrastructure
* Administrative (root-level) access
* Authentication, authorization, and auditability
* Credential lifecycle management

Out of scope:

* End-user application access
* Windows PAM
* Approval-based workflows

---

## NIST SP 800-53 (Rev. 5) Mapping

### AC-2 — Account Management

**Control Intent:** Ensure accounts are managed, monitored, and deprovisioned appropriately.

**Implementation in This Project:**

* Privileged access is not tied to long-lived OS accounts alone.
* Access is granted dynamically through Vault-issued certificates.
* No standing credentials exist that require manual rotation or removal.

**Evidence:**

* Vault SSH roles define allowed users.
* Certificates expire automatically based on TTL.

---

### AC-6 — Least Privilege

**Control Intent:** Ensure users have only the access necessary to perform assigned tasks.

**Implementation in This Project:**

* Privileged access is role-based and time-bound.
* `pamadmin` is the only authorized administrative user.
* sudo controls privilege escalation explicitly.

**Evidence:**

* Vault role configuration (`allowed_users`, `ttl`).
* Linux sudo group membership.

---

### IA-2 — Identification and Authentication

**Control Intent:** Uniquely identify and authenticate users accessing systems.

**Implementation in This Project:**

* Authentication is certificate-based, not password-based.
* Certificates are cryptographically signed by Vault.
* Each certificate issuance is tied to a Vault identity.

**Evidence:**

* Vault audit logs for SSH certificate issuance.
* SSH CA trust configuration on EC2.

---

### IA-5 — Authenticator Management

**Control Intent:** Manage the lifecycle of authenticators to reduce compromise risk.

**Implementation in This Project:**

* No passwords or static SSH keys are stored on target hosts.
* SSH certificates expire automatically.
* Revocation is implicit via expiration, not manual action.

**Evidence:**

* Vault role TTL configuration.
* Absence of authorized_keys files on EC2.

---

### AU-2 — Event Logging

**Control Intent:** Ensure security-relevant events are logged.

**Implementation in This Project:**

* Vault logs all certificate issuance events.
* Linux logs SSH authentication and sudo activity.

**Evidence:**

* Vault audit log entries.
* `/var/log/auth.log` on EC2.

---

### AU-12 — Audit Generation

**Control Intent:** Generate audit records for defined events.

**Implementation in This Project:**

* Certificate issuance events include role, TTL, and identity.
* SSH login and sudo usage are recorded locally.

**Evidence:**

* Vault audit records.
* Host-level authentication logs.

---

## ISO/IEC 27001:2022 Mapping

### A.5.15 — Access Control

**Control Intent:** Implement access controls based on business and security requirements.

**Implementation in This Project:**

* Privileged access requires explicit Vault role authorization.
* Access is not persistent and must be re-issued.

---

### A.5.17 — Authentication Information

**Control Intent:** Protect authentication information from compromise.

**Implementation in This Project:**

* No passwords or static keys are used for privileged access.
* Private keys remain on the admin workstation.

---

### A.8.2 — Privileged Access Rights

**Control Intent:** Restrict and control privileged access rights.

**Implementation in This Project:**

* Administrative access is time-bound.
* Privilege escalation is controlled via sudo.

---

### A.8.15 — Logging

**Control Intent:** Produce and protect logs relevant to security events.

**Implementation in This Project:**

* Centralized issuance logging in Vault.
* Host-level logging of authentication and privilege escalation.

---

## CIS Critical Security Controls (v8)

### Control 5 — Account Management

* Eliminates shared and unmanaged privileged credentials.
* Enforces centralized access issuance.

### Control 6 — Access Control Management

* Enforces least privilege via role-based access.
* Prevents standing administrative access.

### Control 8 — Audit Log Management

* Provides clear audit trails for privileged access events.

---

## Summary

This PAM architecture aligns with multiple regulatory and security frameworks by:

* Eliminating long-lived credentials
* Enforcing least privilege
* Providing centralized, auditable access control
* Supporting Zero Trust access patterns

Rather than implementing controls solely to satisfy compliance, this design embeds compliance into the architecture itself.

---

## Future Compliance Enhancements

* Integrate identity federation (OIDC/SAML) to strengthen user attribution
* Forward Linux logs to a centralized SIEM
* Add approval workflows for high-risk roles
* Implement break-glass access procedures with enhanced monitoring
