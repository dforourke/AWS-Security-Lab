# AWS Security Lab

A hands-on cloud security lab built to demonstrate real-world implementation of security controls aligned to **NIST 800-53** and **DoD RMF** standards. Infrastructure is provisioned with **Terraform** and mapped directly to control families throughout.

This lab is designed to reflect the security engineering work required in DoD/Federal cloud environments — from foundational architecture through continuous monitoring and RMF documentation.

---

## Architecture Overview

> Diagram coming soon — see `/diagrams/` folder.

**Environment:** AWS (us-east-1)  
**IaC:** Terraform  
**OS:** Amazon Linux 2 (EC2)

Key components:
- Segmented VPC with public and private subnets
- EC2 instances in private subnet with IMDSv2 enforced and encrypted EBS
- IAM roles with least-privilege, SSM-only access (no SSH)
- Security Groups enforcing deny-by-default posture
- Centralized logging and monitoring (CloudTrail, VPC Flow Logs, GuardDuty, Security Hub, Inspector)

---

## Lab Phases

| Phase | Focus | Status |
|-------|-------|--------|
| 1 | VPC & IAM Foundation | ✅ Complete |
| 2 | Compute Hardening (EC2, Security Groups) | 🔄 In Progress |
| 3 | Data & Network Security | ⬜ Planned |
| 4 | Logging & Continuous Monitoring | ⬜ Planned |
| 5 | RMF Documentation (SSP, POA&M, SAR) | ⬜ Planned |
| 6 | IaC Repeatability & Validation | ⬜ Planned |

---

## NIST 800-53 Control Mapping

| Control | Family | Implementation |
|---------|--------|----------------|
| AC-3 | Access Enforcement | IAM least-privilege role; EC2 instance profile |
| AC-6 | Least Privilege | SSM-only access; no standing SSH; scoped IAM policy |
| AC-17 | Remote Access | Systems Manager Session Manager replaces SSH |
| SC-7 | Boundary Protection | VPC segmentation; public/private subnet separation |
| SC-28 | Protection at Rest | EBS root volume encryption enabled |
| CM-8 | Information System Component Inventory | All resources tagged by control family and environment |
| AU-2 | Audit Events | CloudTrail enabled; VPC Flow Logs active |
| SI-3 | Malicious Code Protection | GuardDuty threat detection |
| RA-5 | Vulnerability Scanning | AWS Inspector for EC2 vulnerability assessment |
| CA-7 | Continuous Monitoring | Security Hub aggregating findings across services |

---

## Tools & Services

**AWS:** IAM, VPC, EC2, SSM, GuardDuty, Security Hub, CloudTrail, VPC Flow Logs, Config, Inspector  
**IaC:** Terraform  
**Frameworks:** NIST 800-53, DoD RMF, DISA STIG  

---

## Documentation

RMF-style documentation is located in `/docs/`:

- `architecture.md` — environment design and security rationale
- `nist-control-mapping.md` — detailed control implementation statements
- `ssp-sample.md` — System Security Plan excerpts aligned to lab controls

---

## About

Built to demonstrate practical cloud security engineering skills for DoD and federal cloud environments. All security decisions are intentional, documented, and traceable to NIST 800-53 control requirements.
