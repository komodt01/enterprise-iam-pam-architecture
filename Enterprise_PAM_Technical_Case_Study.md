# Technical Case Study: Enterprise Privileged Access Management with Short-Lived SSH Certificates

*Reference Implementation | HCP Vault | AWS EC2 | OpenSSH | Linux sudo*

## Case Study Scope

This case study examines how privileged administrative access to Linux infrastructure can be moved away from persistent SSH authorization toward centrally controlled, time-bound access.

For this scenario, I implemented the core access pattern using HCP Vault as an SSH Certificate Authority and an AWS EC2 Ubuntu instance as the protected administrative target. Vault was configured to sign an administrator's SSH public key and issue a short-lived SSH certificate. The EC2 host was configured to trust the Vault CA, and privileged actions were performed through the Linux sudo mechanism.

The implementation demonstrates the core PAM control flow: centralized certificate issuance, policy-based access constraints, time-limited authentication, host-side trust enforcement, privilege elevation, and generation of audit evidence across Vault and the Linux host.

This reference implementation focuses on the core privileged access pattern rather than representing a complete production PAM deployment. Enterprise identity federation, approval workflows, centralized SIEM integration, high availability, break-glass procedures, session recording, and more granular sudo policies are treated as production architecture considerations rather than implemented capabilities.

## Architecture Problem

Privileged administrative access creates a different level of risk than normal user access because a compromised administrator credential can provide an attacker with the ability to modify systems, change security configurations, access sensitive data, or disrupt services.

Traditional SSH administration often relies on persistent public keys configured on target systems. As environments grow, those keys can become difficult to govern. Security teams need to know who is authorized, how long that authorization should remain valid, how access is removed, and what evidence exists when privileged access occurs.

For this scenario, I wanted to change the trust model rather than simply add another control around static SSH access.

Instead of configuring the target server to permanently trust an administrator's SSH key, the server would trust a centralized Certificate Authority. An administrator would first obtain authorization from the central control plane, receive a short-lived certificate, and then use that certificate to authenticate to the target system.

The resulting design uses HCP Vault as the certificate issuance and policy control plane while the Linux host remains responsible for enforcing SSH authentication, local account authorization, and sudo privilege elevation.

The key architecture decision was therefore not simply to replace an SSH key with a certificate. It was to move privileged access authorization toward a centralized, policy-controlled, time-bound trust model.

## Architecture Decisions and Rationale

### 1. Use a Centralized SSH Certificate Authority

For this scenario, I chose HCP Vault to act as the SSH Certificate Authority and privileged access control plane. Rather than configuring individual administrator public keys as persistent trusted credentials on the EC2 instance, the host was configured to trust the Vault CA. Vault could then sign an administrator's public key and issue an SSH certificate with a defined lifetime.

The reason for this decision was to move the authorization model away from managing persistent per-user SSH keys on individual servers. Trust could instead be based on whether the administrator presented a valid certificate issued by an approved authority.

### 2. Make Privileged Access Time-Bound

The Vault SSH role was configured with a certificate TTL so that issued SSH certificates were valid only for a limited period.

I chose this approach because privileged access should not remain valid indefinitely simply because it was authorized at some point in the past. Requiring a new certificate after expiration creates a natural access lifecycle and reduces the usefulness of an older credential.

Certificate expiration does not address every access-removal scenario. The TTL limits how long the certificate can be used to establish new authenticated access, but it should not be treated as an immediate revocation or session-termination mechanism.

A production design would need to distinguish among certificate expiration, credential revocation, active-session termination, and account disablement. For example, if an administrator leaves the organization or a privileged endpoint is suspected of compromise, waiting for the certificate TTL to expire may not provide sufficiently rapid containment.

The appropriate termination mechanism would depend on the incident and the access path, but the architecture must support removal of privileged authority when the business or security condition changes rather than relying solely on natural certificate expiration.

### 3. Separate Access Issuance from Host Enforcement

I intentionally separated the architecture into two control areas. Vault controls whether an SSH certificate can be issued and applies constraints such as the permitted Linux user and certificate lifetime. The EC2 host independently validates the certificate against the trusted Vault CA and uses its local account and sudo configuration to control what the authenticated administrator can do.

This separation is important because successful authentication should not automatically mean unrestricted administrative authority.

### 4. Keep the Administrator's Private Key on the Workstation

The administrator's SSH private key remained on the WSL workstation. Vault signed the corresponding public key rather than receiving or distributing the private key.

This allowed Vault to control the authorization period through the signed certificate without becoming the repository for the administrator's SSH private key. The security of that workstation and private key therefore remains part of the overall trust model. In a production environment, endpoint security and protection of administrator credentials would need to be considered alongside the PAM architecture.

### 5. Use Managed Vault for the Reference Implementation

I used HCP Vault rather than operating a self-hosted Vault cluster. For this implementation, the purpose was to demonstrate the privileged access control pattern rather than design and operate the Vault platform itself. Using the managed service allowed the implementation to focus on certificate issuance, policy, TTL, host trust, and privileged access.

For a production deployment, the organization would still need to evaluate availability, identity integration, administrative access to Vault, recovery, monitoring, and break-glass requirements.

### 6. Keep Authentication and Privilege Elevation Separate

SSH certificate authentication established access to the Linux account, while sudo controlled elevation to privileged operations.

I chose to preserve this separation because PAM is not only concerned with whether an administrator can connect to a system. It also needs to consider what that administrator is authorized to do after authentication.

The reference implementation demonstrated privilege elevation through sudo. A production implementation would require more granular command restrictions and role design based on administrative responsibilities.

## Implemented Architecture

The reference implementation used three primary components: an administrator workstation, HCP Vault as the privileged access control plane, and an AWS EC2 Ubuntu instance as the protected target.

### Administrator Workstation

I used WSL on a Windows workstation as the administrative access point. The workstation contained the Vault CLI, the administrator's SSH private and public key pair, and the short-lived SSH certificate returned by Vault. The private key remained on the workstation. Vault received the public key for signing and returned the corresponding signed SSH certificate.

### HCP Vault - Control Plane

HCP Vault was configured with the SSH Secrets Engine operating as an SSH Certificate Authority. I configured the components necessary to establish the SSH Certificate Authority, define the SSH role used for privileged access, restrict the Linux user represented by the certificate, define the certificate TTL, sign the administrator's SSH public key, and issue the resulting short-lived SSH certificate.

Vault therefore controlled whether the administrator could obtain a certificate that the target host would trust.

### AWS EC2 Ubuntu - Enforcement Plane

An Ubuntu EC2 instance served as the protected Linux resource. OpenSSH on the instance was configured to trust the Vault CA using TrustedUserCAKeys. This allowed the SSH daemon to validate certificates signed by Vault rather than relying on a persistent administrator public key stored in authorized_keys.

The pamadmin Linux account was used as the administrative account for the implementation. After successful SSH authentication, privilege elevation was performed through sudo. This kept authentication to the host separate from authorization for privileged actions.

### Audit Evidence

Vault provided evidence associated with SSH certificate issuance, while the Ubuntu host generated authentication and sudo activity through its local authentication logs. These were separate sources of evidence in the reference implementation. I did not implement centralized correlation of the Vault and host logs.

In a production architecture, I would forward the relevant events to a centralized logging or SIEM platform so certificate issuance, host authentication, and privileged activity could be correlated and monitored together.

## Privileged Access Flow

### 1. Authenticate to Vault

The administrator first authenticated to HCP Vault from the WSL workstation using the Vault CLI. For this reference implementation, Vault token authentication was used. Federated authentication such as OIDC or SAML was not implemented and would be a production architecture consideration.

### 2. Request Privileged Access

The administrator submitted the SSH public key to the configured Vault SSH signing role. This moved the access decision away from the EC2 instance. Rather than permanently configuring the administrator's public key as trusted on the server, authorization was requested through Vault.

### 3. Apply Vault Role Constraints

Vault evaluated the configured SSH role before issuing the certificate. The role constrained elements of the access request including the permitted Linux user and certificate TTL. If the request satisfied the configured policy, Vault signed the administrator's public key.

### 4. Issue the Short-Lived SSH Certificate

Vault returned a signed SSH certificate to the administrator workstation. The administrator now had the SSH private key that remained on the workstation and the corresponding public key authorization represented by the short-lived Vault-signed certificate. The certificate was usable only during its configured validity period.

### 5. Authenticate to the EC2 Instance

The administrator initiated an SSH connection to the Ubuntu EC2 instance using the private key and Vault-issued certificate. The EC2 SSH daemon did not need a persistent administrator public key in authorized_keys for this access path. Instead, it validated the certificate against the Vault CA configured through TrustedUserCAKeys. A valid certificate issued by the trusted CA allowed the administrator to authenticate as the permitted pamadmin account.

### 6. Elevate Privileges

Authentication to the server did not itself provide unrestricted root access. Once connected as pamadmin, privileged operations were performed through sudo. Linux therefore remained responsible for enforcing privilege elevation after SSH authentication succeeded.

### 7. Generate Audit Evidence

Vault recorded certificate issuance activity. The Ubuntu host separately recorded SSH authentication and sudo activity in its local authentication logs. This provides evidence of both access issuance and subsequent use, although centralized correlation of those records was not implemented in this reference architecture.

### 8. Allow the Certificate to Expire

The SSH certificate became invalid when its configured TTL expired. After expiration, the certificate could no longer be used to establish a new SSH session, requiring the administrator to obtain another valid certificate for subsequent access.

This provides automatic expiration of the issued authorization. It should not be confused with active revocation before TTL expiration, which would require additional design consideration in a production PAM architecture.

## Security Controls and Trust Boundaries

The architecture depends on several distinct trust relationships. I treated these separately because compromising one part of the privileged access path can have a different impact than compromising another.

### Administrator Workstation Trust

The administrator workstation holds the SSH private key and is the point from which Vault access and SSH connections originate. The private key remained on the administrator workstation and only the public key was submitted to Vault for signing.

If the workstation or private key were compromised while a valid certificate was available, an attacker could potentially use those credentials during the certificate's remaining validity period. For production, endpoint protection, strong administrator authentication, device trust, and secure private-key storage would need to be part of the privileged access design.

### Vault Control Plane Trust

Vault is trusted to determine whether a certificate can be issued and to sign certificates using the SSH CA. Implemented controls included SSH certificate issuance, role-based restrictions, allowed Linux user constraints, certificate TTL, and audit evidence associated with issuance.

Because the target host trusts certificates signed by the Vault CA, compromise or misuse of the Vault control plane could undermine the privileged access model.

### Vault-to-Host CA Trust

The EC2 instance trusts the Vault SSH CA through its OpenSSH configuration. The host does not independently ask Vault whether a user should be admitted during each SSH connection. Instead, it validates whether the presented certificate was signed by the CA it already trusts and whether that certificate is valid for the requested access. The CA relationship itself therefore becomes a critical security control.

### EC2 Host Enforcement

The Ubuntu host remains responsible for enforcing the access represented by the certificate through OpenSSH trust of the Vault CA, certificate-based SSH authentication, the pamadmin account, local sudo privilege elevation, and host-level authentication and sudo logging.

Vault issuing a certificate does not itself determine every action the administrator can perform after connecting. Linux account configuration and sudo policy remain separate authorization controls.

### Time and Audit Boundaries

Certificate TTL limits the duration of the authorization for new authentication attempts. It reduces persistence associated with long-lived SSH authorization but does not solve every access-termination requirement.

Audit evidence is generated in more than one location. Vault provides evidence about certificate issuance, while the Linux host provides evidence about SSH authentication and sudo activity. The reference implementation did not centralize or correlate those records.

## Failure and Risk Scenarios

### Vault Is Unavailable

If Vault is unavailable, administrators cannot obtain new SSH certificates. An already-issued certificate may remain usable until it expires, but new privileged access requests depend on the availability of the Vault control plane. Production PAM would require defined availability, recovery, and break-glass requirements.

### Administrator Workstation or Private Key Is Compromised

Possession of the private key alone is not sufficient for this access path because the target also requires a valid Vault-signed certificate. However, compromise of both the private key and a currently valid certificate could allow access during the certificate's remaining validity period.

### Vault Authentication Credential Is Compromised

The reference implementation used Vault token authentication. If a token with sufficient privileges were compromised, an attacker might be able to request certificates within the permissions granted to that token. Production design should use federated identity, strong authentication, short-lived Vault authentication, and least-privilege policies as appropriate.

### Certificate Expires or Access Must Be Removed Early

Once the certificate reaches the end of its validity period, it cannot be used for a new SSH authentication attempt. This does not necessarily terminate an SSH session already established before expiration.

TTL also does not by itself provide immediate termination before expiration. A production architecture would need an explicit strategy for emergency access termination rather than relying solely on short certificate lifetimes.

### Policy, Host, sudo, and Logging Misconfiguration

An overly permissive Vault role could issue certificates for unintended users or excessive durations. Incorrect SSH configuration could block legitimate certificates or permit unintended authentication paths. Overly broad sudo permissions could provide more privilege than required. Loss or misconfiguration of Vault or host logs could create investigation gaps.

In production, these controls should be standardized, reviewed, tested, monitored for drift, and subject to appropriate change governance.

## Evidence and Validation

The reference implementation was validated by exercising the privileged access path and confirming that the individual control points worked together as intended.

### Vault SSH Certificate Issuance

I configured Vault's SSH Secrets Engine as an SSH Certificate Authority and created the role used to sign the administrator's SSH public key. Successful certificate issuance demonstrated that Vault could sign the submitted public key, apply configured access constraints and TTL, and return a certificate usable by the SSH client.

### Host Trust and Certificate-Based Authentication

The Ubuntu EC2 instance was configured with TrustedUserCAKeys so OpenSSH trusted the Vault SSH CA. From the WSL administrator workstation, I used the locally held SSH private key together with the Vault-issued certificate to authenticate to the EC2 instance as pamadmin. Successful authentication demonstrated that the Vault-to-host CA trust relationship was functioning.

### Privilege Elevation and Audit Evidence

After authentication, I used sudo from the pamadmin account to perform privileged operations, demonstrating that SSH authentication and operating-system privilege elevation remained separate control points.

Vault generated evidence associated with SSH certificate issuance, while the Ubuntu host recorded SSH authentication and sudo activity through its local authentication logging. The implementation did not centralize or correlate these records in a SIEM.

### Validation Boundary

The implementation validates the core PAM pattern: centralized certificate issuance, time-bound SSH authorization, CA-based host trust, certificate-based authentication, and separate privilege elevation.

It does not demonstrate a complete production PAM lifecycle. Identity federation, approval workflows, session recording, centralized event correlation, high availability, break-glass access, granular sudo authorization, and enterprise-scale onboarding and offboarding were not implemented and should not be interpreted as validated capabilities of this reference implementation.

## Production Architecture Considerations

### Enterprise Identity and Strong Authentication

For production, I would integrate Vault with the organization's enterprise identity provider using an appropriate federation mechanism such as OIDC. Depending on risk requirements, I would also evaluate MFA, device trust, conditional access, and stronger controls for highly privileged roles.

### Role, Privilege, and Approval Design

Different administrators should not automatically receive the same level of privilege simply because they require access to the same server. Vault roles, permitted Linux identities, certificate TTLs, and sudo policies should align with defined administrative responsibilities and least-privilege requirements.

For higher-risk administrative functions, I would evaluate approval workflows based on the risk of the requested privilege rather than adding the same workflow to every administrative action.

### Emergency Access, Availability, and Recovery

A production architecture would require a documented break-glass mechanism with tightly controlled access, strong monitoring, limited use, and post-event review. Because Vault becomes a critical dependency for new credentials, production requirements would also need to define acceptable availability, recovery objectives, backup and recovery procedures, and failure behavior.

### Certificate Lifetime and Emergency Termination

TTL selection creates a tradeoff between security exposure and operational friction. I would select TTLs based on the risk and operational requirements of the privileged role and separately define how access should be terminated when waiting for normal certificate expiration is not acceptable.

### Centralized Monitoring and Privileged Activity

For production, I would forward relevant Vault and Linux events to the organization's centralized logging or SIEM platform so security operations could correlate enterprise identity, privileged access request, certificate issuance, host authentication, and privilege elevation.

Depending on regulatory requirements and system sensitivity, I would also evaluate session recording, command-level auditing, or additional privileged activity monitoring.

### Administrator Endpoint and Network Access

Production architecture would need to consider endpoint security, device compliance, credential storage, malware protection, administrative workstation isolation, and whether privileged administration should be restricted to hardened or dedicated administrative endpoints.

I would also avoid assuming privileged SSH should be directly exposed through a public network path. Appropriate options could include private addressing and controlled administrative access through a bastion, AWS Systems Manager, or a Zero Trust Network Access solution, selected according to the organization's architecture and requirements.

### Governance and Change Control

Changes to Vault roles, certificate TTLs, allowed Linux users, CA configuration, authentication methods, and privileged policies can directly change who is able to obtain administrative access. I would therefore treat these as security-sensitive changes requiring appropriate authorization, review, testing, auditability, and configuration management.

## Architectural Lessons and Takeaways

### Centralizing Trust Changes the Access Model

The most significant change was moving away from persistent per-user SSH authorization on the target host. By configuring the host to trust the Vault CA, individual privileged access could be issued through a centralized control point and constrained by policy and time.

The architectural value of SSH certificates is not simply that certificates replace keys. The larger value is changing how privileged authorization is governed.

### Short-Lived Access Reduces Persistence but Does Not Solve the Entire Lifecycle

Certificate TTL provides a useful control because authorization naturally expires. At the same time, the implementation highlighted the distinction between expiration and active revocation. Production PAM still needs to address situations where access must be terminated immediately.

### Authentication and Privilege Are Different Decisions

Successfully authenticating to a server should not automatically answer what an administrator can do after connecting. Vault and SSH controlled the authentication path, while Linux and sudo remained responsible for privilege elevation. Least privilege has to continue beyond the initial login.

### The PAM Platform Becomes Critical Security Infrastructure

Centralizing certificate issuance reduces reliance on persistent host credentials, but it also increases the importance of the PAM control plane. If Vault is compromised, misconfigured, or unavailable, the impact can extend across systems that trust it. Production PAM therefore requires strong administrative controls, monitoring, resilience, recovery, and change governance.

### Auditability Requires More Than Generating Logs

The implementation produced evidence at both the Vault and Linux layers, but those records remained separate. Production monitoring should make it possible to correlate who was authorized, what credential was issued, where it was used, and what privileged activity followed.

### PAM Has to Fit the Organization Around It

A production PAM architecture cannot be designed independently of enterprise identity, endpoint security, network access, incident response, logging, operational support, and administrative processes. Moving the reference pattern into production would require understanding the organization's existing architecture, administrative roles, risk tolerance, availability requirements, security platforms, and operational processes.

## Conclusion

The reference implementation demonstrated how privileged Linux access can be moved from persistent host-level SSH authorization toward a centralized, policy-controlled, time-bound trust model.

HCP Vault provided SSH certificate issuance and policy controls, the EC2 host enforced CA-based authentication, and Linux sudo remained responsible for privilege elevation. Together, these components demonstrated the core separation between access issuance, authentication, and privileged authorization.

The larger architectural lesson is that PAM is not a single product or control. It is a chain of trust spanning identity, credential issuance, endpoint security, host enforcement, privilege management, logging, resilience, and operational governance.

The implementation validated the core of that chain while also identifying the additional controls and design decisions that would need to be evaluated before using the pattern as an enterprise production PAM architecture.
