# Enterprise Privileged Access Management (PAM) Architecture

> **Project Scope Note:** This project is a reference implementation
> demonstrating a core Privileged Access Management pattern. It
> validates centralized SSH certificate issuance, time-bound privileged
> access, host-side certificate trust, Linux privilege elevation, and
> generation of audit evidence. It is not presented as a complete
> production PAM deployment. Enterprise identity federation, approval
> workflows, centralized SIEM correlation, high availability,
> break-glass access, session recording, granular privilege policies,
> and other production controls would require additional architecture
> and operational design.

## Project Overview

This project demonstrates a Privileged Access Management (PAM)
architecture for reducing reliance on persistent SSH authorization when
administrators access Linux infrastructure.

For this reference implementation, I used HCP Vault as an SSH
Certificate Authority and an AWS EC2 Ubuntu instance as the protected
administrative target. Instead of configuring a persistent administrator
SSH key on the server, Vault signs the administrator's public key and
issues a short-lived SSH certificate. The EC2 host trusts the Vault
Certificate Authority and validates the certificate before allowing
access.

Once authenticated, privileged actions are controlled separately through
Linux `sudo`, keeping authentication and privilege elevation as distinct
security decisions.

The project demonstrates a core PAM pattern built around centralized
trust, policy-controlled certificate issuance, time-bound access,
host-side enforcement, and audit evidence across the certificate
issuance and Linux host layers.

## Architecture Approach

The design separates privileged access into a centralized issuance layer
and a host enforcement layer.

The access flow is:

1.  An administrator authenticates to HCP Vault from the WSL
    administrative workstation.
2.  The administrator submits an SSH public key to the configured Vault
    signing role.
3.  Vault applies the role constraints and issues a short-lived SSH
    certificate.
4.  The administrator connects to the AWS EC2 Ubuntu instance using the
    locally held private key and Vault-issued certificate.
5.  OpenSSH validates the certificate against the trusted Vault SSH
    Certificate Authority.
6.  The administrator authenticates as the permitted Linux account.
7.  Privileged actions are performed separately through `sudo`.
8.  Vault records certificate issuance activity, while the Linux host
    records SSH authentication and `sudo` activity.
9.  When the certificate TTL expires, the certificate can no longer be
    used for new SSH authentication.

This approach moves the trust decision away from persistent per-user SSH
authorization on individual servers. The target host instead trusts an
approved Certificate Authority, while Vault provides the centralized
control point for issuing time-limited privileged access.

## What I Implemented

The reference implementation included:

-   HCP Vault as the managed PAM control plane.
-   Vault SSH Secrets Engine configured as an SSH Certificate Authority.
-   An SSH signing role with permitted Linux user and certificate TTL
    constraints.
-   Vault signing of an administrator's SSH public key.
-   Short-lived SSH certificate issuance.
-   AWS EC2 Ubuntu as the protected Linux target.
-   OpenSSH configured to trust the Vault SSH CA.
-   Certificate-based authentication to the `pamadmin` Linux account.
-   Local `sudo` privilege elevation after successful SSH
    authentication.
-   WSL as the administrator workstation with the SSH private key
    retained locally.
-   Validation of Vault certificate issuance and successful
    certificate-based SSH access.
-   Vault issuance evidence and Linux SSH/`sudo` authentication logs.

For this access path, the target host does not require a persistent
administrator public key in `authorized_keys`. The administrator's
private key remains on the workstation; Vault signs the corresponding
public key and controls the lifetime of the resulting authorization
through the SSH certificate.

## Security Architecture Decisions

### Centralized Trust

The EC2 host trusts the Vault SSH CA rather than maintaining persistent
per-user SSH authorization for this access path. This shifts privileged
access issuance toward a centralized policy-controlled trust model.

### Time-Bound Authorization

Vault-issued certificates have a defined TTL. Once a certificate
expires, it cannot be used for a new SSH authentication attempt.
Certificate expiration reduces persistence but is not the same as active
revocation or immediate session termination.

### Separation of Authentication and Privilege

The SSH certificate determines whether the administrator can
authenticate to the Linux account. Linux `sudo` separately controls
privilege elevation. This preserves an important distinction between
gaining access to a system and being authorized to perform privileged
actions.

### Separation of Control and Enforcement

Vault controls certificate issuance and associated constraints. The
Linux host independently validates the certificate against its trusted
CA and enforces local account and privilege controls.

### Private-Key Ownership

The administrator's SSH private key remains on the WSL workstation.
Vault receives the public key for signing rather than storing or
distributing the private key. Endpoint and private-key security
therefore remain part of the overall privileged access trust model.

## Security and Risk Coverage

The implementation demonstrates controls that help reduce risks
associated with:

-   Persistent SSH authorization.
-   Administrator key sprawl across target systems.
-   Privileged authorization that remains valid indefinitely.
-   Decentralized host-by-host trust management.
-   Lack of visibility into when temporary privileged authorization is
    issued.
-   Conflating system authentication with privileged operating-system
    authorization.

The architecture does not eliminate all privileged access risk. The
security of the administrator workstation, Vault authentication, Vault
policy, the SSH CA, Linux configuration, `sudo` policy, logging, and the
administrative network path all remain important parts of the trust
chain.

## Audit Evidence

Audit evidence is generated at two different layers.

**Vault** provides evidence associated with SSH certificate issuance,
including the signing operation, role, and certificate lifetime.

**Ubuntu** provides host-side evidence associated with SSH
authentication and `sudo` activity.

The reference implementation does not centrally correlate these records.
In a production environment, I would forward relevant Vault and host
events to an approved centralized logging or SIEM platform so access
issuance and subsequent privileged activity could be correlated.

## Production Architecture Considerations

Moving this pattern into production would require additional decisions
based on the organization's existing identity, security, infrastructure,
compliance, and operational environment.

Areas I would evaluate include:

-   Enterprise identity federation such as OIDC or SAML rather than
    routine administrative token authentication.
-   Strong authentication and potentially MFA, device trust, or
    conditional access for privileged roles.
-   Role design and more granular `sudo` authorization.
-   Risk-based approval or just-in-time workflows for higher-risk
    privileged access.
-   Controlled break-glass and emergency access.
-   Vault availability, recovery, backup, and operational ownership.
-   An emergency access-termination strategy for cases where waiting for
    certificate expiration is not acceptable.
-   Centralized SIEM collection and correlation of Vault and host
    activity.
-   Session recording or additional command-level monitoring where
    required.
-   Hardened administrative endpoints and stronger private-key
    protection.
-   Controlled administrative network paths such as private
    connectivity, bastion access, AWS Systems Manager, or Zero Trust
    Network Access where appropriate.
-   Governance and change control for Vault roles, TTLs, allowed users,
    CA configuration, authentication methods, and privileged policies.

These are production architecture considerations, not capabilities
claimed as implemented in this reference project.

## Technologies Used

-   **HCP Vault** --- managed Vault control plane and SSH Certificate
    Authority
-   **Vault SSH Secrets Engine** --- SSH certificate signing
-   **AWS EC2** --- protected Linux compute target
-   **Ubuntu Linux** --- target operating system
-   **OpenSSH** --- certificate-based SSH authentication
-   **Linux sudo** --- local privilege elevation
-   **Windows Subsystem for Linux (WSL)** --- administrative workstation
    environment
-   **Vault CLI** --- Vault configuration and certificate request
    workflow

## Case Studies

This repository includes two views of the architecture:

-   **Technical Case Study** --- detailed architecture decisions,
    implementation flow, trust boundaries, failure scenarios, validation
    evidence, and production considerations.
-   **Executive Case Study** --- business-focused explanation of the
    privileged access problem, risk reduction approach, operational
    value, and production considerations.

## Key Takeaway

The project demonstrates that PAM is more than storing administrator
credentials. The larger architecture problem is determining where
privileged trust is established, how access is authorized, how long that
authorization remains valid, what the target system independently
enforces, and what evidence is generated when privileged access is
issued and used.

By moving from persistent host-level SSH authorization toward centrally
issued, short-lived SSH certificates, this reference implementation
demonstrates a more controlled and governable privileged access model
while keeping authentication, host enforcement, and privilege elevation
as distinct security decisions.
