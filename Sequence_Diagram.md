 Sequence Diagram
  
  actor Admin as Administrator (WSL)
  participant Vault as HCP Vault (SSH CA + Policies)
  participant EC2 as AWS EC2 (Ubuntu SSHD)
  participant Logs as Audit Logs (Vault + Host)

  Admin->>Vault: Authenticate to Vault (token / future OIDC)
  Admin->>Vault: Request SSH cert (ssh/sign/admin) with public key
  Vault-->>Vault: Evaluate policy + role constraints (allowed_users, TTL)
  Vault-->>Admin: Return signed SSH certificate (expires via TTL)

  Admin->>EC2: SSH using private key + Vault-signed cert
  EC2-->>EC2: Validate cert against TrustedUserCAKeys (Vault CA)
  EC2-->>Admin: Establish session as pamadmin

  Admin->>EC2: sudo privileged command
  EC2-->>EC2: Enforce sudo policy locally
  EC2-->>Logs: Record auth + sudo events (/var/log/auth.log)
  Vault-->>Logs: Record cert issuance event (role, TTL, identity)

## Sequence Diagram — What Each Step Proves

The sequence diagram illustrates the full lifecycle of privileged access and highlights how control, enforcement, and auditability are achieved without long-lived credentials.

### Step 1 — Administrator Authentication to Vault
**What it proves:**  
Privileged access is centrally brokered. Administrators must authenticate to a trusted authority before access can be issued, eliminating direct trust between users and hosts.

### Step 2 — Privileged Access Request
**What it proves:**  
Access is not implicit or permanent. Privileged access must be explicitly requested and evaluated against defined roles and policies.

### Step 3 — Policy Evaluation and Authorization
**What it proves:**  
Access decisions are policy-driven. Vault enforces role constraints such as allowed users and time-to-live (TTL) before issuing credentials.

### Step 4 — SSH Certificate Issuance
**What it proves:**  
Credentials are ephemeral. Vault issues a short-lived SSH certificate instead of distributing passwords or static keys.

### Step 5 — Certificate-Based Authentication to Host
**What it proves:**  
The host does not trust users directly. It trusts the Vault Certificate Authority, enforcing centralized control without storing secrets locally.

### Step 6 — Privileged Session Establishment
**What it proves:**  
Authentication and authorization are separated. Successful authentication establishes identity, not unlimited privilege.

### Step 7 — Privilege Elevation via sudo
**What it proves:**  
Least privilege is enforced at the operating system layer. Administrative actions require explicit elevation and are auditable.

### Step 8 — Audit Logging
**What it proves:**  
Privileged access is observable end-to-end. Vault logs credential issuance while the host logs authentication and privileged command execution.

### Step 9 — Automatic Access Revocation
**What it proves:**  
Access lifecycle management is enforced by design. When the certificate TTL expires, access is revoked automatically without manual intervention.
