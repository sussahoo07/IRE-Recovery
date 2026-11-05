# Implementation Plan: AWS Cyber Recovery — Isolated Recovery Environment (IRE)

## Overview

This plan converts the IRE design into discrete, ordered coding tasks. Tasks are grouped into phases that mirror the layered architecture: foundational AWS infrastructure, core orchestration engine, workload-specific recovery pipelines, security scanning integration, ISO connectivity, promotion pipeline, and compliance/audit. Each phase builds on the previous. Property-based tests (Hypothesis) are placed close to the implementation they validate to catch correctness regressions early.

Implementation language: **Python 3.12**
Property-based testing library: **Hypothesis**
Infrastructure-as-Code: **AWS CDK (Python)**

---

## Tasks

- [ ] 1. Repository and project structure
  - Create Python package layout: `ire_orchestrator/`, `ire_infra/` (CDK), `tests/unit/`, `tests/property/`, `tests/integration/`, `tests/smoke/`
  - Add `pyproject.toml` with runtime dependencies: `boto3`, `aws-cdk-lib`, `constructs`, `hypothesis`, `pytest`, `moto`
  - Add `pytest.ini` with markers: `unit`, `property`, `integration`, `smoke`
  - Configure `mypy`, `ruff`, and `pre-commit` hooks
  - _Requirements: all_

- [ ] 2. Multi-account AWS organisation and OU infrastructure
  - [ ] 2.1 Define CDK stacks for AWS Organization OU placement and SCP attachments
    - Create `ire_infra/stacks/org_policy_stack.py` defining the shared OU and SCP documents
    - Implement SCP: deny `ec2:CreateInternetGateway`, `ec2:AttachInternetGateway` in IRE account
    - Implement SCP: deny `ec2:CreateVpcPeeringConnection` across all OU accounts
    - Implement SCP: deny `iam:CreatePolicyVersion`, `iam:SetDefaultPolicyVersion`, `iam:AttachRolePolicy` except for designated orchestration roles
    - Implement SCP: deny any policy containing `"Action": "*"`
    - _Requirements: 2.1, 2.2_

  - [ ]* 2.2 Write smoke tests for SCP enforcement
    - Verify that `ec2:CreateInternetGateway` is denied at the OU level for IRE account
    - Verify that `ec2:CreateVpcPeeringConnection` is denied
    - Verify that IAM wildcard action policies are blocked
    - _Requirements: 2.2_

- [ ] 3. Transit Gateway and Palo Alto Firewall routing infrastructure
  - [ ] 3.1 Implement CDK construct for Transit Gateway with per-account route tables
    - Create `ire_infra/constructs/transit_gateway.py`
    - Define TGW route tables: RT-Production, RT-IRE, RT-DR, RT-ISO, RT-Firewall
    - Set default route on RT-IRE → Firewall TGW attachment
    - Set default route on RT-Production, RT-DR, RT-ISO → Firewall TGW attachment
    - _Requirements: 2.3_

  - [ ] 3.2 Implement CDK construct for Palo Alto Firewall VPC and TGW attachment
    - Create `ire_infra/constructs/palo_alto_firewall.py`
    - Define Firewall VPC with trust interface and TGW attachment
    - Provision placeholder zone security policy documents (allowlist-based, deny-default)
    - Enforce no default-permit rules in policy template
    - _Requirements: 2.3, 2.4, 15.1_

  - [ ]* 3.3 Write smoke tests for Transit Gateway routing and firewall attachment
    - Verify RT-IRE has default route pointing to Firewall attachment
    - Verify no direct route exists between IRE and Production route tables
    - _Requirements: 2.3, 2.6_

- [ ] 4. IRE VPC and network isolation infrastructure
  - [ ] 4.1 Implement CDK stack for IRE VPC with subnets and route tables
    - Create `ire_infra/stacks/ire_vpc_stack.py`
    - Define subnets: Restore-A (10.100.1.0/24), Restore-B (10.100.2.0/24), Quarantine (10.100.10.0/24), Management (10.100.20.0/24), Endpoint-A (10.100.30.0/24), Endpoint-B (10.100.31.0/24)
    - Restore RT: default route → TGW attachment
    - Quarantine RT: no default route (complete isolation)
    - No Internet Gateway, no NAT Gateway, no Elastic IPs
    - _Requirements: 2.5, 4.5, 15.5_

  - [ ] 4.2 Implement CDK constructs for IRE Security Groups
    - Create `ire_infra/constructs/security_groups.py`
    - Implement `IRE-Restore-SG`: inbound Tanium (TCP 17472, 17486), CrowdStrike (TCP 443), Qualys (TCP 443, ICMP); outbound ISO-AD (TCP/UDP 389, 636), ISO-DNS (TCP/UDP 53), ISO-Mon (TCP 443); deny all else
    - Implement `IRE-Quarantine-SG`: deny all inbound and outbound
    - Implement `IRE-RDS-SG`: inbound on engine default port from scanner CIDRs only; outbound to ISO services only
    - _Requirements: 5.3, 6.3, 9.5, 9.8_

  - [ ] 4.3 Implement CDK constructs for Network ACLs
    - Create `ire_infra/constructs/network_acls.py`
    - Restore subnets NACL: inbound from scanner CIDRs; outbound to ISO service CIDRs
    - Quarantine subnet NACL: deny all inbound and outbound
    - Management subnet NACL: VPC-internal inbound only; TGW outbound only
    - _Requirements: 2.5, 9.5_

  - [ ] 4.4 Implement CDK constructs for VPC Interface Endpoints with endpoint policies
    - Create `ire_infra/constructs/vpc_endpoints.py`
    - Define endpoints for: S3, KMS, SSM, ECR
    - Attach endpoint policies denying requests targeting resources outside the IRE account boundary
    - _Requirements: 15.8_

  - [ ]* 4.5 Write smoke tests for IRE network isolation
    - Verify no Internet Gateway or NAT Gateway attached to IRE VPC
    - Verify Quarantine subnet route table has no default route
    - Verify VPC endpoint policies deny cross-account requests
    - _Requirements: 2.5, 15.5, 15.8_

- [ ] 5. KMS CMK hierarchy
  - [ ] 5.1 Implement CDK stack for IRE KMS CMKs
    - Create `ire_infra/stacks/ire_kms_stack.py`
    - Define CMKs: `ire/ebs`, `ire/rds`, `ire/eks-pv`, `ire/lambda-env`, `ire/breakglass`, `ire/logs`
    - Enable automatic annual key rotation on all CMKs
    - Apply key policy: allow only authorised role ARNs; explicit deny for all other principals
    - _Requirements: 14.1, 14.3, 14.4_

  - [ ] 5.2 Implement CDK stack for DR Environment KMS CMKs (per application)
    - Create `ire_infra/stacks/dr_kms_stack.py`
    - Define CMKs: `dr/ebs`, `dr/rds`, `dr/eks-pv`, `dr/lambda-env`
    - Distinct key ARNs from IRE CMKs; annual rotation enabled
    - _Requirements: 14.2, 14.3_

  - [ ] 5.3 Implement KMS key deletion alert via EventBridge
    - Create `ire_infra/constructs/kms_deletion_alert.py`
    - EventBridge rule on `ScheduleKeyDeletion` → SNS → security team
    - Enforce 30-day deletion waiting period in key policy
    - _Requirements: 14.6_

  - [ ] 5.4 Implement CloudTrail KMS data event logging
    - Enable KMS data events (Encrypt, Decrypt, GenerateDataKey, Sign, Verify) in CloudTrail trail configuration
    - Verify key ARN and calling principal are captured per event
    - _Requirements: 14.5_

  - [ ]* 5.5 Write smoke tests for KMS configuration
    - Verify automatic rotation enabled on all IRE and DR CMKs
    - Verify key policies contain explicit deny for unauthorised principals
    - Verify deletion waiting period is 30 days
    - _Requirements: 14.3, 14.4, 14.6_

- [ ] 6. Centralised Logging Account infrastructure
  - [ ] 6.1 Implement CDK stack for Centralised Logging Account S3 buckets
    - Create `ire_infra/stacks/logging_stack.py`
    - Create S3 buckets with Object Lock in **compliance mode**, minimum 365-day retention
    - Attach bucket policies denying `s3:DeleteObject`, `s3:PutObject` (modification), `s3:DeleteBucket` for all principals
    - Configure cross-account delivery policy allowing CloudTrail and Config from IRE account
    - _Requirements: 16.5, 16.7_

  - [ ] 6.2 Implement CDK constructs for CloudTrail multi-region trail in IRE account
    - Create `ire_infra/constructs/cloudtrail.py`
    - Enable multi-region trail with management events + data events (S3, KMS, Lambda)
    - Enable log file validation (SHA-256 digest files)
    - Deliver to Centralised Logging Account S3 bucket
    - _Requirements: 16.1_

  - [ ] 6.3 Implement CDK constructs for AWS Config recorder in IRE account
    - Create `ire_infra/constructs/config_recorder.py`
    - Enable Config recorder for all resource types
    - Deliver configuration history and snapshots to Centralised Logging Account S3
    - _Requirements: 16.2_

  - [ ] 6.4 Implement CDK constructs for Amazon GuardDuty with SIEM forwarding
    - Create `ire_infra/constructs/guardduty.py`
    - Enable GuardDuty in IRE account
    - EventBridge rule → Kinesis Data Firehose → SIEM endpoint
    - Target delivery within 5 minutes of finding
    - _Requirements: 16.6_

  - [ ]* 6.5 Write smoke tests for logging infrastructure
    - Verify S3 Object Lock is in compliance mode with 365-day retention
    - Verify CloudTrail multi-region trail is enabled with log file validation
    - Verify Config recorder is enabled for all resource types
    - _Requirements: 16.1, 16.5, 16.7_

- [ ] 7. Break-glass access control
  - [ ] 7.1 Implement CDK stack for break-glass IAM users and group
    - Create `ire_infra/stacks/breakglass_iam_stack.py`
    - Define IAM users: `bg-recovery-engineer-01`, `bg-recovery-engineer-02`, `bg-security-admin-01`
    - Create group `bg-recovery-operators`
    - Attach MFA enforcement policy: deny all actions when `aws:MultiFactorAuthPresent == false`
    - Attach IP restriction policy: deny `sts:AssumeRole` from IPs not in approved CIDR list
    - Disable SSO federation at account level
    - Disable root account access keys; enforce root MFA
    - _Requirements: 3.1, 3.4, 3.5, 18.5_

  - [ ] 7.2 Implement CDK constructs for break-glass Secrets Manager integration
    - Create `ire_infra/constructs/breakglass_secrets.py`
    - Store credentials in Secrets Manager encrypted with `ire/breakglass` CMK
    - Resource policy restricts access to rotation Lambda and IRE orchestration roles only
    - _Requirements: 3.2_

  - [ ] 7.3 Implement break-glass credential rotation Lambda
    - Create `ire_orchestrator/lambdas/breakglass_rotation.py`
    - Implement scheduled rotation: trigger every 90 days via EventBridge schedule
    - Implement post-use rotation: CloudTrail `ConsoleLogin` event → EventBridge rule → trigger rotation Lambda within 1 hour of session end
    - _Requirements: 3.7, 3.8_

  - [ ]* 7.4 Write unit tests for break-glass rotation scheduling logic
    - Test that scheduled rotation fires at ≤90-day intervals
    - Test that post-use rotation is triggered within 1 hour of session-end CloudTrail event
    - Test that rotation Lambda correctly calls Secrets Manager rotate API
    - _Requirements: 3.7, 3.8_

  - [ ]* 7.5 Write smoke tests for break-glass access control
    - Verify SSO federation is disabled in IRE account
    - Verify MFA is enforced on break-glass accounts
    - Verify root access keys are disabled and root MFA is active
    - _Requirements: 3.1, 3.4, 18.5_

- [ ] 8. IAM permission boundaries and least-privilege roles
  - [ ] 8.1 Implement CDK constructs for IAM permission boundary policy
    - Create `ire_infra/constructs/iam_boundaries.py`
    - Permission boundary: deny `iam:*`, `s3:*` on log buckets, `cloudtrail:*` for all roles except those in IRE role registry
    - Deny `"Action": "*"` wildcard policies (complement to SCP)
    - Enforce `aws:SourceVpc` condition on all workload API calls
    - _Requirements: 18.1, 18.2, 18.3_

  - [ ] 8.2 Implement CDK constructs for designated IRE orchestration roles
    - Create `ire_infra/constructs/orchestration_roles.py`
    - Define roles: `ire-scanning-orchestrator-role`, `ire-promotion-service-role`, `ire-breakglass-rotation-role`, `ire-rubrik-restore-role`, `ire-config-recorder-role`
    - Apply least-privilege inline policies for each role
    - Register roles in IRE role registry (SSM Parameter Store list)
    - _Requirements: 18.1_

  - [ ] 8.3 Implement new IAM role creation alert via EventBridge
    - Create `ire_infra/constructs/iam_creation_alert.py`
    - EventBridge rule on CloudTrail `CreateRole` event → SNS → security team within 15 minutes
    - _Requirements: 18.4_

  - [ ]* 8.4 Write smoke tests for IAM permission boundaries
    - Verify all IRE roles have permission boundary attached
    - Verify no role in IRE account has a wildcard action policy
    - Verify `aws:SourceVpc` condition is present on workload execution role policies
    - _Requirements: 18.1, 18.2, 18.3_

- [ ] 9. Checkpoint — infrastructure baseline
  - Ensure all CDK stacks synthesise without errors (`cdk synth`)
  - Ensure all smoke tests pass against synthesised CloudFormation templates
  - Ask the user if questions arise before proceeding to orchestration code.

- [ ] 10. IRE Workflow Registry (DynamoDB)
  - [ ] 10.1 Implement CDK construct for IRE Workflow Registry DynamoDB table
    - Create `ire_infra/constructs/workflow_registry.py`
    - Define table with PK `workflowId` (String), SK `workloadId` (String)
    - Enable point-in-time recovery and encryption with `ire/logs` CMK
    - Define GSI on `state` for query-by-state operations
    - _Requirements: 16.4_

  - [ ] 10.2 Implement Python data model and registry client for workflow state
    - Create `ire_orchestrator/registry.py`
    - Implement `WorkflowRegistry` class: `create_workflow()`, `update_state()`, `get_workflow()`, `list_active_workflows()`
    - Define `WorkflowState` enum matching all states in design (DECLARED through FAILED)
    - Implement `is_authorised_activity(iam_principal, resource_id)` — queries registry to detect out-of-band modifications
    - _Requirements: 16.4_

  - [ ]* 10.3 Write unit tests for WorkflowRegistry client
    - Test state transitions using moto-mocked DynamoDB
    - Test `is_authorised_activity` returns True for registered orchestration role + active workflow, False otherwise
    - Test `create_workflow()` generates UUID and sets `state = DECLARED`
    - _Requirements: 16.4_

- [ ] 11. Rubrik integration client
  - [ ] 11.1 Implement Rubrik REST API client
    - Create `ire_orchestrator/rubrik_client.py`
    - Implement `RubrikClient`: `initiate_restore(workload_id, backup_id, target_account)`, `get_restore_status(job_id)`, `list_backups(workload_id)`
    - All HTTP calls use `requests.Session` with `ssl.PROTOCOL_TLS`, minimum TLS 1.2 enforced via `SSLContext`
    - Implement retry logic: max 3 attempts, 2-minute backoff between attempts
    - _Requirements: 1.6, 4.1, 4.7_

  - [ ] 11.2 Implement backup completion record writer
    - Create `ire_orchestrator/backup_records.py`
    - Implement `BackupCompletionRecord` dataclass with fields: `workload_id`, `timestamp` (ISO 8601, second precision), `backup_size_bytes`, `kms_key_arn`, `backup_job_id`, `status`
    - Implement `write_backup_record(record)` → serialise to JSON, write to Centralised Logging S3 bucket via VPC endpoint
    - Implement `BackupJobValidator.check_coverage(job_id, inventory)` → returns set of omitted workload IDs
    - _Requirements: 1.3, 1.7_

  - [ ] 11.3 Implement restore completion record writer
    - Create `ire_orchestrator/restore_records.py`
    - Implement `RestoreCompletionRecord` dataclass: `workload_id`, `backup_timestamp`, `restore_duration_seconds`, `target_resource_ids`, `kms_key_arn`, `workflow_id`
    - Implement `write_restore_record(record)` → write to Centralised Logging S3
    - _Requirements: 4.3_

  - [ ]* 11.4 Write property test for backup completeness (Property 1)
    - **Property 1: Backup completeness**
    - **Validates: Requirements 1.7**
    - For any backup job execution and workload inventory, every workload in the inventory at job execution time either appears in the completed backup job or triggers an alert identifying it by `workload_id`
    - Tag: `Feature: aws-cyber-recovery-ire, Property 1: Backup completeness`

  - [ ]* 11.5 Write unit tests for Rubrik client and backup/restore records
    - Test `RubrikClient` retry: assert exactly one retry after 2-minute wait on first failure; assert pipeline halts and alert fires after third failure
    - Test `BackupCompletionRecord` serialisation includes all required fields
    - Test `RestoreCompletionRecord` includes `kms_key_arn` matching IRE CMK pattern
    - _Requirements: 1.3, 4.3, 4.4, 4.7_

- [ ] 12. Re-encryption logic
  - [ ] 12.1 Implement cross-account EBS snapshot re-encryption helper
    - Create `ire_orchestrator/encryption/ebs_reencrypt.py`
    - Implement `reencrypt_ebs_snapshot(source_snapshot_id, source_cmk_arn, target_cmk_arn, target_account_id)` using EC2 `copy_snapshot` with `KmsKeyId` override
    - Verify resulting snapshot's `KmsKeyId` matches IRE EBS CMK ARN before returning
    - _Requirements: 4.2, 5.2_

  - [ ] 12.2 Implement cross-account RDS snapshot re-encryption helper
    - Create `ire_orchestrator/encryption/rds_reencrypt.py`
    - Implement `reencrypt_rds_snapshot(source_snapshot_id, target_cmk_arn)` using RDS `restore_db_instance_from_db_snapshot` with `KmsKeyId`
    - Verify `StorageEncrypted=True` and `KmsKeyId` matches IRE RDS CMK before instance is `available`
    - _Requirements: 4.2, 6.2_

  - [ ]* 12.3 Write property test for re-encryption on restore (Property 3)
    - **Property 3: Re-encryption on restore**
    - **Validates: Requirements 4.2**
    - For any restored workload, every resource's encryption key ARN must match the IRE CMK ARN for that workload type and must not match any Production account CMK ARN
    - Tag: `Feature: aws-cyber-recovery-ire, Property 3: Re-encryption on restore`

  - [ ]* 12.4 Write property test for retry before halt (Property 4)
    - **Property 4: Retry before halt**
    - **Validates: Requirements 4.7**
    - For any restore failure where retry count < 3, the system schedules exactly one retry after a 2-minute wait without halting the pipeline or sending a failure notification
    - Tag: `Feature: aws-cyber-recovery-ire, Property 4: Retry before halt`

- [ ] 13. Workload-specific recovery: EC2/EBS
  - [ ] 13.1 Implement EC2 restore handler
    - Create `ire_orchestrator/workloads/ec2_restore.py`
    - Implement `EC2RestoreHandler.restore(workload_id, backup_id, workflow_id)`
    - Select pre-registered IRE launch template by `workload_type` tag from backup metadata
    - Create EBS snapshot in IRE (calling `ebs_reencrypt` helper)
    - Launch EC2 instance: no public IP, assign `IRE-Restore-SG`, apply backup metadata tags, IAM profile ref, volume attachment config
    - Write `RestoreCompletionRecord` on success
    - Raise alert to recovery team within 5 minutes on failure
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6_

  - [ ]* 13.2 Write unit tests for EC2 restore handler
    - Test launch template selection by `workload_type` tag
    - Test that `IRE-Restore-SG` is assigned and public IP is absent
    - Test that backup metadata tags are applied identically to restored instance
    - Test failure alert fires within 5-minute threshold (use moto EC2 mock)
    - _Requirements: 5.1, 5.3, 5.4, 5.5_

- [ ] 14. Workload-specific recovery: RDS
  - [ ] 14.1 Implement RDS restore handler
    - Create `ire_orchestrator/workloads/rds_restore.py`
    - Implement `RDSRestoreHandler.restore(workload_id, snapshot_id, workflow_id)`
    - Call `rds_reencrypt` helper to invoke `restore_db_instance_from_db_snapshot` with IRE RDS CMK, IRE private subnet group, `IRE-RDS-SG`, `PubliclyAccessible=False`
    - Verify `PubliclyAccessible=False` and `StorageEncrypted=True` before instance is `available`
    - Write original `parameter_group_name` and `option_group_name` to restore metadata record
    - Emit alert within 5 minutes on engine version incompatibility
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6_

  - [ ]* 14.2 Write unit tests for RDS restore handler
    - Test that `PubliclyAccessible` is set to `False` before instance is available
    - Test that IRE RDS CMK ARN is used at restore time
    - Test that original parameter group and option group names are written to metadata
    - Test engine version incompatibility alert fires within 5 minutes
    - _Requirements: 6.2, 6.4, 6.5, 6.6_

- [ ] 15. Workload-specific recovery: EKS
  - [ ] 15.1 Implement EKS restore handler
    - Create `ire_orchestrator/workloads/eks_restore.py`
    - Implement `EKSRestoreHandler.restore(workload_id, backup_id, workflow_id)`
    - Restore persistent volume snapshots into IRE EKS cluster, re-encrypting with `ire/eks-pv` CMK before pods reach Running state
    - Redeploy Kubernetes manifests: original namespace structure, resource quotas, service account configs
    - Apply Kubernetes NetworkPolicies: allow inbound from scanner endpoints only, deny all other inbound
    - Disable all external LoadBalancer services on restore
    - Implement registry substitution: if Production registry unreachable within 30s, substitute with IRE internal registry and log original + substitute refs
    - On partial failure: remove all partially restored resources and report failure
    - Complete within 60 minutes of initiation
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7_

  - [ ]* 15.2 Write unit tests for EKS restore handler
    - Test PV encryption uses `ire/eks-pv` CMK before pod reaches Running state
    - Test NetworkPolicies permit only scanner endpoint CIDRs
    - Test registry substitution triggers when Production registry does not respond within 30s, with both refs logged
    - Test external LoadBalancer services are disabled on restore
    - Test partial failure triggers full resource cleanup
    - _Requirements: 7.2, 7.3, 7.5, 7.6, 7.7_

- [ ] 16. Workload-specific recovery: Lambda
  - [ ] 16.1 Implement Lambda restore handler
    - Create `ire_orchestrator/workloads/lambda_restore.py`
    - Implement `LambdaRestoreHandler.restore(workload_id, backup_id, workflow_id)`
    - Deploy function via Lambda API: code package, runtime, handler, memory, timeout, layers, environment variables
    - Encrypt environment variables at rest with `ire/lambda-env` CMK
    - Deploy inside VPC with security group restricting outbound to scanner endpoints only
    - Record original execution role ARN in restore metadata (do NOT apply as active role)
    - Derive IRE-scoped execution role from original policy, scoped to IRE resources only; apply as active execution role
    - Scan env vars for Production account ARN patterns; set function state to DISABLED and record flagged variable names if found
    - On partial failure: leave resources disabled, record function name and failure reason
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7_

  - [ ]* 16.2 Write unit tests for Lambda restore handler
    - Test env var encryption uses `ire/lambda-env` CMK
    - Test original execution role ARN is recorded in metadata but not applied as active role
    - Test Production ARN pattern detection: function set to DISABLED with flagged variable names recorded
    - Test IRE-scoped execution role is derived and applied
    - Test partial failure leaves resources disabled with failure reason logged
    - _Requirements: 8.2, 8.4, 8.5, 8.6, 8.7_

- [ ] 17. Checkpoint — workload recovery handlers
  - Ensure unit tests for all four workload restore handlers pass
  - Ensure property tests 3 and 4 pass (re-encryption invariant and retry-before-halt)
  - Ask the user if questions arise before proceeding to scanning orchestration.

- [ ] 18. IRE Scanning Orchestrator — Step Functions state machine
  - [ ] 18.1 Implement Step Functions state machine definition
    - Create `ire_orchestrator/state_machine/definition.py`
    - Define Express Workflow states matching all transitions in design: DeclareEvent → InitiateRestore → WaitForRestore → RestoreComplete → EnforceIsolation → TriggerTanium → ... → DRActivationReady
    - Implement 6-hour RTO checkpoint: parallel EventBridge timer branch fires Lambda at T+6h to notify recovery team if pipeline has not reached QualysClean
    - Each pipeline execution keyed by unique `workflowId` (independent Express Workflow instance)
    - _Requirements: 9.1, 17.2, 17.3, 17.4_

  - [ ] 18.2 Implement network isolation enforcement on restore completion
    - Create `ire_orchestrator/state_machine/isolation_enforcer.py`
    - Apply `IRE-Restore-SG` and NACL rules within 60 seconds of workload reaching running state
    - Block all outbound connections from partially restored workload until scan job is registered
    - On SG/NACL application failure: halt workload, raise alert to operations team, retain in stopped state
    - _Requirements: 4.6, 15.6, 15.7_

  - [ ]* 18.3 Write unit tests for isolation enforcement
    - Test `IRE-Restore-SG` is applied within 60-second window (mock EC2 API)
    - Test that failure to apply SG halts workload and raises alert
    - Test outbound blocking is in place until scan job is registered
    - _Requirements: 15.6, 15.7_

- [ ] 19. Security scanning integration — Tanium
  - [ ] 19.1 Implement Tanium scanner client and orchestrator integration
    - Create `ire_orchestrator/scanners/tanium_client.py`
    - Implement `TaniumClient.initiate_scan(endpoint_ip)` → calls Tanium REST API over TLS 1.2+
    - Implement `TaniumClient.get_scan_result(scan_id)` → returns `ScanResult` with status CLEAN | FINDING | TIMEOUT | ERROR
    - Implement `TaniumClient.execute_remediation(scan_id, finding_id)` → triggers automated remediation, returns `RemediationResult`
    - Enforce 2-hour scan timeout; treat timeout as scan failure
    - Enforce 30-minute remediation timeout; trigger quarantine on timeout
    - Write `ScanResult` record to Centralised Logging S3 on completion (initiation time, scanner identity, completion time, outcome)
    - _Requirements: 9.2, 9.7, 9.9, 10.1, 10.3_

  - [ ]* 19.2 Write unit tests for Tanium client
    - Test clean result path: `ScanResult.status == CLEAN` on zero threats
    - Test timeout path: scan treated as failure after 2 hours
    - Test remediation audit record includes tool name, threat ID, remediation status
    - Test 30-minute remediation timeout triggers quarantine
    - _Requirements: 9.2, 9.7, 10.1, 10.3_

- [ ] 20. Security scanning integration — CrowdStrike
  - [ ] 20.1 Implement CrowdStrike scanner client and orchestrator integration
    - Create `ire_orchestrator/scanners/crowdstrike_client.py`
    - Implement `CrowdStrikeClient.initiate_scan(endpoint_ip)` → calls CrowdStrike API over TLS 1.2+
    - Implement `CrowdStrikeClient.get_scan_result(scan_id)` → returns `ScanResult`
    - Implement `CrowdStrikeClient.execute_remediation(scan_id, detection_id)` → isolates process/file and attempts remediation
    - Enforce 2-hour scan timeout; 30-minute remediation timeout
    - Write `ScanResult` record to Centralised Logging S3 on completion
    - _Requirements: 9.3, 9.7, 9.9, 10.2, 10.3_

  - [ ]* 20.2 Write unit tests for CrowdStrike client
    - Test clean result path: `ScanResult.status == CLEAN` on zero detections
    - Test timeout detection and failure classification after 2 hours
    - Test remediation audit record includes tool name, detection ID, remediation status
    - _Requirements: 9.3, 9.7, 10.2_

- [ ] 21. Security scanning integration — Qualys
  - [ ] 21.1 Implement Qualys scanner client and orchestrator integration
    - Create `ire_orchestrator/scanners/qualys_client.py`
    - Implement `QualysClient.initiate_scan(target_ip)` → calls Qualys API over TLS 1.2+
    - Implement `QualysClient.get_scan_result(scan_id)` → returns `ScanResult` with per-finding CVSS scores
    - Clean status: zero findings with CVSS v3 ≥ 7.0
    - Enforce 4-hour scan timeout; treat timeout as scan failure
    - Write `ScanResult` record to Centralised Logging S3 on completion
    - _Requirements: 9.4, 9.7, 9.9_

  - [ ]* 21.2 Write unit tests for Qualys client
    - Test clean status when all findings have CVSS < 7.0
    - Test finding status when at least one finding has CVSS ≥ 7.0
    - Test 4-hour timeout treated as scan failure
    - _Requirements: 9.4, 9.7_

- [ ] 22. Scanning orchestration — sequencing, quarantine, and rescan
  - [ ] 22.1 Implement scan sequencing orchestrator
    - Create `ire_orchestrator/scanning_orchestrator.py`
    - Implement `ScanningOrchestrator.run_scan_cycle(workflow_id, workload_id)`:
      1. Invoke `TaniumClient.initiate_scan()`; wait for result
      2. On CLEAN: invoke `CrowdStrikeClient.initiate_scan()`; wait for result
      3. On CLEAN: invoke `QualysClient.initiate_scan()`; wait for result
      4. On CLEAN: return `ScanCycleResult.ALL_CLEAN`
    - Never invoke a subsequent scanner before the preceding scanner has returned a result
    - On any FINDING or TIMEOUT: invoke `quarantine_workload()` and notify security ops within 15 minutes
    - _Requirements: 9.1, 9.5, 9.6, 9.7_

  - [ ] 22.2 Implement quarantine mechanism
    - Create `ire_orchestrator/scanning_orchestrator.py` (extend)
    - Implement `quarantine_workload(workload_id, workflow_id, finding)`:
      - Move workload to Quarantine Subnet (10.100.10.0/24)
      - Replace security group with `IRE-Quarantine-SG`
      - Set pipeline state to QUARANTINED in WorkflowRegistry
      - Notify security operations team within 15 minutes via SNS
      - Write quarantine event to audit log (workload ID, triggering tool, timestamp)
    - _Requirements: 9.5, 10.5_

  - [ ] 22.3 Implement post-cleansing rescan logic
    - Implement `ScanningOrchestrator.handle_remediation_result(workflow_id, remediation_result)`:
      - On success: restart full scan cycle from Tanium
      - On failure or timeout (>30 min): quarantine workload and escalate to manual review
    - Promotion only allowed when all three scanners return clean on the same cycle
    - _Requirements: 10.3, 10.4_

  - [ ]* 22.4 Write property test for scan sequencing invariant (Property 5)
    - **Property 5: Scan sequencing invariant**
    - **Validates: Requirements 9.1**
    - For any completed restore event, scan invocations always follow the order Tanium → CrowdStrike → Qualys, and no subsequent scanner is invoked before the preceding scanner has returned a result
    - Tag: `Feature: aws-cyber-recovery-ire, Property 5: Scan sequencing invariant`

  - [ ]* 22.5 Write property test for quarantine on high-severity finding (Property 6)
    - **Property 6: Quarantine on high-severity finding**
    - **Validates: Requirements 9.5**
    - For any scan result where at least one finding has CVSS v3 ≥ 7.0 or vendor-native severity of Critical/High, the workload is moved to quarantine subnet, security group replaced with deny-all, and promotion pipeline halted — regardless of which scanner produced the finding
    - Tag: `Feature: aws-cyber-recovery-ire, Property 6: Quarantine on high-severity finding`

  - [ ]* 22.6 Write property test for promotion gate — all scanners clean (Property 7)
    - **Property 7: Promotion gate — all scanners clean**
    - **Validates: Requirements 9.6**
    - A promotion attempt succeeds if and only if all three scanners have returned clean status for that workload on the same scan cycle; any absent, failed, or finding-bearing result causes the promotion attempt to be rejected
    - Tag: `Feature: aws-cyber-recovery-ire, Property 7: Promotion gate — all scanners clean`

  - [ ]* 22.7 Write unit tests for scan orchestration
    - Test security ops notification fires within 15 minutes of quarantine trigger
    - Test `ScanResult` records are written to immutable S3 audit log for each scan
    - Test rescan restarts from Tanium after successful remediation
    - _Requirements: 9.5, 9.7, 9.9, 10.4_

- [ ] 23. Checkpoint — scanning pipeline
  - Ensure all scanner unit tests pass
  - Ensure property tests 5, 6, and 7 pass
  - Ask the user if questions arise before proceeding to ISO connectivity and promotion.

- [ ] 24. ISO Environment connectivity
  - [ ] 24.1 Implement ISO Environment CDK constructs for service endpoints
    - Create `ire_infra/stacks/iso_connectivity_stack.py`
    - Define TGW attachment for ISO Environment account
    - Configure PA Firewall zone policy: allow IRE → ISO on AD (TCP/UDP 389, 636), DNS (TCP/UDP 53), Monitoring (TCP 443), PKI (TCP 443, 80); deny all other IRE-to-ISO
    - Provision dedicated IRE service account scope in AD (separate from Production accounts)
    - _Requirements: 13.1, 13.2, 13.3_

  - [ ] 24.2 Implement DNS zone allowlist enforcement
    - Create `ire_orchestrator/iso/dns_allowlist.py`
    - Implement `DNSAllowlistEnforcer.resolve(zone)` → returns resolution if zone is on allowlist, else raises `DNSZoneNotPermitted` (REFUSED/NXDOMAIN)
    - Never silently drop or forward a non-allowlisted query
    - Configure IRE VPC DHCP options set to point to ISO DNS server
    - _Requirements: 13.4_

  - [ ] 24.3 Implement ISO fallback caching mechanism
    - Create `ire_orchestrator/iso/fallback_cache.py`
    - Implement DNS fallback: locally cached DNS entries, maximum cache TTL 24 hours
    - Implement certificate fallback: pre-staged certificates and CRLs in IRE S3 bucket (encrypted with `ire/logs` CMK); certificate client points to S3 as fallback
    - Implement ISO outage detection: CloudWatch Alarm on TGW flow metrics → SNS to operations team within 5 minutes
    - Implement auto-recovery: once ISO connectivity restored, workloads resume live ISO services (DNS TTL expiry triggers refresh)
    - _Requirements: 13.6_

  - [ ]* 24.4 Write property test for DNS allowlist enforcement (Property 8)
    - **Property 8: DNS allowlist enforcement**
    - **Validates: Requirements 13.4**
    - For any DNS query from an IRE workload to the ISO DNS Service, the response is a successful resolution if and only if the queried zone is on the approved IRE DNS zone allowlist; all other zones receive REFUSED or NXDOMAIN
    - Tag: `Feature: aws-cyber-recovery-ire, Property 8: DNS allowlist enforcement`

  - [ ]* 24.5 Write unit tests for ISO connectivity
    - Test DNS allowlist returns NXDOMAIN for zones not on allowlist
    - Test DNS fallback cache honours 24-hour maximum TTL
    - Test ISO outage alert fires within 5 minutes of connectivity loss detection
    - Test certificate fallback switches to S3 pre-staged CRLs when ISO is unavailable
    - _Requirements: 13.4, 13.6_

- [ ] 25. Promotion Service
  - [ ] 25.1 Implement checksum computation and storage
    - Create `ire_orchestrator/promotion/checksum.py`
    - Implement `ChecksumService.compute_artefact_checksums(workflow_id)` → computes SHA-256 for EBS snapshot data, RDS snapshot, ECR image digests, Lambda package; stores in WorkflowRegistry DynamoDB (encrypted with `ire/logs` CMK)
    - Implement `ChecksumService.validate_transferred_artefact(workflow_id, artefact_id, transferred_data)` → compare against stored checksum; raise `ChecksumMismatch` if different
    - _Requirements: 11.7_

  - [ ] 25.2 Implement human approval gate
    - Create `ire_orchestrator/promotion/approval_gate.py`
    - Implement `ApprovalGate.request_approval(workflow_id, workload_id, scan_summary)`:
      - Publish to SNS approval topic with workload ID, scan results summary, pre-signed approval URL (4-hour TTL)
      - EventBridge rule monitors for approval event
    - Implement `ApprovalGate.handle_expiry(workflow_id)`: transition state to APPROVAL_EXPIRED, notify recovery team, require re-initiation
    - _Requirements: 11.3_

  - [ ] 25.3 Implement artefact transfer and promotion Lambda
    - Create `ire_orchestrator/lambdas/promotion_service.py`
    - Implement `PromotionService.promote(workflow_id, workload_id, dr_account_id)`:
      - Validate DR infrastructure pre-provisioned and type matches workload type; abort with mismatch log on failure
      - Copy EBS/RDS snapshots cross-account with DR CMK re-encryption over TLS 1.2+
      - Copy ECR images and Lambda packages to DR account
      - After each artefact transfer, call `ChecksumService.validate_transferred_artefact()`; abort and rollback all on mismatch
      - Write `PromotionAuditRecord` to Centralised Logging S3
    - Implement rollback: delete all transferred artefacts in DR account, notify recovery team within 15 minutes on any failure
    - _Requirements: 11.1, 11.2, 11.4, 11.5, 11.6, 11.7_

  - [ ]* 25.4 Write property test for artefact integrity on promotion (Property 9)
    - **Property 9: Artefact integrity on promotion**
    - **Validates: Requirements 11.7**
    - For any workload artefact transferred from IRE to DR Environment, the SHA-256 checksum after transfer equals the checksum recorded at end of IRE scanning phase; if any checksum does not match, the promotion is aborted and all transferred artefacts are rolled back
    - Tag: `Feature: aws-cyber-recovery-ire, Property 9: Artefact integrity on promotion`

  - [ ]* 25.5 Write unit tests for approval gate and promotion service
    - Test approval gate publishes SNS message with correct fields and 4-hour TTL URL
    - Test approval expiry transitions state to APPROVAL_EXPIRED and notifies recovery team
    - Test DR infrastructure type mismatch aborts promotion with mismatch log
    - Test rollback deletes all transferred artefacts and notifies within 15 minutes on failure
    - _Requirements: 11.3, 11.5, 11.6_

- [ ] 26. DR Environment activation
  - [ ] 26.1 Implement DR Environment CDK stack for pre-provisioned infrastructure
    - Create `ire_infra/stacks/dr_environment_stack.py`
    - Pre-provision per-application: VPC, subnets, security groups, IAM roles, KMS CMKs, EC2 launch templates, RDS subnet groups, EKS node groups, Lambda config stubs
    - _Requirements: 12.1_

  - [ ] 26.2 Implement DR workload import validation and activation event
    - Create `ire_orchestrator/dr/activation.py`
    - Implement `DRActivation.validate_import(workload_id, artefact)` → verify artefact importable into target infrastructure; mark as activation-ready
    - Implement `DRActivation.activate(workload_id, iam_principal)` → start workload; emit CloudTrail event with workload ID, activation timestamp, IAM principal that triggered activation
    - _Requirements: 12.2, 12.4_

  - [ ]* 26.3 Write unit tests for DR activation
    - Test `validate_import` returns activation-ready on valid artefact
    - Test CloudTrail activation event includes workload ID, timestamp, IAM principal
    - _Requirements: 12.2, 12.4_

- [ ] 27. Out-of-band modification detection
  - [ ] 27.1 Implement out-of-band resource modification alert
    - Create `ire_orchestrator/compliance/oob_monitor.py`
    - EventBridge rule on CloudTrail resource modification events (EC2, RDS, Lambda, EKS API calls)
    - Implement `OOBMonitor.evaluate(event)`: query WorkflowRegistry; if modification is from authorised orchestration role with active workflow → allow; else send high-severity alert to security ops within 5 minutes via SNS → SIEM
    - _Requirements: 16.4_

  - [ ]* 27.2 Write unit tests for out-of-band modification detection
    - Test that modification from `ire-scanning-orchestrator-role` with active workflow entry is allowed without alert
    - Test that modification from any other principal, or with no active workflow, triggers high-severity alert within 5 minutes
    - _Requirements: 16.4_

- [ ] 28. Recovery audit report generation
  - [ ] 28.1 Implement end-to-end recovery audit report writer
    - Create `ire_orchestrator/compliance/audit_report.py`
    - Implement `AuditReportWriter.generate(workflow_id)` → query WorkflowRegistry + ScanResults + PromotionAuditRecord; compose `RecoveryAuditReport` JSON matching design schema
    - Write to Centralised Logging Account S3 within 30 minutes of pipeline completion
    - _Requirements: 16.3_

  - [ ]* 28.2 Write unit tests for audit report generation
    - Test report includes all required fields: backup source, restore timestamp, per-tool scan results, cleansing actions, approval record, promotion record, DR activation status
    - Test report is written to S3 within 30-minute deadline
    - _Requirements: 16.3_

- [ ] 29. Break-glass IP access control
  - [ ] 29.1 Implement IP-based access control enforcement for break-glass accounts
    - Ensure IP restriction IAM policy (from task 7.1) is enforced: deny `sts:AssumeRole` from IPs outside approved CIDR list
    - Implement `BreakGlassAccessMonitor.check_login_attempt(source_ip, approved_cidrs)` → deny and generate security alert for out-of-range IPs
    - CloudTrail event on `ConsoleLogin` from break-glass IAM user: capture IAM principal, source IP, action, timestamp
    - Increment failed attempt counter and generate CloudTrail audit event on MFA failure
    - _Requirements: 3.3, 3.5, 3.6_

  - [ ]* 29.2 Write property test for IP-based access control (Property 2)
    - **Property 2: IP-based access control**
    - **Validates: Requirements 3.5**
    - For any source IP attempting break-glass account authentication, access is permitted if and only if the source IP falls within the approved IP range list; all other source IPs are denied with a security alert generated
    - Tag: `Feature: aws-cyber-recovery-ire, Property 2: IP-based access control`

  - [ ]* 29.3 Write unit tests for break-glass access monitoring
    - Test approved IP succeeds authentication
    - Test unapproved IP triggers denial and security alert
    - Test MFA failure increments counter and generates CloudTrail event with correct fields
    - _Requirements: 3.5, 3.6_

- [ ] 30. RTO/RPO monitoring
  - [ ] 30.1 Implement RTO monitoring Lambda and 6-hour checkpoint
    - Create `ire_orchestrator/rto_monitor.py`
    - Implement independent EventBridge timer per workflow: starts at `InitiateRestore` timestamp
    - At T+6 hours: if pipeline has not reached `SCAN_COMPLETE` state, query WorkflowRegistry for current state and send SNS notification to recovery team and incident manager with current state + estimated completion time
    - _Requirements: 17.3_

  - [ ] 30.2 Implement parallel workload RTO tracking
    - Ensure each Express Workflow execution tracks its own `startTime` independently in WorkflowRegistry
    - Implement `RTOTracker.check_rto_breach(workflow_id)` → returns True if `now - startTime > 8 hours`
    - Each workload's RTO measured independently regardless of concurrent executions
    - _Requirements: 17.2, 17.4_

  - [ ]* 30.3 Write unit tests for RTO monitoring
    - Test 6-hour checkpoint notification fires at correct elapsed time
    - Test parallel executions each have independent RTO timers
    - Test `RTOTracker.check_rto_breach()` returns True only after 8 hours from individual workload's `InitiateRestore`
    - _Requirements: 17.2, 17.3, 17.4_

- [ ] 31. Integration wiring — Step Functions and Lambda integration
  - [ ] 31.1 Wire all orchestrator components into Step Functions state machine
    - Update `ire_orchestrator/state_machine/definition.py` to reference all handler functions: EC2/RDS/EKS/Lambda restore handlers, isolation enforcer, scanning orchestrator, approval gate, promotion service, RTO monitor
    - Ensure state machine transitions match the full design state diagram
    - Register state machine with `ire-scanning-orchestrator-role`
    - _Requirements: 9.1, 17.2_

  - [ ] 31.2 Wire CDK stacks into unified app entry point
    - Create `ire_infra/app.py`
    - Instantiate and connect all CDK stacks: OrgPolicyStack, IRE VPC, KMS, LoggingStack, BreakGlassIAMStack, WorkflowRegistryConstruct, CloudTrail, Config, GuardDuty, DREnvironmentStack
    - Validate cross-stack dependency ordering
    - _Requirements: all_

  - [ ]* 31.3 Write integration tests for end-to-end pipeline with mocked external APIs
    - Test end-to-end restore → scan → promotion pipeline using moto and mock Rubrik/scanner API fixtures
    - Verify CloudTrail audit event is generated for break-glass account usage
    - Verify KMS cross-account snapshot re-encryption produces correct CMK ARN on restored resource
    - Verify S3 Object Lock enforcement blocks deletion of log objects
    - Verify GuardDuty finding is forwarded to SIEM within 5 minutes
    - Verify ISO fallback caching activates when ISO Environment is unavailable
    - _Requirements: 3.3, 4.2, 5.2, 16.6, 16.7, 13.6_

- [ ] 32. Final checkpoint — full pipeline validation
  - Ensure all unit tests pass across all modules
  - Ensure all 9 property-based tests pass (Properties 1–9)
  - Ensure all integration tests pass against mocked infrastructure
  - Ensure all smoke tests pass against synthesised CDK CloudFormation templates
  - Ask the user if questions arise before considering the implementation complete.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP; core pipeline functionality will still be complete
- All 9 design correctness properties are covered by dedicated property-based tests (Hypothesis, min 100 iterations each)
- Each property test is tagged `Feature: aws-cyber-recovery-ire, Property N: <title>` as specified in the design
- Checkpoints (tasks 9, 17, 23, 32) require all tests at that stage to pass before proceeding
- CDK stacks use Python CDK (`aws-cdk-lib`) throughout to match the Python runtime of orchestration code
- Smoke tests validate synthesised CloudFormation templates and can be run without deploying to AWS
- Integration tests use `moto` for AWS service mocking and stub fixtures for Rubrik, Tanium, CrowdStrike, Qualys APIs
- TLS 1.2+ enforcement is applied at the HTTP client layer (`SSLContext`) for all external API calls (Rubrik, Tanium, CrowdStrike, Qualys)

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1"] },
    { "id": 1, "tasks": ["2.1", "3.1", "4.1", "5.1", "5.2", "6.1", "7.1", "8.1"] },
    { "id": 2, "tasks": ["2.2", "3.2", "4.2", "4.3", "5.3", "5.4", "6.2", "6.3", "6.4", "7.2", "8.2", "8.3"] },
    { "id": 3, "tasks": ["3.3", "4.4", "4.5", "5.5", "6.5", "7.3", "8.4"] },
    { "id": 4, "tasks": ["7.4", "7.5", "8.4", "10.1"] },
    { "id": 5, "tasks": ["10.2", "11.1"] },
    { "id": 6, "tasks": ["10.3", "11.2", "11.3"] },
    { "id": 7, "tasks": ["11.4", "11.5", "12.1", "12.2"] },
    { "id": 8, "tasks": ["12.3", "12.4", "13.1", "14.1", "15.1", "16.1"] },
    { "id": 9, "tasks": ["13.2", "14.2", "15.2", "16.2"] },
    { "id": 10, "tasks": ["18.1", "19.1", "20.1", "21.1", "24.1", "25.1", "26.1"] },
    { "id": 11, "tasks": ["18.2", "19.2", "20.2", "21.2", "24.2", "25.2", "26.2"] },
    { "id": 12, "tasks": ["18.3", "22.1", "24.3", "25.3", "26.3", "27.1", "28.1", "29.1", "30.1", "30.2"] },
    { "id": 13, "tasks": ["22.2", "22.3", "24.4", "24.5", "25.4", "25.5", "27.2", "28.2", "29.2", "29.3", "30.3"] },
    { "id": 14, "tasks": ["22.4", "22.5", "22.6", "22.7"] },
    { "id": 15, "tasks": ["31.1", "31.2"] },
    { "id": 16, "tasks": ["31.3"] }
  ]
}
```
