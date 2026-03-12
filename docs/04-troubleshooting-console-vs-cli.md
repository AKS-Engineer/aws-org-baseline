# Troubleshooting AWS Console vs CLI Authentication Behavior

This document explains why the AWS Console may appear to switch roles successfully while backend STS calls fail, and why the AWS CLI and Python automation always work reliably. The goal is to provide a clear diagnostic framework for understanding authentication failures, session corruption, and browser‑level issues that do not originate from IAM or SCPs.

---

## Overview

During setup and testing, the CLI successfully executed the full identity chain:

```
admin-user (MFA)
    ↓
AdminAccess-mylab
    ↓
OrganizationAccountAccessRole (sandbox1 or sandbox2)
```

However, the AWS Console UI appeared to switch roles but failed to authenticate backend STS calls, resulting in:

```
MissingAuthenticationToken
```

This discrepancy highlights a critical distinction:

- **CLI failures** indicate IAM or SCP issues  
- **Console failures** often indicate browser session or cookie issues  

Understanding this difference is essential for diagnosing multi‑account identity problems.

---

## 1. What the CLI Proved

Successful CLI role chaining confirmed that:

- IAM user configuration was correct  
- MFA enforcement was correct  
- AdminAccess‑mylab trust policy was correct  
- Sandbox trust policies were correct  
- Cross‑account access was correctly configured  
- SCPs were not blocking any required actions  

If any IAM or SCP issue existed, the CLI would have returned:

```
AccessDenied
```

Instead, all CLI calls succeeded, proving the identity model was correct.

---

## 2. Why the Console Failed

The AWS Console relies on browser cookies to authenticate calls to:

- `signin.aws.amazon.com`
- `console.aws.amazon.com`
- `sts.amazonaws.com`

Modern browsers increasingly isolate or block cross‑site cookies. When this happens:

- The UI may appear logged in  
- The role switch UI may appear successful  
- But backend STS calls receive **no authentication cookies**  
- Resulting in `MissingAuthenticationToken`  

This creates a “zombie session”:

- The console *looks* authenticated  
- But the backend *is not* authenticated  

This is a browser‑level failure, not an IAM failure.

---

## 3. Symptoms of Console Session Corruption

Common indicators include:

- Role switch UI works visually but identity does not change  
- STS URLs return `MissingAuthenticationToken`  
- Console pages partially load or show inconsistent identity  
- Switching roles repeatedly does nothing  
- Logging out/in does not fix the issue  
- CLI works perfectly while console fails  

These symptoms point to cookie isolation, not IAM misconfiguration.

---

## 4. Root Causes in Modern Browsers

Modern browsers (Chrome, Edge, Firefox) increasingly enforce:

- Third‑party cookie blocking  
- Partitioned cookies  
- Strict tracking protection  
- Cross‑domain cookie isolation  
- Profile‑level cookie segregation  

AWS Console depends on cross‑domain cookies to maintain session continuity.  
If these cookies are blocked or partitioned, the console cannot authenticate STS calls.

---

## 5. Why SCPs Cannot Cause This Issue

SCPs operate at the **authorization** layer, not the authentication layer.

SCP failures always produce:

```
AccessDenied
```

SCPs cannot:

- Remove or invalidate console cookies  
- Prevent cookies from being attached to STS calls  
- Cause `MissingAuthenticationToken`  
- Cause the console to appear logged in while backend calls fail  

Therefore, SCPs were not involved in the console failure.

---

## 6. Why CLI and Python Always Work

The CLI and Python SDK use:

- Direct credential injection  
- Explicit AccessKey/SecretKey/SessionToken  
- No cookies  
- No browser session state  
- No cross‑domain authentication  

This bypasses the entire browser layer.

As long as IAM and trust policies are correct, CLI and Python will always work.

---

## 7. How to Fix Console Authentication Issues

### Option 1 — Use a different browser
Switching to a fresh browser profile often resolves cookie isolation issues.

### Option 2 — Disable strict tracking protection
Relaxing tracking/cookie settings for AWS domains restores cross‑site cookie flow.

### Option 3 — Use dedicated browser profiles per account
This prevents session contamination and aligns with enterprise best practices.

### Option 4 — Clear cookies for AWS domains
This resets the authentication state and forces a clean login.

---

## 8. Recommended Operational Practice

For all privileged operations:

- Use **CLI or Python automation**  
- Treat the console as a convenience layer  
- Never rely on the console for identity verification  
- Always verify identity with:

```
aws sts get-caller-identity
```

This ensures deterministic, auditable identity transitions.

---

## Summary

- The CLI proved the IAM model was correct.  
- The console failed due to browser cookie isolation.  
- SCPs were not involved.  
- CLI/Python automation is the reliable, enterprise‑grade path.  
- Browser profile isolation and cookie settings prevent future issues.

This troubleshooting model is now part of the baseline and should be referenced whenever console behavior diverges from CLI results.