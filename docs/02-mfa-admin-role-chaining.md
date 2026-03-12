# MFA‑Backed Admin Role Chaining (Management → Sandbox Accounts)

This document defines the identity and role‑chaining model used across the AWS Organization. It establishes a secure, MFA‑enforced workflow for administering sandbox accounts from the management account using AWS IAM roles and STS. The design mirrors enterprise landing‑zone patterns while remaining intentionally minimal.

---

## Identity Architecture

The baseline identity chain is:

```
admin-user (MFA)
    ↓
AssumeRole: AdminAccess-mylab
    ↓
AssumeRole: OrganizationAccountAccessRole (sandbox1 or sandbox2)
```

This pattern enforces:

- MFA for all privileged operations  
- No long‑term credentials in sandbox accounts  
- Clear separation of authentication (user) and authorization (role)  
- A single administrative choke point in the management account  
- Explicit, auditable trust boundaries  

---

## 1. Base Identity: admin-user

### Purpose
`admin-user` is the only human login identity in the management account. It has:

- No direct permissions  
- MFA required  
- A single responsibility: authenticate → assume the admin role  

This prevents accidental privilege escalation and aligns with enterprise SSO → AdminRole patterns.

---

## 2. Create the AdminAccess‑mylab Role (Management Account)

### Steps

1. IAM → Roles → **Create role**
2. Trusted entity type: **AWS account**
3. Use case: **This account**
4. Do **not** enable external ID or MFA here  
5. Permissions: attach **AdministratorAccess**
6. Name the role: **AdminAccess-mylab**
7. Create the role

### Replace the trust policy with:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::602131314874:user/admin-user"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

### Result
- Only `admin-user` can assume this role  
- MFA is required  
- This role holds full administrative privileges in the management account  

---

## 3. Allow admin-user to Assume AdminAccess‑mylab

IAM → Users → **admin-user** → Permissions → Add inline policy

**Policy Name:** `AssumeRole-AdminAccess-mylab`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::602131314874:role/AdminAccess-mylab"
    }
  ]
}
```

### Result
`admin-user` can now assume the admin role after MFA authentication.

---

## 4. Grant Cross‑Account Access to Sandbox Accounts

Each sandbox account contains the AWS‑managed role:

```
OrganizationAccountAccessRole
```

This role is created automatically when the account is created through AWS Organizations.

### Goal
Allow the management account’s `AdminAccess-mylab` role to assume this role in each sandbox account.

### Steps (repeat for sandbox1 and sandbox2)

1. Log in to the sandbox account as **root**
2. IAM → Roles → open **OrganizationAccountAccessRole**
3. Trust relationships → **Edit trust policy**
4. Replace with:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::602131314874:role/AdminAccess-mylab"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Result
The management account can now administer each sandbox account through a secure, MFA‑backed chain.

---

## 5. Identity Chain Verification

After configuration, the following STS calls should succeed:

### MFA session (admin-user)
```
aws sts get-session-token --serial-number <MFA_ARN> --token-code <code>
```

### First hop (AdminAccess‑mylab)
```
aws sts assume-role \
  --role-arn arn:aws:iam::602131314874:role/AdminAccess-mylab \
  --role-session-name admin-hop
```

### Second hop (sandbox account)
```
aws sts assume-role \
  --role-arn arn:aws:iam::<sandbox-account-id>:role/OrganizationAccountAccessRole \
  --role-session-name chain
```

Each hop should produce valid credentials and a correct identity when calling:

```
aws sts get-caller-identity
```

---

## 6. Console Role Switching (Optional)

Direct console switch-role URLs:

### Management Account Admin Role
```
https://signin.aws.amazon.com/switchrole?account=602131314874&roleName=AdminAccess-mylab&displayName=Admin-mylab
```

### Sandbox 1
```
https://signin.aws.amazon.com/switchrole?account=608051150029&roleName=OrganizationAccountAccessRole&displayName=sandbox1
```

### Sandbox 2
```
https://signin.aws.amazon.com/switchrole?account=255737342348&roleName=OrganizationAccountAccessRole&displayName=sandbox2
```

---

## 7. Why This Pattern Works

This identity chain provides:

- MFA‑backed privileged access  
- No long‑term credentials in sandbox accounts  
- A single administrative choke point  
- Predictable, auditable trust relationships  
- A clean foundation for automation and SCP governance  
- Alignment with enterprise landing‑zone identity models  

---

## Next Steps

The next document covers automation:

**03-automation-cli-python.md**  
How to reproduce the entire identity chain using AWS CLI and Python, including deterministic session creation and identity verification.