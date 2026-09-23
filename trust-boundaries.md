# Trust Boundaries — Enterprise Privileged Access Management Architecture

## Purpose

This document identifies the major trust and privilege boundaries demonstrated by the Enterprise Privileged Access Management reference implementation.

The implemented access path uses HCP Vault as an SSH Certificate Authority, an administrator operating from WSL, and an AWS EC2 Ubuntu instance as the protected administrative target.

The objective is not to treat PAM as a single authentication event. Privileged access crosses several distinct boundaries where identity, authorization, trust, or privilege changes.

The implemented flow is:

**Administrator → Vault Authentication → SSH Certificate Issuance → Administrative Connection → OpenSSH Validation → Linux Account → sudo Privilege Elevation → Privileged Activity**

Crossing one boundary successfully does not automatically authorize the next boundary.

---

## 1. Administrator → Administrative Workstation

The first boundary exists between the human administrator and the workstation used to initiate privileged access.

In the reference implementation, WSL provides the administrative environment and the administrator's SSH private key remains locally controlled.

The workstation therefore participates directly in the privileged access trust chain.

### Security Significance

A valid Vault-issued certificate does not protect the environment if an attacker also obtains control of the corresponding private key or administrative endpoint.

Production architecture would need to consider:

- Administrative endpoint hardening.
- Strong user authentication.
- Private-key protection.
- Endpoint monitoring.
- Device trust or posture.
- Separation of privileged and general-purpose administrative activity where appropriate.

The administrative endpoint should not be treated merely as a client device. It is part of the privileged security boundary.

---

## 2. Administrator → Vault Authentication

Before privileged authorization can be issued, the administrator must establish an authenticated identity with Vault.

This is a separate security decision from SSH authentication to the target system.

### Implemented Boundary

The reference implementation authenticates the administrator to HCP Vault before allowing interaction with the SSH signing capability.

Vault authentication establishes who is requesting privileged authorization.

### Production Considerations

A production implementation could integrate Vault with enterprise identity through mechanisms such as OIDC or SAML and apply stronger controls such as MFA, conditional access, or device requirements.

The important architecture distinction is:

**Authentication to Vault establishes the requester identity. It does not by itself authorize unrestricted privileged access to target systems.**

---

## 3. Vault Identity → SSH Certificate Issuance

After authentication, a second boundary determines whether Vault will issue privileged authorization.

The administrator submits an SSH public key to the configured Vault signing role.

Vault evaluates the request against the signing role and its constraints before issuing a certificate.

### Implemented Enforcement

The signing role constrains elements such as:

- Permitted Linux user.
- Certificate lifetime.
- Certificate issuance through the configured SSH role.

Vault therefore acts as the centralized authorization point for issuance of temporary SSH credentials.

This separates:

**Who the requester is**

from

**What privileged authorization may be issued to that requester.**

---

## 4. Vault Control Plane → SSH Certificate Authority

The SSH Certificate Authority represents a particularly sensitive trust boundary.

Target hosts trust certificates signed by the Vault SSH CA.

This means the CA has greater authority than an individual administrator certificate.

### Security Significance

Compromise of one administrator's private key and certificate affects that administrator's authorization within the certificate's permitted scope and lifetime.

Compromise or misuse of the trusted SSH CA could potentially allow unauthorized certificates to be created for multiple protected systems that trust that CA.

The CA therefore represents a high-value security asset.

Production architecture should protect:

- CA configuration.
- Signing roles.
- Allowed principals.
- Certificate TTL policies.
- Vault administrative permissions.
- Changes to trusted CA relationships.
- Audit records associated with certificate issuance.

Administration of the CA should be treated differently from routine use of the CA to request authorized access.

---

## 5. Vault-Issued Certificate → Administrator Private Key

Vault signs the administrator's public key.

The corresponding private key remains on the administrator workstation.

The certificate and private key therefore represent separate components of the authentication mechanism.

### Security Significance

Vault does not need possession of the administrator's SSH private key to issue the certificate.

This reduces the need to centralize private-key storage but places responsibility for protecting the private key on the administrative endpoint.

Possession of a certificate alone should not provide access without the corresponding private key.

The architecture therefore distributes trust between:

**Centralized authorization through Vault**

and

**Private-key possession on the administrative endpoint.**

---

## 6. Administrative Workstation → Protected Network Path

After receiving a certificate, the administrator must reach the protected Linux target.

Network reachability and privileged authorization are separate security decisions.

### Implemented Scope

The reference implementation validates the certificate-based SSH access path to the EC2 Ubuntu target.

### Production Considerations

A production architecture could restrict the administrative path through mechanisms such as:

- Private connectivity.
- Bastion infrastructure.
- AWS Systems Manager.
- Zero Trust Network Access.
- Administrative network segmentation.
- Firewall policy.
- Controlled management networks.

Network access to a server should not itself establish privileged trust.

Likewise, possession of valid privileged credentials does not imply that every network path to every administrative target should be available.

---

## 7. Administrative Connection → OpenSSH Host Enforcement

The target host independently determines whether the presented SSH certificate is trusted.

OpenSSH validates the certificate against the configured Vault SSH CA.

This creates an important separation between centralized authorization and local enforcement.

### Centralized Control

Vault:

- Issues the certificate.
- Applies signing-role constraints.
- Defines certificate lifetime.

### Local Enforcement

The Linux host:

- Trusts the configured SSH CA.
- Validates the presented certificate.
- Determines whether authentication to the requested account is permitted.

The target does not simply trust the administrator because Vault exists.

It trusts a specific cryptographic authorization issued by an approved CA.

---

## 8. SSH Authentication → Linux Account

Successful certificate validation establishes access to a permitted Linux account.

This represents another boundary because authentication to the operating system does not automatically imply unrestricted operating-system privilege.

In the reference implementation, certificate-based authentication provides access to the `pamadmin` account.

The authenticated Linux identity becomes the basis for subsequent host authorization decisions.

---

## 9. Linux Account → sudo Privilege Elevation

Privilege elevation is deliberately separated from SSH authentication.

Successful SSH authentication answers:

**May this identity access this Linux account?**

`sudo` answers a different question:

**What privileged operating-system actions may this authenticated account perform?**

This separation prevents privileged access from being treated as a single binary decision.

### Implemented Boundary

Linux `sudo` provides the local privilege-elevation mechanism after successful SSH authentication.

### Production Considerations

A production environment could implement more granular authorization based on:

- Administrative role.
- Permitted commands.
- Target system.
- Business function.
- Environment.
- Risk level.
- Approval state.

Authentication and privilege elevation should remain separate enforcement decisions.

---

## 10. Standard Administration → Privileged Administration

Privileged access represents a different trust level from ordinary user or administrative activity.

An identity authorized for general enterprise access should not automatically receive the ability to administer infrastructure.

Likewise, an administrator authorized for one system or administrative function should not automatically receive equivalent authority elsewhere.

Production PAM architecture should therefore distinguish among:

- Standard workforce identity.
- Administrative identity.
- Privileged role.
- Target-system authorization.
- Elevated operating-system privilege.
- PAM platform administration.

This limits the effect of a compromised identity and reduces unnecessary privilege concentration.

---

## 11. PAM User → PAM Administrator

Using the privileged access system and administering the privileged access system are different authority levels.

An administrator who requests an SSH certificate should not automatically have authority to:

- Modify signing roles.
- Increase certificate TTLs.
- Add permitted principals.
- Change authentication methods.
- Alter the SSH CA.
- Disable auditing.
- Modify privileged-access policy.

This separation becomes especially important because changing the control plane can affect every system relying on it.

Production design should therefore separate routine privileged-access consumption from administration of the PAM platform itself.

---

## 12. Vault Policy → Host Privilege Policy

Vault and Linux enforce different parts of the privileged-access decision.

Vault determines what temporary SSH authorization may be issued.

The target host determines whether the certificate is trusted and what the authenticated Linux account may subsequently do.

This creates two distinct policy domains:

**Central privileged-access policy**

and

**Local target-system privilege policy.**

Neither should be assumed to replace the other.

A restrictive Vault certificate does not correct an overly permissive `sudo` configuration, and a restrictive `sudo` configuration does not compensate for uncontrolled certificate issuance.

Both layers contribute to the effective privileged-access policy.

---

## 13. Certificate Lifetime → Active Access Termination

Certificate expiration creates a time boundary.

Once the certificate expires, it cannot be used for a new SSH authentication attempt.

However, expiration is not equivalent to immediate revocation or termination of an already established session.

This distinction matters when an organization needs to terminate access immediately because of:

- Suspected credential compromise.
- Administrator termination.
- Incident response.
- Privilege removal.
- Emergency containment.

Production architecture therefore requires an access-termination strategy beyond relying solely on short certificate TTLs.

---

## 14. Privileged Activity → Audit Evidence

Privileged access generates evidence across multiple systems.

### Vault Evidence

Vault can record activity associated with certificate issuance.

### Host Evidence

The Linux host records SSH authentication and `sudo` activity.

These records represent different portions of the privileged-access lifecycle.

A production environment should correlate them so investigators can reconstruct:

**Who requested access → What authorization was issued → Which target was accessed → What privileged activity occurred → When the authorization expired or was terminated**

The reference implementation demonstrates evidence at both layers but does not implement centralized correlation.

---

## 15. Security Control → Security Administrator

The systems protecting privileged access must themselves be protected from unauthorized modification.

This includes:

- Vault configuration.
- SSH CA configuration.
- Signing roles.
- Authentication methods.
- Linux trusted CA configuration.
- SSH configuration.
- `sudo` policy.
- Logging configuration.

An attacker who cannot bypass certificate validation directly may instead attempt to modify the control that performs the validation.

Production architecture should therefore govern who may change security controls and ensure those changes generate protected audit evidence.

---

## Failure and Bypass Paths

Trust-boundary analysis should also consider how the architecture behaves when a control is compromised or unavailable.

### Compromised Administrator Workstation

An attacker controlling the administrative endpoint may gain access to locally stored private-key material or active administrative sessions.

Controls outside the PoC would be required to reduce this risk.

### Compromised Administrator Identity

A compromised enterprise or Vault identity could potentially request privileged authorization permitted to that identity.

Strong authentication, constrained roles, monitoring, and approval requirements may reduce this risk.

### Compromised Private Key

A stolen private key alone does not create permanent authorization if access also requires a valid Vault-issued certificate.

The certificate lifetime limits how long an already-issued certificate can be reused for new authentication.

### Compromised Vault Authentication

Compromise of Vault authentication does not necessarily provide unrestricted target access if certificate issuance remains constrained by Vault policy and host enforcement.

### Compromised SSH CA

Compromise of the trusted CA is significantly more serious because protected hosts rely on that CA when deciding which certificates to trust.

CA protection, administrative separation, recovery, and trust replacement therefore require explicit production design.

### Overly Permissive Signing Role

A signing role that permits excessive principals or certificate lifetimes could weaken the intended privileged-access boundary even if Vault itself remains uncompromised.

### Overly Permissive sudo Policy

Strong SSH authentication does not prevent excessive privilege if the authenticated account receives unrestricted local elevation.

### Logging Failure

Failure to collect Vault or host evidence may not prevent authentication but reduces the organization's ability to investigate and govern privileged access.

Production environments should determine whether critical audit failures should alert, block certain operations, or trigger another defined response.

### Vault Unavailability

Centralized certificate issuance introduces dependency on Vault availability.

A production architecture must balance PAM availability with the risk of creating uncontrolled fallback access.

Emergency access should be explicitly designed rather than emerging as an informal bypass when the PAM platform is unavailable.

---

## Implemented vs. Production Trust Boundaries

### Demonstrated by the Reference Implementation

The project demonstrates:

- Administrator interaction with HCP Vault.
- Vault authentication before certificate issuance.
- Policy-controlled SSH certificate issuance.
- Short-lived SSH authorization.
- Separation of certificate issuance from private-key possession.
- Host-side validation of the Vault SSH CA.
- Separation of SSH authentication from `sudo` privilege elevation.
- Vault certificate-issuance evidence.
- Linux SSH and `sudo` evidence.

### Additional Production Architecture Considerations

The project does not claim implementation of:

- Enterprise identity federation.
- MFA or device trust.
- Automated approval workflows.
- Just-in-time access orchestration beyond certificate TTL.
- Granular enterprise `sudo` policy.
- Dedicated privileged-access workstations.
- Centralized session recording.
- Centralized SIEM correlation.
- Immediate certificate revocation or session termination.
- Break-glass access.
- High-availability PAM architecture.
- Enterprise administrative network segmentation.
- Full separation of PAM administration and PAM consumption.

These would require additional design based on enterprise risk, operational requirements, and existing security architecture.

---

## Core Architecture Principle

Privileged access should not be represented as one successful login.

It is a chain of independently controlled trust decisions:

**Verify the Administrator → Authorize Temporary Access → Protect the Credential → Restrict the Network Path → Validate at the Target → Authorize Privilege → Record the Activity**

Compromise of one layer should not automatically eliminate every subsequent security boundary.
