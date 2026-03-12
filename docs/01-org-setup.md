# AWS Organization Setup (Management Account Foundation)

This document describes the foundational setup of the AWS Organization, including the management account, sandbox accounts, OU structure, and the identity model that supports secure administration. The goal is to establish a minimal but architecturally correct landing zone that enforces MFA-backed administration and clean trust boundaries.

---

## Organization Overview

The baseline consists of:

- A **management account** (payer + identity root)
- Two **sandbox accounts** for experimentation and testing
- A simple **Organizational Unit (OU)** structure
- **Service Control Policies (SCPs)** enabled at the organization level
- A dedicated **admin-user** IAM identity
- A privileged **AdminAccess-mylab** role for all administrative actions

This structure mirrors the identity and governance patterns used in real landing zones while remaining intentionally lightweight.

---

## Management Account Setup

### Root Account Hardening
The management account root user is secured with:

- MFA enabled
- Strong, unique password
- No programmatic access keys
- Root usage restricted to:
  - Billing configuration
  - Organization creation
  - Account creation
  - Break-glass scenarios

Root is never used for daily administration.

### Create the IAM User: admin-user
A single IAM user is created to serve as the human login identity:

- Username: **admin-user**
- Permissions: **none**
- MFA: **required**
- Purpose: authenticate → then assume the admin role

This separation ensures that authentication and authorization remain distinct.

---

## Organizational Structure

### Create the AWS Organization
Using the management account root:

1. Navigate to **AWS Organizations**
2. Choose **Create Organization**
3. Select **Enable all features**

This enables SCPs, consolidated billing, and full governance capabilities.

### Organizational Units (OUs)
A minimal OU structure is created:

```
Root
└── Sandbox
    ├── sandbox1
    └── sandbox2
```

This provides a clean separation for experimentation accounts and allows SCPs to be applied at the OU level.

### Create Sandbox Accounts
From the management account:

1. AWS Organizations → **Accounts** → **Add an AWS account**
2. Create:
   - **sandbox1**
   - **sandbox2**
3. Assign both to the **Sandbox OU**

Each sandbox account automatically receives the AWS-managed role:

```
OrganizationAccountAccessRole
```

This role is used for cross-account administration.

---

## Service Control Policies (SCPs)

SCPs are enabled at the organization level. Even without custom SCPs applied, enabling SCPs ensures:

- All accounts are governed by the organization
- Future guardrails can be added without restructuring
- The identity model aligns with enterprise landing zone patterns

No restrictive SCPs are applied initially to avoid interfering with setup.

---

## Browser Profile Isolation (Operational Safety)

To prevent accidental cross-account actions in the console:

- Each AWS account is assigned a dedicated browser profile
- Profiles use distinct colors and icons
- This prevents session contamination and reduces operational risk

This practice mirrors enterprise operational safety guidelines.

---

## Identity Model Summary

The management account implements a clean identity chain:

```
admin-user (MFA)
    ↓
AssumeRole: AdminAccess-mylab
    ↓
AssumeRole: OrganizationAccountAccessRole (sandbox1 or sandbox2)
```

This ensures:

- MFA is required for all privileged actions
- No long-term credentials exist in sandbox accounts
- All administration originates from the management account
- Trust boundaries are explicit and auditable

---

## Next Steps

With the organization and accounts established, the next document covers:

**02-mfa-admin-role-chaining.md**  
How to create the AdminAccess-mylab role, enforce MFA, configure trust policies, and enable cross-account access into sandbox accounts.