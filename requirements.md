# Requirements Document

## Introduction

This document specifies the functional, security, network, and compliance requirements for an AWS Cyber Recovery solution centred on an Isolated Recovery Environment (IRE). The solution protects production workloads using Rubrik backup, restores them into a strictly isolated AWS account (IRE) for security scanning and validation, and promotes clean workloads to a pre-provisioned Disaster Recovery (DR) environment. Foundational services such as Active Directory, DNS, monitoring, and certificate services are provided by a dedicated ISO (Isolated Services/Foundational Services) environment consumed securely by IRE workloads. All data is encrypted at rest and in transit using AWS KMS.

---

## Glossary

- **IRE**: Isolated Recovery Environment — the dedicated AWS account into which Rubrik restores workloads for scanning and validation before promotion to DR.
- **DR Environment**: Disaster Recovery Environment — the pre-provisioned AWS account per application that receives clean, validated workloads promoted from the IRE.
- **ISO Environment**: Isolated Services/Foundational Services Environment — the environment hosting trusted foundational services (Active Directory, DNS, monitoring, certificate services) consumed by IRE workloads.
- **Rubrik**: The backup and recovery platform used to protect production workloads and restore them into the IRE.
- **Tanium**: Endpoint security platform used to scan and cleanse restored workloads inside the IRE.
- **CrowdStrike**: Endpoint Detection and Response (EDR) platform used to scan restored workloads inside the IRE.
- **Qualys**: Vulnerability scanning platform used to assess restored workloads inside the IRE.
- **Palo Alto Firewall**: The next-generation firewall enforcing traffic inspection and routing restrictions on IRE-bound and IRE-originating traffic traversing the Transit Gateway.
- **Transit Gateway**: The AWS Transit Gateway shared between Production, DR, and IRE environments, with routing restricted via the Palo Alto Firewall for IRE traffic.
- **Break-Glass Account**: A tightly controlled, non-SSO AWS IAM account used exclusively for emergency access to the IRE, bypassing corporate identity federation.
- **KMS**: AWS Key Management Service — used to manage encryption keys for data at rest and in transit across all environments.
- **EC2/EBS**: Amazon Elastic Compute Cloud instances and Elastic Block Store volumes.
- **RDS**: Amazon Relational Database Service managed database instances.
- **EKS**: Amazon Elastic Kubernetes Service managed Kubernetes clusters.
- **Lambda**: AWS Lambda serverless compute functions.
- **Promotion**: The controlled process of moving a scanned, validated workload from IRE to the DR Environment.
- **OU**: AWS Organizations Organizational Unit.
- **SCPs**: AWS Organizations Service Control Policies applied at the OU level.

---

## Requirements

### Requirement 1: Rubrik Backup of Production Workloads

**User Story:** As a cloud operations engineer, I want production workloads to be continuously protected by Rubrik backups, so that clean, restorable copies are available for recovery into the IRE.

#### Acceptance Criteria

1. THE Rubrik_Backup_Service SHALL protect all in-scope production workloads including EC2/EBS volumes, RDS instances, EKS clusters, and Lambda functions, where "all in-scope" means every workload registered in the backup inventory at the time of job execution.
2. THE Rubrik_Backup_Service SHALL encrypt all backup data at rest using AWS KMS customer-managed keys (CMKs) assigned to the source environment, such that no backup object is written to storage without a CMK association verifiable via the backup completion record.
3. WHEN a scheduled backup job completes, THE Rubrik_Backup_Service SHALL record a backup completion event including workload identifier, timestamp accurate to the second, backup size in bytes, and KMS key ARN used.
4. IF a scheduled backup job fails, THEN THE Rubrik_Backup_Service SHALL generate an alert within 15 minutes of the failure event timestamp, including the workload identifier and failure reason, without altering or deleting the most recent successful backup for that workload.
5. THE Rubrik_Backup_Service SHALL retain each backup copy for a minimum of 30 days and a maximum of 365 days from the backup completion timestamp, in accordance with the defined retention policy.
6. WHEN backup data is transferred between environments, THE Rubrik_Backup_Service SHALL encrypt data in transit using TLS 1.2 or higher.
7. WHEN a scheduled backup job completes, THE Rubrik_Backup_Service SHALL verify that all in-scope workloads present in the backup inventory were included in the job, and IF any registered workload was not backed up, THEN THE Rubrik_Backup_Service SHALL generate an alert within 15 minutes identifying each omitted workload by its workload identifier.

---

### Requirement 2: IRE Account Isolation and Placement

**User Story:** As a security architect, I want the IRE to be a dedicated AWS account within the same AWS Organization and OU as Production and DR, so that organisational policies are consistently enforced while maintaining strict isolation.

#### Acceptance Criteria

1. THE IRE_Account SHALL reside within the same AWS Organisation and the same Organisational Unit (OU) as the Production and DR environments.
2. THE IRE_Account SHALL have AWS Organizations Service Control Policies (SCPs) applied at the OU level that deny, at minimum, the following action categories: direct internet egress from IRE subnets, cross-account VPC peering or resource sharing with the Production account outside of the Transit Gateway attachment, and IAM privilege escalation beyond roles explicitly designated for recovery operations.
3. THE IRE_Account SHALL share the Transit Gateway used by the Production environment, with all IRE-bound and IRE-originating traffic routed exclusively through the Palo Alto Firewall before reaching any destination.
4. THE Palo Alto_Firewall SHALL enforce allowlist-based rules such that traffic originating from any source other than Rubrik, Tanium, CrowdStrike, Qualys, and the ISO Environment is denied and dropped before reaching the IRE.
5. THE IRE_Account SHALL block all inbound and outbound internet traffic at the network layer using AWS security groups and network ACLs, permitting no direct internet paths in either direction.
6. WHILE the IRE is in steady-state (no active recovery operation is authorised), THE IRE_Account SHALL maintain no active route table entries targeting Production subnets and no established network sessions between the IRE and the Production environment.

---

### Requirement 3: Break-Glass Access Control for IRE

**User Story:** As a security administrator, I want all human access to the IRE to use break-glass accounts rather than corporate SSO, so that compromised corporate identity credentials cannot be used to access the recovery environment.

#### Acceptance Criteria

1. THE IRE_Account SHALL enforce access exclusively through designated Break-Glass Accounts, with corporate SSO federation disabled at the account level.
2. THE Break-Glass_Account_Manager SHALL store break-glass credentials in AWS Secrets Manager within the IRE account, encrypted with a dedicated KMS CMK.
3. WHEN a Break-Glass Account is used to authenticate or when break-glass credentials are rotated, THE IRE_Account SHALL generate a CloudTrail audit event capturing the IAM principal, source IP address, action performed, and timestamp.
4. WHEN a login is attempted using a Break-Glass Account, THE Break-Glass_Account_Manager SHALL require multi-factor authentication (MFA) to be provided before access is granted.
5. IF a Break-Glass Account login is attempted from an IP address outside the approved range, THEN THE IRE_Account SHALL deny the authentication attempt and generate a security alert.
6. IF a Break-Glass Account MFA challenge fails, THEN THE Break-Glass_Account_Manager SHALL deny access, increment the failed attempt counter for that account, and generate a CloudTrail audit event recording the IAM principal, source IP, and failure timestamp.
7. THE Break-Glass_Account_Manager SHALL rotate break-glass credentials on a schedule not exceeding 90 days from the last rotation date.
8. WHEN a Break-Glass Account is used to access the IRE, THE Break-Glass_Account_Manager SHALL initiate credential rotation for that account within 1 hour of the session ending.

---

### Requirement 4: Restore Workloads into the IRE

**User Story:** As a recovery engineer, I want Rubrik to restore workloads into the IRE on demand, so that the recovery process can begin from a known-good backup point.

#### Acceptance Criteria

1. WHEN a recovery event is declared, THE Rubrik_Restore_Service SHALL initiate the restore of the designated workload backup into the IRE Account within 5 minutes of the event declaration, using the restore mechanism native to the target workload type.
2. THE Rubrik_Restore_Service SHALL decrypt backup data using the source environment's KMS CMK and re-encrypt restored data using the IRE Account's KMS CMK during the restore operation.
3. WHEN a restore operation completes successfully, THE Rubrik_Restore_Service SHALL record a restore completion event including workload identifier, backup timestamp, restore duration, and target IRE resource identifiers.
4. IF a restore operation fails after exhausting 3 retry attempts, THEN THE Rubrik_Restore_Service SHALL halt the restoration, log a failure record containing the workload identifier, failure timestamp, and failure description, and dispatch a notification containing the workload identifier and failure summary to the on-call recovery team within 15 minutes of the final failure.
5. THE Rubrik_Restore_Service SHALL restore workloads into designated IRE subnets with no public IP addresses assigned.
6. WHILE a restore operation is in progress, THE IRE_Account SHALL enforce network-level blocking of all outbound connections from the partially restored workload until the Rubrik_Restore_Service registers a scan job for that workload.
7. IF a restore attempt fails and the retry count has not reached 3, THEN THE Rubrik_Restore_Service SHALL automatically retry the restore operation, waiting 2 minutes between each attempt.

---

### Requirement 5: EC2/EBS Workload Recovery

**User Story:** As a recovery engineer, I want EC2 instances and EBS volumes to be restored into the IRE using snapshot-based recovery, so that server workloads can be recovered with their full data state.

#### Acceptance Criteria

1. WHEN an EC2 workload recovery is initiated, THE Rubrik_Restore_Service SHALL restore the workload by creating EBS volume snapshots in the IRE Account from the encrypted Rubrik backup and attaching them to a pre-registered EC2 launch template defined in the IRE Account for that workload type.
2. THE IRE_Account SHALL store all restored EBS volumes encrypted using the IRE's KMS CMK.
3. WHEN an EC2 instance is restored into the IRE, THE IRE_Account SHALL assign the instance to an isolated security group that denies all inbound traffic except from Tanium, CrowdStrike, and Qualys scanner endpoints.
4. WHEN an EC2 instance is restored into the IRE, THE Rubrik_Restore_Service SHALL set the restored instance's tags, IAM instance profile reference, and volume attachment configuration to values identical to those recorded in the source workload's backup metadata.
5. IF the EC2 workload restore fails for any reason, THEN THE Rubrik_Restore_Service SHALL log the incompatibility or failure details and raise an alert to the recovery team within 5 minutes.
6. WHEN an EC2 restore operation completes successfully, THE Rubrik_Restore_Service SHALL record the restored instance identifier, restore timestamp, and source backup identifier in the audit log.

---

### Requirement 6: RDS Workload Recovery

**User Story:** As a recovery engineer, I want RDS database instances to be restored into the IRE using snapshot-based recovery, so that database workloads can be recovered with full data integrity.

#### Acceptance Criteria

1. WHEN a recovery job targets an RDS workload, THE Rubrik_Restore_Service SHALL restore the workload by invoking the native RDS snapshot restore API to create a new RDS instance from the designated encrypted snapshot in the IRE Account.
2. WHEN an RDS instance is restored into the IRE Account, THE IRE_Account SHALL apply the IRE's KMS CMK as the encryption key for the restored instance at the time of restore, before the instance enters an available state.
3. WHEN an RDS instance is restored into the IRE, THE IRE_Account SHALL place the instance in a private subnet and attach a security group that permits inbound traffic on the engine's default database port exclusively from the CIDR ranges listed in the IRE's approved scanning tools configuration.
4. WHEN an RDS instance is restored into the IRE, THE IRE_Account SHALL set the instance's public accessibility attribute to disabled before the instance transitions to an available state.
5. IF the RDS snapshot restore fails due to an engine version incompatibility, THEN THE Rubrik_Restore_Service SHALL emit an alert to the recovery team within 5 minutes of failure detection and record a log entry containing the instance identifier, source snapshot identifier, detected engine version, and the reason for incompatibility.
6. WHEN an RDS instance is restored into the IRE, THE Rubrik_Restore_Service SHALL write the original parameter group name and option group name to the recovery job's restore metadata record, such that both values are retrievable by post-validation tooling via the recovery job identifier.

---

### Requirement 7: EKS Workload Recovery

**User Story:** As a recovery engineer, I want EKS cluster workloads to be restored into the IRE, so that containerised applications can be recovered and validated before promotion.

#### Acceptance Criteria

1. WHEN an EKS workload recovery is initiated, THE Rubrik_Restore_Service SHALL restore persistent volume snapshots and redeploy Kubernetes manifests into a pre-provisioned EKS cluster within the IRE Account within 60 minutes of initiation.
2. WHEN an EKS workload is restored into the IRE Account, THE IRE_Account SHALL encrypt all persistent volumes attached to the restored EKS pods using the IRE's KMS CMK before the pods reach a running state.
3. WHEN an EKS workload is restored into the IRE, THE IRE_Account SHALL configure Kubernetes network policies to permit only inbound traffic from scanning tool endpoints and deny all other inbound traffic.
4. WHEN an EKS workload recovery is initiated, THE Rubrik_Restore_Service SHALL restore the original namespace structure, resource quotas, and service account configurations from the backup metadata.
5. IF a Kubernetes manifest references an image stored in a Production container registry that does not return a successful response within 30 seconds of connection attempt from the IRE network, THEN THE IRE_Account SHALL substitute the image pull source with the IRE-approved internal registry and log the substitution including the original registry reference and the substitute registry reference.
6. WHEN an EKS workload restore is complete, THE IRE_Account SHALL disable all external load balancer services on the restored EKS workload until the workload has been promoted.
7. IF the Rubrik_Restore_Service fails to restore one or more persistent volume snapshots or fails to redeploy one or more Kubernetes manifests, THEN THE Rubrik_Restore_Service SHALL halt the restore operation, remove any partially restored resources from the EKS cluster, and report the failure with an error message indicating which component failed.

---

### Requirement 8: Lambda Workload Recovery

**User Story:** As a recovery engineer, I want Lambda functions to be restored into the IRE, so that serverless workloads can be validated before promotion to DR.

#### Acceptance Criteria

1. THE Rubrik_Restore_Service SHALL restore Lambda functions from backup into the IRE Account using the AWS Lambda API, deploying the function code package along with its runtime, handler, memory allocation, timeout, layers, and environment variables.
2. THE IRE_Account SHALL encrypt Lambda environment variables at rest using the IRE's KMS CMK.
3. WHEN a Lambda function is restored into the IRE, THE IRE_Account SHALL deploy the function inside a VPC with a security group restricting outbound traffic to the IRE-defined scanning tool endpoints only, blocking all other outbound destinations.
4. THE Rubrik_Restore_Service SHALL record the original Lambda execution role ARN in the restored function's metadata without applying it as the active execution role.
5. WHEN a Lambda function is restored into the IRE, THE IRE_Account SHALL assign an IRE-scoped execution role derived from the original execution role's policy, scoped to IRE Account resources only, as the function's active execution role.
6. IF a restored Lambda function contains an environment variable whose value matches a Production account ARN pattern — defined as any ARN containing the Production AWS account ID registered in the IRE configuration — THEN THE IRE_Account SHALL set the function state to disabled and record each matching variable name as requiring manual review, until all flagged variables have been updated to non-Production ARN values or explicitly acknowledged by a recovery engineer and the function is manually re-enabled.
7. IF the Rubrik_Restore_Service fails to restore a Lambda function, THEN THE Rubrik_Restore_Service SHALL leave any partially deployed function resources in a disabled state, record the function name and the reason for failure, and surface an error indication to the recovery engineer.

---

### Requirement 9: Security Scanning of Restored Workloads

**User Story:** As a security operations engineer, I want all restored workloads to be scanned by Tanium, CrowdStrike, and Qualys before promotion, so that only clean, vulnerability-free workloads reach the DR environment.

#### Acceptance Criteria

1. WHEN a restore operation completes, THE IRE_Scanning_Orchestrator SHALL automatically trigger security scans against the restored workload in the following sequence: Tanium first, CrowdStrike second, and Qualys third.
2. WHEN the IRE_Scanning_Orchestrator triggers a Tanium scan, THE Tanium_Scanner SHALL perform endpoint threat detection on restored EC2 instances and EKS nodes and SHALL produce a scan result report within 2 hours of scan initiation, where a clean status means zero threats detected across all scanned endpoints.
3. WHEN the IRE_Scanning_Orchestrator triggers a CrowdStrike scan, THE CrowdStrike_Scanner SHALL perform endpoint detection and response (EDR) analysis on restored EC2 instances and EKS nodes and SHALL produce a scan result report within 2 hours of scan initiation, where a clean status means zero detections of any severity across all scanned endpoints.
4. WHEN the IRE_Scanning_Orchestrator triggers a Qualys scan, THE Qualys_Scanner SHALL perform a vulnerability assessment on all restored EC2 instances and EKS nodes and SHALL produce a vulnerability report within 4 hours of scan initiation, where a clean status means zero findings rated CVSS v3 score 7.0 or above across all scanned components.
5. IF any scanner returns a finding with a CVSS v3 score of 7.0 or above, or a vendor-native severity of Critical or High, THEN THE IRE_Scanning_Orchestrator SHALL isolate the affected workload by removing it from all network segments except the IRE quarantine VLAN, halt the promotion pipeline for that workload, and notify the security operations team within 15 minutes of detection.
6. THE IRE_Scanning_Orchestrator SHALL not permit promotion of any workload to the DR Environment until all three scanners have returned a clean status, as defined in criteria 2, 3, and 4, for that workload.
7. IF any scanner fails to produce a result report within its defined time limit, THEN THE IRE_Scanning_Orchestrator SHALL treat that workload as failed scanning, halt the promotion pipeline for that workload, and notify the security operations team within 15 minutes of the timeout.
8. THE IRE_Account SHALL pre-approve network access for Tanium, CrowdStrike, and Qualys endpoints through the Palo Alto Firewall so that scanning tools can reach restored workloads without additional firewall rule changes at recovery time.
9. THE IRE_Scanning_Orchestrator SHALL record the scan initiation time, scanner identity, scan completion time, and scan outcome for each workload scan in an immutable audit log retained for a minimum of 365 days.

---

### Requirement 10: Workload Cleansing

**User Story:** As a security operations engineer, I want infected or compromised workload components identified during scanning to be cleansed before promotion, so that malware or threat artefacts are not carried into the DR environment.

#### Acceptance Criteria

1. WHEN Tanium identifies a threat on a restored workload, THE Tanium_Scanner SHALL execute automated remediation to remove the identified threat artefact and record the remediation action in the audit log, including the tool name, threat identifier, and remediation status (success or failure).
2. WHEN CrowdStrike identifies a threat on a restored workload, THE CrowdStrike_Scanner SHALL isolate the affected process or file, attempt automated remediation, and record the outcome in the audit log, including the tool name, threat identifier, and remediation status (success or failure).
3. IF automated remediation by Tanium or CrowdStrike reports a failure status or does not complete within 30 minutes of initiation, THEN THE IRE_Scanning_Orchestrator SHALL place the workload in a quarantine state that blocks promotion and escalate the workload to manual security review.
4. IF automated remediation completes without a failure status, THEN THE IRE_Scanning_Orchestrator SHALL re-run a full scan cycle across all three scanning tools and SHALL allow promotion only if all three tools report zero detected threats.
5. WHEN a workload is placed in quarantine, THE IRE_Scanning_Orchestrator SHALL record the quarantine event in the audit log, including the workload identifier, the triggering tool name, and the timestamp.

---

### Requirement 11: Promotion of Validated Workloads to DR Environment

**User Story:** As a recovery engineer, I want validated workloads to be promoted from the IRE to the DR environment in a controlled manner, so that clean workloads are available for service restoration.

#### Acceptance Criteria

1. WHEN all scanning and validation checks pass for a workload, THE Promotion_Service SHALL initiate the promotion workflow to the designated DR Environment for that application.
2. WHEN approval is granted by the designated recovery authority, THE Promotion_Service SHALL transfer workload artefacts (snapshots, container images, function packages) from the IRE to the DR Environment using encrypted channels (TLS 1.2 or higher) and SHALL re-encrypt data at rest in the DR Environment using the DR Environment's KMS CMK.
3. THE Promotion_Service SHALL require approval from a designated recovery authority (human approval gate) before executing the final promotion step, and IF no approval decision is received within 4 hours of the approval request being raised, THEN THE Promotion_Service SHALL expire the pending promotion request, notify the recovery team of the expiry, and require the promotion workflow to be re-initiated.
4. WHEN a workload is promoted to the DR Environment, THE Promotion_Service SHALL record the promotion event including workload identifier, IRE scan results summary, approving authority, and promotion timestamp in the audit log.
5. IF a promotion operation fails, THEN THE Promotion_Service SHALL roll back any partially transferred artefacts, log the failure reason, and notify the recovery team within 15 minutes.
6. IF the target DR infrastructure for a workload is not pre-provisioned or does not match the workload's designated type (EC2 instance for VM workloads, RDS instance for database workloads, EKS node group for container workloads, or Lambda function configuration for serverless workloads), THEN THE Promotion_Service SHALL abort the promotion for that workload, log the infrastructure mismatch reason, and notify the recovery team.
7. WHEN each workload artefact transfer completes, THE Promotion_Service SHALL validate that the checksum of the transferred artefact matches the checksum recorded at the end of the IRE scanning phase, and IF any checksum does not match, THEN THE Promotion_Service SHALL abort the promotion, roll back all transferred artefacts for that workload, and notify the recovery team of the integrity failure before confirming promotion success.

---

### Requirement 12: DR Environment Workload Activation

**User Story:** As a recovery engineer, I want promoted workloads in the DR environment to be activatable on demand, so that services can be restored after a cyber event.

#### Acceptance Criteria

1. THE DR_Environment SHALL maintain pre-provisioned infrastructure (VPC, subnets, security groups, IAM roles, KMS keys) for each in-scope application, ready to receive promoted workloads.
2. WHEN a workload is successfully promoted, THE DR_Environment SHALL validate that the promoted artefact is importable into the target infrastructure before marking the workload as activation-ready.
3. THE DR_Environment SHALL encrypt all workload data at rest using the DR Environment's KMS CMK.
4. WHEN a workload is activated in the DR Environment, THE DR_Environment SHALL generate an activation event logged in CloudTrail including the workload identifier, activation timestamp, and the IAM principal that triggered activation.

---

### Requirement 13: ISO Environment Foundational Services Consumption

**User Story:** As a platform engineer, I want IRE workloads to be able to securely consume approved foundational services from the ISO Environment, so that recovery operations are not blocked by the absence of critical infrastructure services.

#### Acceptance Criteria

1. THE IRE_Account SHALL permit outbound connectivity to the ISO Environment for the following approved services only: Active Directory authentication, DNS resolution, monitoring and logging ingestion, certificate services, and security tooling.
2. THE Palo Alto_Firewall SHALL enforce allowlist rules restricting IRE-to-ISO traffic to the specific ports and protocols required by each approved foundational service, and SHALL deny all other IRE-to-ISO traffic.
3. WHEN an IRE workload submits a service request to the ISO_Environment, THE ISO_Environment SHALL authenticate that request using service accounts or certificates provisioned exclusively in a dedicated IRE service account scope that does not include any Production service accounts.
4. WHEN an IRE workload sends a DNS query, THE ISO_DNS_Service SHALL resolve only DNS zones listed in the approved IRE DNS zone allowlist, and IF the queried zone is not on that allowlist, THEN THE ISO_DNS_Service SHALL return a REFUSED or NXDOMAIN response and SHALL NOT silently drop or forward the query.
5. THE ISO_Environment SHALL provide monitoring and logging ingestion endpoints that accept log writes from IRE workloads with write-only access, such that IRE workloads are not granted read or list access to any Production log data.
6. IF the ISO_Environment is unavailable, THEN THE IRE_Account SHALL fall back to locally cached DNS entries with a maximum cache age of 24 hours and locally pre-staged certificates, SHALL alert the operations team within 5 minutes of outage detection, and SHALL resume consuming live ISO services automatically once connectivity is restored.

---

### Requirement 14: KMS Encryption at Rest

**User Story:** As a security architect, I want all data stored in IRE and DR environments to be encrypted using AWS KMS customer-managed keys, so that encryption keys are fully auditable and independently rotatable per environment.

#### Acceptance Criteria

1. THE IRE_Account SHALL maintain a dedicated AWS KMS customer-managed key (CMK) for each workload type (EC2/EBS, RDS, EKS persistent volumes, Lambda environment variables).
2. THE DR_Environment SHALL maintain a dedicated AWS KMS CMK for each workload type (EC2/EBS, RDS, EKS persistent volumes, Lambda environment variables), with distinct key ARNs from the IRE CMKs.
3. THE KMS_Key_Manager SHALL enforce automatic annual key rotation for all CMKs used in the IRE and DR environments.
4. THE IRE_Account SHALL restrict KMS CMK usage to explicitly authorised IAM roles and services via KMS key policies, with an explicit deny statement covering all principals not in the authorised set.
5. WHEN any of the following KMS API actions are invoked — Encrypt, Decrypt, GenerateDataKey, Sign, or Verify — THE KMS_Key_Manager SHALL record the event in AWS CloudTrail including the key ARN, calling principal, and action performed, where CloudTrail KMS data event logging is enabled for all CMKs in the IRE and DR environments.
6. IF a KMS CMK is scheduled for deletion, THEN THE KMS_Key_Manager SHALL deliver an alert to the security team within 24 hours via the configured notification channel and enforce a 30-day waiting period before deletion proceeds.

---

### Requirement 15: Network Isolation and Traffic Control

**User Story:** As a network security engineer, I want all traffic into and out of the IRE to be strictly controlled through the Palo Alto Firewall, so that the IRE cannot be used as a pivot point to reach Production or other environments.

#### Acceptance Criteria

1. THE Palo Alto_Firewall SHALL inspect all traffic entering and leaving the IRE via the Transit Gateway and SHALL deny any traffic that does not match an explicitly defined allow rule in the IRE zone security policy.
2. THE IRE_Account SHALL have no direct VPC peering connections to Production VPCs; all inter-environment traffic SHALL traverse the Transit Gateway and Palo Alto Firewall.
3. WHILE the IRE_Recovery_State is active, THE Palo Alto_Firewall SHALL block all traffic originating from any IRE subnet destined for any Production environment IP range, where "active" means the period from when an IRE recovery operation is initiated until it is formally closed.
4. THE Palo Alto_Firewall SHALL log all allowed and denied traffic flows traversing the IRE boundary, including source IP, destination IP, port, protocol, and action taken, and SHALL retain those logs for a minimum of 90 days.
5. THE IRE_Account SHALL use private, non-routable IP address ranges with no Elastic IP addresses or public-facing endpoints assigned to restored workloads.
6. WHEN a new workload is restored into the IRE, THE IRE_Account SHALL automatically apply the designated IRE security group and network ACL rules to the workload within 60 seconds of the workload reaching a running state, without requiring manual intervention.
7. IF the automatic application of IRE security group or network ACL rules to a restored workload fails, THEN THE IRE_Account SHALL halt the workload, raise an alert to the operations team, and retain the workload in a stopped state until rules are successfully applied.
8. THE IRE_Account SHALL enforce VPC endpoint policies for all AWS service access (S3, KMS, SSM, ECR) such that any request from an IRE workload targeting a service endpoint outside the IRE Account boundary is denied and logged.

---

### Requirement 16: Compliance, Auditability, and Logging

**User Story:** As a compliance officer, I want all recovery actions, access events, and security findings to be captured in immutable audit logs, so that the organisation can demonstrate compliance and conduct forensic investigations.

#### Acceptance Criteria

1. THE IRE_Account SHALL enable AWS CloudTrail with multi-region logging for all management and data events, with logs delivered to a centralised S3 bucket in a dedicated logging account.
2. THE IRE_Account SHALL enable AWS Config to record all resource configuration changes within the IRE Account and store configuration history in the centralised logging account.
3. WHEN a recovery event is completed, THE IRE_Scanning_Orchestrator SHALL produce an end-to-end recovery audit report within 30 minutes of pipeline completion, stored in the centralised logging account, including: backup source, restore timestamp, scanning results per tool, cleansing actions taken, promotion approval record, and DR activation status.
4. WHEN any resource in the IRE is modified outside of an active recovery workflow, THE IRE_Account SHALL generate a high-severity alert to the security operations team within 5 minutes of the detected modification, where an active recovery workflow is defined as a pipeline execution initiated by an authorised IRE orchestration role and recorded in the IRE workflow registry.
5. THE Centralised_Logging_Account SHALL retain IRE CloudTrail logs, AWS Config snapshots, and GuardDuty findings for a minimum of 365 days.
6. THE IRE_Account SHALL enable Amazon GuardDuty to detect anomalous behaviour within the IRE, with findings forwarded to the security operations team's SIEM within 5 minutes of detection.
7. THE IRE_Account SHALL enforce S3 bucket policies on all log storage buckets to prevent deletion or modification of log objects, using S3 Object Lock in compliance mode with a minimum retention of 365 days.

---

### Requirement 17: Recovery Time and Recovery Point Objectives

**User Story:** As a business continuity manager, I want the cyber recovery process to meet defined RTO and RPO targets, so that service restoration occurs within acceptable business impact thresholds.

#### Acceptance Criteria

1. THE Rubrik_Backup_Service SHALL achieve a Recovery Point Objective (RPO) of no greater than 24 hours for all workloads registered in the IRE workload inventory, measured as the time between the last successful backup and the declared recovery event.
2. THE IRE_Scanning_Orchestrator SHALL complete the full restore, scan, and validation pipeline for a single workload within a Recovery Time Objective (RTO) of 8 hours from the time the restore operation is initiated, regardless of whether intermediate scan attempts fail and are retried within that window.
3. WHEN the elapsed time for a workload's recovery pipeline reaches 6 hours without reaching the validation stage, THE IRE_Scanning_Orchestrator SHALL notify the recovery team and incident manager with an updated estimated completion time.
4. WHEN two or more workload recovery pipelines are executing concurrently, THE IRE_Scanning_Orchestrator SHALL apply the 8-hour RTO independently to each workload, measured from the initiation time of each individual restore operation.

---

### Requirement 18: Identity and Access Management within IRE

**User Story:** As a security architect, I want all IAM roles and policies within the IRE to follow least-privilege principles, so that the blast radius of any compromised credential within the IRE is minimised.

#### Acceptance Criteria

1. THE IRE_Account SHALL enforce IAM permission boundaries on all IAM roles created within the IRE Account, where the permission boundary policy explicitly denies all IAM, S3 log bucket, and CloudTrail management actions unless the role is a designated IRE orchestration role listed in the IRE role registry.
2. THE IRE_Account SHALL deny the use of IAM wildcard actions (`*`) in any inline or managed IAM policy attached to any role within the IRE Account, including workload execution roles, operator roles, and cross-account roles.
3. THE IRE_Account SHALL require all AWS API calls to workload resources to originate from within the IRE VPC, enforced via IAM condition keys (`aws:SourceVpc`).
4. WHEN a new IAM role is created in the IRE Account, THE IRE_Account SHALL record the creation event in CloudTrail and notify the security operations team within 15 minutes of the creation event.
5. THE IRE_Account SHALL disable all IAM access keys for the root account and enforce MFA on the root account.
