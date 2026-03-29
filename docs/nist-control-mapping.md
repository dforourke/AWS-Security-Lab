# NIST 800-53 Control Mapping
## AWS Security Lab — Control Implementation Statements

**System Name:** AWS Security Lab  
**Environment:** Amazon Web Services (AWS), us-east-1  
**IaC Platform:** Terraform  
**Framework:** NIST SP 800-53 Rev 5  
**Prepared By:** Daniel O'Rourke  
**Last Updated:** 2026

---

## How to Read This Document

Each entry below represents a NIST 800-53 control implemented or planned within this lab environment. Implementation statements describe *how* the control is satisfied by a specific technical configuration, AWS service, or documented procedure. This format is modeled after System Security Plan (SSP) control implementation statements used in DoD RMF authorization packages.

**Status Definitions:**

| Status | Meaning |
|--------|---------|
| ✅ Implemented | Control is active and verified in the environment |
| 🔄 In Progress | Control is partially implemented; work ongoing |
| ⬜ Planned | Control is scoped and mapped; implementation pending |

---

## Access Control (AC)

---

### AC-2 — Account Management
**Status:** ⬜ Planned (Phase 3)

**Control Statement:** The organization manages information system accounts, including establishing, activating, modifying, reviewing, disabling, and removing accounts.

**Implementation:** IAM user accounts and roles are managed through Terraform-defined configurations. No standing user accounts with persistent credentials are provisioned. Access is granted through IAM roles attached to EC2 instance profiles. Role definitions are version-controlled in Terraform, enabling review and audit of all account configurations. Account review procedures will be documented in the SSP.

---

### AC-3 — Access Enforcement
**Status:** ✅ Implemented (Phase 1)

**Control Statement:** The information system enforces approved authorizations for logical access to information and system resources.

**Implementation:** Access to EC2 compute resources is enforced through IAM role-based access control. An IAM role with a least-privilege policy is attached to the EC2 instance profile, granting only the permissions required for Systems Manager Session Manager connectivity. No additional IAM permissions are granted. Resource-level policies prevent unauthorized access to S3, RDS, and other services. All access enforcement configurations are defined and version-controlled in `iam.tf`.

---

### AC-4 — Information Flow Enforcement
**Status:** ✅ Implemented (Phase 1–2)

**Control Statement:** The information system enforces approved authorizations for controlling the flow of information within the system and between interconnected systems.

**Implementation:** Information flow is enforced through VPC architecture and Security Group rules. Public and private subnets are logically separated, with route tables controlling traffic paths. Security Groups are configured as stateful deny-by-default firewalls, permitting only explicitly authorized inbound and outbound traffic. No direct internet access is permitted to resources in the private subnet. All flow enforcement configurations are defined in `vpc.tf` and `ec2.tf`.

---

### AC-6 — Least Privilege
**Status:** ✅ Implemented (Phase 1)

**Control Statement:** The organization employs the principle of least privilege, allowing only authorized accesses for users and processes which are necessary to accomplish assigned tasks.

**Implementation:** The IAM role attached to the EC2 instance profile is scoped exclusively to the permissions required for AWS Systems Manager Session Manager. No console access, SSH key pairs, or additional IAM permissions are granted. The IAM policy is defined with explicit Allow statements limited to SSM-required actions (`ssm:StartSession`, `ssmmessages:*`, `ec2messages:*`) and nothing beyond. The role and policy are defined in `iam.tf` and are reviewable as code.

---

### AC-17 — Remote Access
**Status:** ✅ Implemented (Phase 1)

**Control Statement:** The organization establishes and documents usage restrictions, configuration/connection requirements, and implementation guidance for each type of remote access allowed.

**Implementation:** Remote access to EC2 instances is provided exclusively through AWS Systems Manager Session Manager, eliminating the need for SSH key pairs, bastion hosts, or open inbound ports. No inbound SSH (port 22) is permitted in any Security Group. Session Manager provides encrypted, audited, and IAM-controlled access to instances without exposing network attack surface. Session logging is configured to capture session activity for audit purposes.

---

### AC-22 — Publicly Accessible Content
**Status:** ⬜ Planned (Phase 3)

**Control Statement:** The organization designates individuals authorized to post information onto a publicly accessible information system, trains those individuals, and reviews proposed content prior to posting.

**Implementation:** S3 bucket policies will enforce block-public-access settings by default. No S3 buckets in this environment are configured for public access. Bucket ACLs and policies will be defined in Terraform with explicit DenyPublicAccess conditions. Public access block settings will be enabled at both the bucket and account level.

---

## Audit and Accountability (AU)

---

### AU-2 — Event Logging
**Status:** ⬜ Planned (Phase 4)

**Control Statement:** The organization determines that the information system is capable of auditing defined auditable events.

**Implementation:** AWS CloudTrail is configured to capture all management events across the AWS account, including API calls, console sign-ins, and resource configuration changes. VPC Flow Logs are enabled on the lab VPC to capture accepted and rejected network traffic. Log data is stored in a dedicated S3 bucket with server-side encryption and lifecycle policies. CloudTrail log file integrity validation is enabled to detect tampering.

---

### AU-3 — Content of Audit Records
**Status:** ⬜ Planned (Phase 4)

**Control Statement:** The information system generates audit records containing information that establishes what type of event occurred, when the event occurred, where the event occurred, the source of the event, the outcome of the event, and the identity of any individuals or subjects associated with the event.

**Implementation:** CloudTrail audit records include event time, source IP, IAM identity, AWS region, resource ARN, and request/response parameters. VPC Flow Logs capture source/destination IP, port, protocol, packet count, and accept/reject status. All records are written to S3 with timestamps and are queryable through AWS Athena for investigation purposes.

---

### AU-6 — Audit Record Review, Analysis, and Reporting
**Status:** ⬜ Planned (Phase 4)

**Control Statement:** The organization reviews and analyzes information system audit records for indications of inappropriate or unusual activity and reports findings to designated organizational officials.

**Implementation:** AWS Security Hub aggregates findings from GuardDuty, Inspector, and Config into a centralized dashboard. Security Hub is configured with the AWS Foundational Security Best Practices standard and NIST 800-53 standard enabled. Finding severity levels are mapped to response priorities. Automated EventBridge rules will trigger notifications for High and Critical findings.

---

### AU-9 — Protection of Audit Information
**Status:** ⬜ Planned (Phase 4)

**Control Statement:** The information system protects audit information and audit tools from unauthorized access, modification, and deletion.

**Implementation:** CloudTrail logs are stored in a dedicated S3 bucket with server-side encryption (SSE-S3 or SSE-KMS). Bucket policies restrict write and delete access to the CloudTrail service principal only. S3 Object Lock or MFA Delete will be evaluated to prevent log tampering. Log file integrity validation is enabled in CloudTrail to detect unauthorized modification.

---

## Configuration Management (CM)

---

### CM-2 — Baseline Configuration
**Status:** ✅ Implemented (Phase 1–2)

**Control Statement:** The organization develops, documents, and maintains a current baseline configuration of the information system.

**Implementation:** The baseline configuration of all AWS resources is defined as Terraform Infrastructure as Code (IaC). The Terraform configuration files (`main.tf`, `vpc.tf`, `iam.tf`, `ec2.tf`) represent the authoritative baseline for all provisioned resources. Changes to the environment are made exclusively through Terraform, ensuring the codebase reflects the current deployed state. Version control through Git provides a full history of configuration changes.

---

### CM-6 — Configuration Settings
**Status:** 🔄 In Progress (Phase 2)

**Control Statement:** The organization establishes and documents configuration settings for information technology products employed within the information system that reflect the most restrictive mode consistent with operational requirements.

**Implementation:** EC2 instances are hardened in alignment with DISA STIG guidance for Amazon Linux 2. IMDSv2 is enforced on all instances, disabling IMDSv1 to prevent SSRF-based metadata attacks. EBS root volumes are encrypted at rest. Security Groups are configured with deny-by-default posture. Additional STIG-aligned configurations will be applied via automation and documented for assessment use.

---

### CM-7 — Least Functionality
**Status:** 🔄 In Progress (Phase 2)

**Control Statement:** The organization configures the information system to provide only essential capabilities, prohibiting or restricting the use of functions, ports, protocols, and services not required.

**Implementation:** EC2 instances are provisioned without unnecessary services, open ports, or public IP addresses. Security Groups permit only the minimum required traffic. IMDSv2 enforcement disables unauthenticated metadata access. SSH is disabled in favor of Systems Manager. Future phases will apply OS-level hardening to disable unnecessary services and daemons.

---

### CM-8 — Information System Component Inventory
**Status:** ✅ Implemented (Phase 1)

**Control Statement:** The organization develops and documents an inventory of information system components that accurately reflects the current information system.

**Implementation:** All Terraform-provisioned AWS resources are tagged with a consistent tagging schema including `Environment`, `Project`, `ManagedBy`, and `NISTControl` tags. Tags map each resource to its associated NIST 800-53 control family, enabling automated inventory queries through AWS Config and the AWS Tag Editor. The tagging schema supports CM-8 compliance by providing a continuously updated, queryable component inventory.

---

## Identification and Authentication (IA)

---

### IA-2 — Identification and Authentication (Organizational Users)
**Status:** ✅ Implemented (Phase 1)

**Control Statement:** The information system uniquely identifies and authenticates organizational users.

**Implementation:** All access to AWS resources requires authentication through IAM. EC2 instance access is provided through Systems Manager Session Manager, which requires valid AWS credentials and IAM authorization before a session can be established. No anonymous or shared credential access is permitted. IAM roles are uniquely assigned and not shared across principals.

---

### IA-5 — Authenticator Management
**Status:** ✅ Implemented (Phase 1)

**Control Statement:** The organization manages information system authenticators by verifying the identity of the individual, group, role, or device receiving the authenticator as part of the initial authenticator distribution.

**Implementation:** No SSH key pairs are generated or distributed in this environment. Authentication is managed through IAM roles and instance profiles, eliminating the need for long-term credentials on EC2 instances. The IAM role attached to the instance profile uses temporary, automatically rotated credentials provided by the EC2 metadata service (IMDSv2). No static access keys are used for instance-level operations.

---

## Risk Assessment (RA)

---

### RA-3 — Risk Assessment
**Status:** ⬜ Planned (Phase 5)

**Control Statement:** The organization conducts an assessment of risk, including the likelihood and magnitude of harm, from the unauthorized access, use, disclosure, disruption, modification, or destruction of the information system and the information it processes, stores, or transmits.

**Implementation:** A risk assessment will be conducted and documented as part of the RMF documentation phase. The assessment will identify threats and vulnerabilities specific to the AWS lab environment, assess likelihood and impact, and document residual risk. Findings from AWS Inspector, GuardDuty, and Security Hub will inform the risk assessment. Results will be documented in the SAR and referenced in the SSP.

---

### RA-5 — Vulnerability Monitoring and Scanning
**Status:** ⬜ Planned (Phase 4)

**Control Statement:** The organization scans for vulnerabilities in the information system and hosted applications periodically and when new vulnerabilities potentially affecting the system are identified.

**Implementation:** AWS Inspector is configured to perform continuous vulnerability assessments of EC2 instances, scanning for OS-level CVEs and software vulnerabilities. Inspector findings are aggregated in Security Hub and prioritized by severity. Scan results are documented and tracked as POA&M entries where remediation is required. Inspector is enabled at the account level and applies to all in-scope EC2 instances.

---

## System and Communications Protection (SC)

---

### SC-7 — Boundary Protection
**Status:** ✅ Implemented (Phase 1)

**Control Statement:** The information system monitors and controls communications at the external boundary of the system and at key internal boundaries within the system.

**Implementation:** Network boundary protection is implemented through a multi-layer VPC architecture. The Internet Gateway provides the external boundary, with route tables controlling which subnets have internet access. Public and private subnets are separated, with only the public subnet having a route to the Internet Gateway. Security Groups enforce stateful boundary protection at the instance level, permitting only explicitly authorized traffic. Private subnet resources have no direct internet exposure.

---

### SC-8 — Transmission Confidentiality and Integrity
**Status:** ⬜ Planned (Phase 3)

**Control Statement:** The information system implements cryptographic mechanisms to prevent unauthorized disclosure of information and detect changes to information during transmission.

**Implementation:** All data in transit between AWS services uses TLS encryption enforced by AWS service defaults. Application Load Balancer will be configured to enforce HTTPS with a minimum TLS 1.2 policy. HTTP-to-HTTPS redirection will be configured at the ALB listener level. VPC endpoints will be used where available to keep traffic off the public internet.

---

### SC-28 — Protection of Information at Rest
**Status:** ✅ Implemented (Phase 1)

**Control Statement:** The information system implements cryptographic protections for information at rest.

**Implementation:** EBS root volumes on all EC2 instances are encrypted using AWS-managed KMS keys (aws/ebs). Encryption is enforced at the Terraform resource level in `ec2.tf` via the `encrypted = true` parameter on the root block device. Future phases will apply encryption to RDS storage and S3 buckets. An AWS Config rule will be implemented to detect and alert on any unencrypted EBS volumes.

---

### SC-39 — Process Isolation
**Status:** ✅ Implemented (Phase 1–2)

**Control Statement:** The information system maintains a separate execution domain for each executing process.

**Implementation:** EC2 instances are provisioned in isolated private subnets with Security Groups that enforce process-level network isolation. IMDSv2 enforcement prevents processes from accessing instance metadata without explicit token-based requests, reducing the risk of SSRF attacks accessing credentials. Each instance operates within its own virtualized environment, providing OS-level process isolation by default through the AWS hypervisor.

---

## System and Information Integrity (SI)

---

### SI-2 — Flaw Remediation
**Status:** ⬜ Planned (Phase 4)

**Control Statement:** The organization identifies, reports, and corrects information system flaws; tests software and firmware updates related to flaw remediation for effectiveness and potential side effects before installation; and installs security-relevant software updates within organizationally defined time periods.

**Implementation:** AWS Inspector continuously scans EC2 instances for known CVEs and software vulnerabilities. Findings are aggregated in Security Hub and assigned severity levels. Critical and High findings will be tracked as POA&M entries with remediation timelines. SSM Patch Manager will be configured to automate OS patching on a defined schedule, with patch compliance reporting available through the SSM console.

---

### SI-3 — Malicious Code Protection
**Status:** ⬜ Planned (Phase 4)

**Control Statement:** The organization implements malicious code protection mechanisms at information system entry and exit points.

**Implementation:** AWS GuardDuty provides continuous threat detection by analyzing CloudTrail logs, VPC Flow Logs, and DNS logs for indicators of malicious activity including malware, command-and-control traffic, and compromised credentials. GuardDuty findings are aggregated in Security Hub. High-severity findings will trigger automated EventBridge notifications. GuardDuty is enabled at the account level and requires no agents on EC2 instances.

---

### SI-4 — System Monitoring
**Status:** ⬜ Planned (Phase 4)

**Control Statement:** The organization monitors the information system to detect attacks and indicators of potential attacks and unauthorized local, network, and remote connections.

**Implementation:** Continuous system monitoring is implemented through a layered approach: CloudTrail captures all API activity, VPC Flow Logs capture network-level traffic, GuardDuty performs behavioral threat detection, and Security Hub aggregates findings from all sources. AWS Config monitors resource configurations for drift from defined baselines and generates findings for non-compliant resources. Monitoring coverage spans identity, network, and host layers.

---

## Planning (PL)

---

### PL-2 — System Security Plan
**Status:** ⬜ Planned (Phase 5)

**Control Statement:** The organization develops a security plan for the information system that is consistent with the organization's enterprise architecture.

**Implementation:** A System Security Plan (SSP) will be developed during Phase 5 of the lab, documenting the system boundary, environment of operation, security control implementations, and authorization decisions. The SSP will reference this control mapping document and the architecture documentation as supporting artifacts. SSP format will follow NIST SP 800-18 guidance and align with DoD RMF package requirements.

---

### PL-8 — Security and Privacy Architectures
**Status:** ✅ Implemented (Phase 1)

**Control Statement:** The organization develops a security architecture for the information system that is consistent with and supports the enterprise security architecture.

**Implementation:** The lab architecture is designed from the ground up with security as a foundational requirement. The VPC design enforces network segmentation, the IAM architecture enforces least privilege, and all compute resources are hardened before deployment. Security architecture decisions are documented in `docs/architecture.md` and traced to specific NIST 800-53 control families. Infrastructure as Code enables the architecture to be reviewed, validated, and reproduced consistently.

---

## Contingency Planning (CP)

---

### CP-9 — System Backup
**Status:** ⬜ Planned (Phase 3)

**Control Statement:** The organization conducts backups of user-level information, system-level information, and information system documentation including security-related documentation.

**Implementation:** AWS Backup will be configured to perform automated EBS snapshots and RDS backups on a defined schedule. Backup retention policies will be defined in Terraform. Backup vaults will be encrypted and access-controlled via IAM. Recovery procedures will be documented and tested as part of the contingency planning documentation in Phase 5.

---

*This document will be updated as each lab phase is completed. Control status will be updated to reflect verified implementation.*
