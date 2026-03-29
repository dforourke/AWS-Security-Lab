# Architecture Documentation
## AWS Security Lab

**System Name:** AWS Security Lab  
**Environment:** Amazon Web Services (AWS), us-east-1  
**IaC Platform:** Terraform  
**Last Updated:** 2026

---

## Overview

This document describes the security architecture of the AWS Security Lab environment. Every design decision is intentional and traceable to a NIST 800-53 control requirement. The architecture is provisioned entirely through Terraform, ensuring configurations are reviewable as code and reproducible across deployments.

The environment is modeled after the security requirements found in DoD/Federal cloud deployments — segmented networking, least-privilege identity, hardened compute, encrypted storage, and continuous monitoring — structured around the RMF control families.

---

## Architecture Diagram

```
                          ┌──────────────────────────────────────────────────────┐
                          │                    AWS Account                       │
                          │                                                      │
                          │   ┌──────────────────────────────────────────────┐   │
                          │   │              VPC (10.0.0.0/16)               │   │
                          │   │                                              │   │
                          │   │  ┌──────────────────┐  ┌─────────────────┐   │   │
                          │   │  │  Public Subnet   │  │  Private Subnet │   │   │
                          │   │  │  (10.0.1.0/24)   │  │  (10.0.2.0/24)  │   │   │
                          │   │  │                  │  │                 │   │   │
                          │   │  │  ┌────────────┐  │  │  ┌────────────┐ │   │   │
                          │   │  │  │  (Future)  │  │  │  │  EC2       │ │   │   │
                          │   │  │  │  ALB /     │  │  │  │  Instance  │ │   │   │
                          │   │  │  │  Bastion   │  │  │  │  IMDSv2    │ │   │   │
                          │   │  │  └────────────┘  │  │  │  EBS Enc.  │ │   │   │
                          │   │  │                  │  │  └─────┬──────┘ │   │   │
                          │   │  └────────┬─────────┘  └────────┼────────┘   │   │
                          │   │           │                     │            │   │
                          │   │    ┌──────▼──────┐              │            │   │
                          │   │    │   Internet  │        ┌─────▼──────┐     │   │
                          │   │    │   Gateway   │        │    SSM     │     │   │
                          │   │    └──────┬──────┘        │  Endpoint  │     │   │
                          │   │           │               └────────────┘     │   │
                          │   │    ┌──────▼──────┐                           │   │
                          │   │    │  Route      │                           │   │
                          │   │    │  Tables     │                           │   │
                          │   │    └─────────────┘                           │   │
                          │   │                                              │   │
                          │   │  ┌────────────────────────────────────────┐  │   │
                          │   │  │     Security & Monitoring Layer        │  │   │
                          │   │  │                                        │  │   │
                          │   │  │     CloudTrail  │  VPC Flow Logs       │  │   │
                          │   │  │     GuardDuty   │  Security Hub        │  │   │
                          │   │  │     Inspector   │  AWS Config          │  │   │
                          │   │  └────────────────────────────────────────┘  │   │
                          │   └──────────────────────────────────────────────┘   │
                          │                                                      │
                          │   ┌──────────────────────────────────────────────┐   │
                          │   │               IAM Layer                      │   │
                          │   │  EC2 Instance Profile → Least-Privilege Role │   │
                          │   │  SSM-only Access │ No SSH │ No Static Keys   │   │
                          │   └──────────────────────────────────────────────┘   │
                          └──────────────────────────────────────────────────────┘
```

---

## Component Descriptions

### VPC — Virtual Private Cloud
**NIST Controls: SC-7, AC-4**

The VPC (10.0.0.0/16) forms the outer network boundary of the lab environment. It provides logical isolation from other AWS accounts and the public internet, serving as the foundational boundary protection mechanism.

Design decisions:
- CIDR range sized to accommodate current and future subnet requirements
- DNS resolution and DNS hostnames enabled to support SSM endpoint resolution
- No default VPC resources used — all components are explicitly defined in Terraform

---

### Public Subnet (10.0.1.0/24)
**NIST Controls: SC-7, AC-4**

The public subnet is reserved for internet-facing resources such as an Application Load Balancer (ALB) in future phases. No compute instances with sensitive workloads are placed in the public subnet.

Design decisions:
- Route table includes a route to the Internet Gateway for outbound internet access
- Security Groups on any public-facing resources will restrict inbound traffic to required ports only
- EC2 instances running application workloads are not placed here

---

### Private Subnet (10.0.2.0/24)
**NIST Controls: SC-7, AC-4, SC-39**

The private subnet hosts all compute resources. Instances in this subnet have no direct internet access and are not reachable from the public internet.

Design decisions:
- No route to the Internet Gateway in the private subnet route table
- Outbound internet access will be provided through a NAT Gateway (Phase 3) for controlled egress only
- All EC2 instances are deployed here by default

---

### Internet Gateway
**NIST Controls: SC-7**

The Internet Gateway provides the boundary between the VPC and the public internet. It is attached to the VPC but only accessible to resources in the public subnet via route table configuration.

---

### Route Tables
**NIST Controls: SC-7, AC-4**

Separate route tables are associated with the public and private subnets to enforce traffic flow boundaries.

- **Public route table:** Routes 0.0.0.0/0 to the Internet Gateway
- **Private route table:** No internet route — all traffic stays within the VPC or exits through a future NAT Gateway

---

### EC2 Instance
**NIST Controls: CM-6, CM-7, SC-28, IA-5, SC-39**

EC2 instances are deployed in the private subnet with a hardened baseline configuration.

Hardening decisions:
- **IMDSv2 enforced:** Disables IMDSv1, preventing SSRF-based metadata credential theft. All metadata requests require a token-based request with `HttpTokens = required`
- **EBS encryption:** Root volume encrypted using AWS-managed KMS key (aws/ebs), satisfying SC-28
- **No public IP:** Instances have no public IP address assigned
- **No SSH key pair:** Key pairs are not generated or associated with instances; access is exclusively through SSM
- **Instance profile:** Attached IAM role provides temporary, automatically rotated credentials via IMDSv2

---

### IAM Role and Instance Profile
**NIST Controls: AC-3, AC-6, IA-2, IA-5**

An IAM role scoped to the minimum permissions required for Systems Manager Session Manager is attached to EC2 instances through an instance profile.

Design decisions:
- Policy allows only `ssm:*`, `ssmmessages:*`, and `ec2messages:*` actions required for SSM
- No `AdministratorAccess` or broad managed policies attached
- Role is defined entirely in Terraform (`iam.tf`) and is reviewable as code
- Temporary credentials are automatically rotated by the EC2 metadata service

---

### Security Groups
**NIST Controls: SC-7, AC-4, CM-7**
**Status: 🔄 In Progress (Phase 2)**

Security Groups act as stateful instance-level firewalls, enforcing a deny-by-default posture.

Design decisions:
- Default deny on all inbound traffic — only explicitly required ports are opened
- Outbound traffic restricted to required destinations only
- No inbound SSH (port 22) permitted on any Security Group
- Security Group rules are defined in Terraform for auditability and repeatability

---

### AWS Systems Manager (SSM)
**NIST Controls: AC-17, AC-3, AU-2**

SSM Session Manager replaces SSH as the remote access mechanism for all EC2 instances.

Design decisions:
- Eliminates open inbound ports for remote access
- All sessions are authenticated via IAM before establishment
- Session activity is logged and auditable
- No SSH key management overhead or attack surface

---

### CloudTrail
**NIST Controls: AU-2, AU-3, AU-9**
**Status: ⬜ Planned (Phase 4)**

CloudTrail captures all AWS API activity across the account, providing a complete audit trail of management-plane events.

Design decisions:
- Multi-region trail enabled to capture all regional API activity
- Log file integrity validation enabled to detect tampering
- Logs stored in a dedicated encrypted S3 bucket
- CloudWatch Logs integration for real-time alerting on specific events

---

### VPC Flow Logs
**NIST Controls: AU-2, SI-4**
**Status: ⬜ Planned (Phase 4)**

VPC Flow Logs capture network-level traffic metadata for all traffic within the VPC, providing visibility into accepted and rejected connections.

Design decisions:
- Enabled at the VPC level to capture all subnet traffic
- Logs stored in S3 for long-term retention and Athena querying
- Rejected traffic logs used to identify unauthorized access attempts

---

### GuardDuty
**NIST Controls: SI-3, SI-4**
**Status: ⬜ Planned (Phase 4)**

GuardDuty provides continuous threat detection by analyzing CloudTrail, VPC Flow Logs, and DNS logs for indicators of compromise.

Design decisions:
- Enabled at the account level — no agents required on instances
- Findings aggregated in Security Hub for centralized visibility
- High and Critical findings will trigger automated notifications via EventBridge

---

### Security Hub
**NIST Controls: AU-6, CA-7, RA-5**
**Status: ⬜ Planned (Phase 4)**

Security Hub aggregates findings from GuardDuty, Inspector, and Config into a centralized security dashboard.

Design decisions:
- AWS Foundational Security Best Practices standard enabled
- NIST 800-53 standard enabled for direct control compliance visibility
- Finding severity levels mapped to response priorities
- Central visibility for continuous monitoring requirements

---

### AWS Inspector
**NIST Controls: RA-5, SI-2**
**Status: ⬜ Planned (Phase 4)**

Inspector performs continuous vulnerability assessments of EC2 instances, scanning for OS-level CVEs and software package vulnerabilities.

Design decisions:
- Enabled at the account level — automatically covers all EC2 instances
- Findings aggregated in Security Hub
- Critical and High findings tracked as POA&M entries

---

### AWS Config
**NIST Controls: CM-2, CM-6, CA-7**
**Status: ⬜ Planned (Phase 4)**

AWS Config continuously monitors resource configurations and evaluates them against defined compliance rules.

Design decisions:
- Managed rules will detect unencrypted EBS volumes, open Security Groups, and disabled CloudTrail
- Configuration history provides a timeline of all resource changes
- Non-compliant resources generate findings in Security Hub

---

## Security Design Principles

The following principles guided all architecture decisions in this environment:

**1. Defense in Depth**  
Security controls are layered across network, identity, compute, and monitoring layers. No single control is relied upon exclusively.

**2. Least Privilege**  
Every IAM role, Security Group rule, and network path is scoped to the minimum required access. Permissions are granted explicitly, never implicitly.

**3. Assume Breach**  
Monitoring and detection are treated as required — not optional. GuardDuty, CloudTrail, and Security Hub are planned from the start, not added after the fact.

**4. Immutable Infrastructure**  
All resources are defined as Terraform IaC. Changes are made to code, not to live resources directly. This supports CM-2 baseline configuration and provides an auditable change history.

**5. Traceability**  
Every resource is tagged to a NIST 800-53 control family. Every design decision in this document references the control it satisfies. Architecture and compliance are not separate concerns.

---

## Terraform File Structure

```
lab/
├── main.tf          # Provider configuration, backend
├── variables.tf     # Input variable definitions
├── terraform.tfvars # Variable values
├── vpc.tf           # VPC, subnets, IGW, route tables
├── iam.tf           # IAM roles, policies, instance profiles
└── ec2.tf           # EC2 instances, security groups, EBS
```

---

*This document is updated as each lab phase is completed. Component status reflects current implementation state.*
