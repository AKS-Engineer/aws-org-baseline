# AWS Org Baseline

A minimal AWS Organizations baseline that establishes a clean identity and access foundation across a management account and sandbox accounts. This baseline includes:

- A secure MFA-backed admin workflow
- A dedicated admin role in the management account
- Cross-account access into sandbox accounts
- Deterministic CLI and Python automation for role chaining
- Clear trust boundaries and reproducible identity flows

This repository is intentionally small but architecturally correct. It mirrors the identity model used in real landing zones while remaining simple enough for learning, testing, and interviews.

---

## Identity Model

The baseline uses a three‑layer identity chain:

```
IAM User (admin-user) → MFA → AdminAccess-mylab → OrganizationAccountAccessRole (SB1/SB2)
```

This pattern ensures:

- No long-term credentials in sandbox accounts  
- All privileged actions originate from the management account  
- MFA is required for all administrative operations  
- Trust boundaries are explicit and auditable  

---

## Repository Structure

```
aws-org-baseline/
│
├── docs/
│   ├── 01-org-setup.md
│   ├── 02-mfa-admin-role-chaining.md
│   ├── 03-automation-cli-python.md
│   ├── 04-troubleshooting-console-vs-cli.md
│   └── images/
│
├── policies/
│   ├── adminaccess-mylab-trust.json
│   ├── adminaccess-mylab-inline.json
│   ├── sandbox-trust-policy.json
│   └── scp-examples.json
│
├── scripts/
│   ├── assume-admin.sh
│   ├── assume-sb1.sh
│   ├── assume-sb2.sh
│   └── iam_chain.py
│
└── README.md
```

---

## Use Cases

This baseline is useful for:

- Learning AWS Organizations and multi-account governance  
- Practicing IAM role chaining and MFA-backed admin flows  
- Testing SCPs, trust policies, and cross-account access  
- Building a minimal landing zone for personal or team projects  
- Demonstrating platform engineering fundamentals in interviews  

---

## Documentation

All documentation lives in the `docs/` directory:

- **01-org-setup.md** — Management account, OUs, sandbox accounts, SCP enablement  
- **02-mfa-admin-role-chaining.md** — Identity chain, trust policies, role switching  
- **03-automation-cli-python.md** — CLI commands and Python automation  
- **04-troubleshooting-console-vs-cli.md** — Diagnosing console session issues  

---

## Automation

The `scripts/` directory contains:

- CLI helpers for assuming roles  
- A Python script (`iam_chain.py`) that rebuilds the entire identity chain in one command  
- Environment variable export blocks for deterministic sessions  

These scripts ensure reproducibility and eliminate manual steps.

---

## Status

This is a foundational baseline and can be extended with:

- SCP guardrails  
- OIDC integration for GitHub Actions  
- IaC (Terraform/CDK)  
- Networking and VPC baselines  
- Centralized logging and security services  

The current version focuses on identity and access — the core of any landing zone.