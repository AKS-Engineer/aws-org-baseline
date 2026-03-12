# Automation for MFA‑Backed Role Chaining (CLI and Python)

This document provides deterministic, reproducible automation for the identity chain defined in the baseline. It includes AWS CLI commands for each hop and a Python script that rebuilds the entire chain in one command. The goal is to eliminate manual steps, reduce human error, and ensure consistent identity transitions across the management and sandbox accounts.

---

## Overview

The identity chain is:

```
admin-user (MFA)
    ↓
AssumeRole: AdminAccess-mylab
    ↓
AssumeRole: OrganizationAccountAccessRole (sandbox1 or sandbox2)
```

Automation ensures:

- Correct sequence of STS calls  
- MFA enforcement  
- Clean separation between hops  
- No credential contamination  
- Predictable, auditable identity transitions  

This is essential for IAM testing, SCP evaluation, and multi-account workflows.

---

## 1. AWS CLI Automation

### Prerequisites

- AWS CLI v2 installed  
- A configured profile named `admin-user`  
- MFA device ARN for `admin-user`  
- Role ARNs for:
  - `AdminAccess-mylab`
  - `OrganizationAccountAccessRole` in each sandbox account  

---

## 2. Step 1 — Get MFA Session for admin-user

```
aws sts get-session-token \
  --serial-number arn:aws:iam::602131314874:mfa/admin-user \
  --token-code <MFA_CODE> \
  --profile admin-user
```

This returns temporary credentials:

- `AccessKeyId`
- `SecretAccessKey`
- `SessionToken`

Export them:

```
set AWS_ACCESS_KEY_ID=<value>
set AWS_SECRET_ACCESS_KEY=<value>
set AWS_SESSION_TOKEN=<value>
```

Verify identity:

```
aws sts get-caller-identity
```

---

## 3. Step 2 — Assume AdminAccess‑mylab (Management Account)

```
aws sts assume-role \
  --role-arn arn:aws:iam::602131314874:role/AdminAccess-mylab \
  --role-session-name admin-hop
```

Export the returned credentials:

```
set AWS_ACCESS_KEY_ID=<value>
set AWS_SECRET_ACCESS_KEY=<value>
set AWS_SESSION_TOKEN=<value>
```

Verify identity:

```
aws sts get-caller-identity
```

---

## 4. Step 3 — Assume Sandbox Role (SB1 or SB2)

### Sandbox 1

```
aws sts assume-role \
  --role-arn arn:aws:iam::608051150029:role/OrganizationAccountAccessRole \
  --role-session-name chain
```

### Sandbox 2

```
aws sts assume-role \
  --role-arn arn:aws:iam::255737342348:role/OrganizationAccountAccessRole \
  --role-session-name chain
```

Export credentials and verify identity.

---

## 5. Why CLI Automation Matters

CLI automation guarantees:

- Correct hop order  
- No stale credentials  
- No browser session issues  
- Full visibility into identity transitions  
- Deterministic, repeatable behavior  

This is the foundation for the Python automation layer.

---

## 6. Python Automation Script (Full Chain)

The following script performs the entire identity chain in one command:

- MFA session  
- AdminAccess‑mylab  
- Sandbox role  
- Identity verification at each hop  
- Final export block  

Save as: `scripts/iam_chain.py`

```python
import boto3
import json
import sys

MFA_SERIAL = "arn:aws:iam::602131314874:mfa/admin-user"
ADMIN_ROLE = "arn:aws:iam::602131314874:role/AdminAccess-mylab"
SB_ROLES = {
    "sb1": "arn:aws:iam::608051150029:role/OrganizationAccountAccessRole",
    "sb2": "arn:aws:iam::255737342348:role/OrganizationAccountAccessRole",
}

SESSION_DURATION = 3600


def get_mfa_session(mfa_code):
    client = boto3.client("sts", profile_name="admin-user")
    resp = client.get_session_token(
        SerialNumber=MFA_SERIAL,
        TokenCode=mfa_code,
        DurationSeconds=SESSION_DURATION,
    )
    return resp["Credentials"]


def assume(creds, role_arn, session_name):
    client = boto3.client(
        "sts",
        aws_access_key_id=creds["AccessKeyId"],
        aws_secret_access_key=creds["SecretAccessKey"],
        aws_session_token=creds["SessionToken"],
    )
    resp = client.assume_role(
        RoleArn=role_arn,
        RoleSessionName=session_name,
        DurationSeconds=SESSION_DURATION,
    )
    return resp["Credentials"]


def identity(creds):
    client = boto3.client(
        "sts",
        aws_access_key_id=creds["AccessKeyId"],
        aws_secret_access_key=creds["SecretAccessKey"],
        aws_session_token=creds["SessionToken"],
    )
    return client.get_caller_identity()


def main():
    if len(sys.argv) != 3:
        print("Usage: python iam_chain.py <mfa_code> <sb1|sb2>")
        sys.exit(1)

    mfa_code = sys.argv[1]
    target = sys.argv[2].lower()

    if target not in SB_ROLES:
        print("Target must be sb1 or sb2")
        sys.exit(1)

    mfa_creds = get_mfa_session(mfa_code)
    print("\n[MFA Session]")
    print(json.dumps(identity(mfa_creds), indent=4))

    admin_creds = assume(mfa_creds, ADMIN_ROLE, "admin-hop")
    print("\n[AdminAccess-mylab]")
    print(json.dumps(identity(admin_creds), indent=4))

    sb_creds = assume(admin_creds, SB_ROLES[target], "chain")
    print(f"\n[{target.upper()}]")
    print(json.dumps(identity(sb_creds), indent=4))

    print("\n[EXPORT]")
    print(f"set AWS_ACCESS_KEY_ID={sb_creds['AccessKeyId']}")
    print(f"set AWS_SECRET_ACCESS_KEY={sb_creds['SecretAccessKey']}")
    print(f"set AWS_SESSION_TOKEN={sb_creds['SessionToken']}")


if __name__ == "__main__":
    main()
```

---

## 7. Running the Script

### Sandbox 1

```
python iam_chain.py 123456 sb1
```

### Sandbox 2

```
python iam_chain.py 123456 sb2
```

The script prints:

- Identity at each hop  
- Final environment variables to export  
- A deterministic, auditable chain  

---

## 8. Why Python Automation Matters

Python automation provides:

- A single command to rebuild the entire chain  
- Zero manual credential handling  
- Guaranteed hop order  
- Clean identity transitions  
- A foundation for future IAM harness automation  

This is the same pattern used in enterprise platform engineering teams.

---

## Next Steps

The next document covers:

**04-troubleshooting-console-vs-cli.md**  
A detailed explanation of why the AWS Console can fail to pass authentication cookies, how to diagnose session corruption, and why CLI/Python automation always works reliably.