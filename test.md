# Test Specification — AWS Cyber Recovery IRE Solution

## 1. Introduction

### 1.1 Purpose

This document defines the complete test specification for the AWS Cyber Recovery Isolated Recovery Environment (IRE) solution. It provides structured test cases, property-based test definitions, security tests, and performance tests derived directly from the requirements in `requirements.md` and the technical design in `design.md`.

### 1.2 Scope

The test specification covers all eighteen functional and non-functional requirements, including:

- Rubrik backup protection of production workloads (Requirement 1)
- IRE account isolation and network placement (Requirement 2)
- Break-glass access control (Requirement 3)
- Workload restore into IRE for EC2/EBS, RDS, EKS, and Lambda (Requirements 4–8)
- Security scanning and automated cleansing via Tanium, CrowdStrike, and Qualys (Requirements 9–10)
- Promotion to DR Environment and DR activation (Requirements 11–12)
- ISO Environment foundational services (Requirement 13)
- KMS encryption at rest (Requirement 14)
- Network isolation and traffic control (Requirement 15)
- Compliance, auditability, and logging (Requirement 16)
- RTO/RPO targets (Requirement 17)
- IAM least privilege (Requirement 18)

### 1.3 Objectives

- Verify that every acceptance criterion in Requirements 1–18 is satisfied.
- Validate the nine correctness properties defined in the design through property-based testing using Python Hypothesis.
- Confirm that security boundaries (network isolation, IAM, KMS, SCP) are enforced and cannot be bypassed.
- Demonstrate that RTO (8 hours) and RPO (24 hours) targets are achievable under load.
- Establish a fully traceable mapping between each acceptance criterion and at least one test case.

### 1.4 References

- `requirements.md` — Functional and non-functional requirements for the IRE solution
- `design.md` — Technical design including architecture, data models, state machine, and correctness properties
- AWS Organizations Service Control Policies documentation
- Rubrik Cloud Data Management REST API documentation
- Tanium, CrowdStrike, Qualys API references

### 1.5 Test Conventions

Test case identifiers follow the pattern `TC-Rn-nn` where `n` is the requirement number and `nn` is a sequential number within that requirement. Security test cases use the prefix `SECSCAN`. Performance test cases use the prefix `PERF`. Property-based test cases use the prefix `PBT-Pn`.

Pass/Fail criteria are stated in terms of observable, deterministic outcomes. All timing assertions include a 10% tolerance margin unless otherwise specified (e.g., "within 15 minutes" means the event must occur no later than 16 minutes 30 seconds after the trigger).


---

## 2. Test Environment

### 2.1 AWS Multi-Account Test Setup

Testing requires a dedicated non-production AWS Organization that mirrors the production account structure. The following accounts must be provisioned within a dedicated test OU:

| Account | Role | Notes |
|---|---|---|
| IRE Test Account | Isolated Recovery Environment under test | Mirrors production IRE configuration |
| DR Test Account (per application) | Target for validated workload promotion | Pre-provisioned with matching infrastructure |
| ISO Test Account | Foundational services (AD, DNS, monitoring, PKI) | Isolated from any production AD domain |
| Centralised Logging Test Account | Immutable audit log storage | S3 Object Lock enabled in compliance mode |
| Rubrik Management Test Account | Rubrik CDM control plane | Points only at test environment accounts |
| Production Simulator Account | Source workloads and Rubrik backup source | Does not connect to real production |

All accounts must reside in the same test OU so that SCP tests reflect production policy enforcement. SCPs used in tests are identical copies of the production SCP set.

### 2.2 Network Infrastructure

- Transit Gateway shared across all test accounts, with routing tables configured identically to the production design.
- Palo Alto Firewall (virtual or physical) deployed in a dedicated Firewall VPC, with zone policies matching the production IRE zone policy set.
- No internet gateways or NAT gateways provisioned in the IRE test VPC.
- VPC endpoints for S3, KMS, SSM, and ECR provisioned in the IRE test VPC.

### 2.3 Rubrik Test Instance

- A dedicated Rubrik Cloud Data Management instance configured to protect test workloads in the Production Simulator Account.
- Backup jobs scheduled to run against registered test workloads (EC2, RDS, EKS, Lambda).
- Restore operations target the IRE Test Account exclusively.
- TLS 1.2 enforced on all Rubrik API and data channels.

### 2.4 Security Scanning Tools

| Tool | Mode | Configuration |
|---|---|---|
| Tanium | Sandbox / test tenant | Agents pre-installed on test AMIs; points to test Tanium Server |
| CrowdStrike | Sandbox CID | Falcon Sensor pre-installed on test AMIs; isolated CID |
| Qualys | Sandbox subscription | Virtual Scanner Appliance pre-deployed in IRE test restore subnet |

All scanner API credentials are stored in AWS Secrets Manager in the IRE Test Account, encrypted with the IRE test break-glass CMK.

### 2.5 Test Workloads

The following test workloads are registered in the backup inventory and pre-backed-up before test execution:

| Workload ID | Type | Description |
|---|---|---|
| `test-ec2-linux-01` | EC2/EBS | Linux server with sample application data on EBS volumes |
| `test-ec2-windows-01` | EC2/EBS | Windows server with sample data |
| `test-rds-mysql-01` | RDS MySQL | MySQL 8.0 instance with sample schema |
| `test-rds-postgres-01` | RDS PostgreSQL | PostgreSQL 14 instance with sample schema |
| `test-eks-cluster-01` | EKS | EKS cluster with two namespaces, PVs, and sample manifests |
| `test-lambda-01` | Lambda | Python 3.11 Lambda with environment variables |
| `test-lambda-prod-arn` | Lambda | Lambda with an environment variable containing a Production ARN pattern (negative test) |

A separate "infected" EC2 image (`test-ec2-infected-01`) is maintained with a known EICAR test string and a simulated CVSS 8.0 vulnerability for quarantine and cleansing tests.

### 2.6 Test Data and Credentials

- Break-glass test credentials stored in Secrets Manager; approved test IP ranges configured in the IAM policy.
- A dedicated "non-approved" IP range reserved for negative access-control tests.
- Test KMS CMKs provisioned per workload type in IRE Test Account and DR Test Account with distinct ARNs.
- CloudTrail, AWS Config, GuardDuty, and S3 Object Lock enabled in all test accounts.


---

## 3. Test Approach

### 3.1 Unit Tests

Unit tests target individual functions, classes, and modules of the IRE orchestration logic in isolation, using mocks for external dependencies (Rubrik API, scanner APIs, AWS services). They validate business logic correctness, field validation, state transition logic, timeout detection, and error classification.

Tooling: `pytest` with `unittest.mock` and `moto` (for AWS service mocking).

Coverage target: 90% line coverage on all orchestration modules.

### 3.2 Integration Tests

Integration tests verify the interaction between IRE components and real (or localstack-backed) AWS services. They cover cross-account KMS snapshot re-encryption, CloudTrail event generation, S3 Object Lock enforcement, GuardDuty forwarding, and Transit Gateway routing behaviour.

Tooling: `pytest` with real AWS test accounts or `localstack` for S3/KMS/DynamoDB interactions; Terraform or AWS CDK to provision test infrastructure.

### 3.3 Property-Based Tests

Property-based tests verify universal invariants defined in the design's Correctness Properties section (Properties 1–9). They use the Python `Hypothesis` library to generate diverse inputs and assert that invariants hold across all generated cases.

Each test runs a minimum of 100 examples (`@settings(max_examples=100)`). Each test is tagged with its design property reference using the format:

```
# Feature: aws-cyber-recovery-ire, Property N: <property name>
```

Tooling: `pytest`, `hypothesis`, `hypothesis[strategies]`.

### 3.4 Security Tests

Security tests attempt adversarial actions — network bypasses, IAM escalation, log tampering, unauthenticated promotions — and verify they are blocked at the correct enforcement point. Each test has an explicit deny assertion and verifies that an appropriate audit log entry is generated.

Tooling: AWS CLI, `boto3`, custom test scripts using the Production Simulator Account as the attacker origin.

### 3.5 End-to-End Recovery Tests

End-to-end tests execute the complete recovery pipeline — backup → declare recovery event → restore → scan → approve → promote → DR activation — for each workload type. These tests validate pipeline state transitions, timing constraints, and the contents of the final recovery audit report.

They require all test environment components to be active simultaneously. End-to-end tests are run in a dedicated test window and are not part of the standard CI/CD pipeline due to their duration.

### 3.6 Performance Tests (RTO/RPO Validation)

Performance tests measure actual elapsed time for each pipeline stage and the overall pipeline, comparing against defined RTO (8 hours) and RPO (24 hours) targets. Timing measurements are recorded using pipeline start/end timestamps from the IRE Workflow Registry (DynamoDB).

Concurrent pipeline tests launch five independent workload recovery pipelines simultaneously to validate that the RTO is met per workload when the IRE is under concurrent load.


---

## 4. Property-Based Tests (Correctness Properties)

Each of the nine correctness properties from the design is implemented as a Hypothesis-driven property-based test. All tests are located in `tests/property/` and are tagged with their design property identifier.

---

### PBT-P1: Backup Completeness

**Property ID:** P1
**Validates:** Requirement 1.7

**Formal Description:**
For any backup job execution and registered workload inventory, every workload present in the inventory at job execution time must either appear in the completed backup job's output records, or an alert must be generated identifying that workload by its workload identifier. There is no third outcome.

**Test Input Generators:**
- `workload_inventory`: a set of between 1 and 50 workload identifiers (UUIDs)
- `backup_job_output`: a subset of the inventory (may be a proper subset to simulate omissions)

**Expected Invariants:**
- Every workload in `workload_inventory` that is absent from `backup_job_output` must appear in `generated_alerts` with a matching workload identifier.
- No alert is generated for a workload that is present in `backup_job_output`.

**Python Hypothesis Example:**

```python
# Feature: aws-cyber-recovery-ire, Property 1: Backup completeness
from hypothesis import given, settings
from hypothesis import strategies as st
import uuid

workload_id_strategy = st.uuids().map(str)

@given(
    inventory=st.frozensets(workload_id_strategy, min_size=1, max_size=50),
    backed_up=st.data()
)
@settings(max_examples=100)
def test_backup_completeness(inventory, backed_up):
    backed_up_set = backed_up.draw(st.frozensets(st.sampled_from(sorted(inventory))))
    omitted = inventory - backed_up_set

    result = backup_service.verify_completeness(
        inventory=list(inventory),
        backup_job_output=list(backed_up_set)
    )

    for wid in omitted:
        assert wid in result.alerts, f"Expected alert for omitted workload {wid}"
    for wid in backed_up_set:
        assert wid not in result.alerts, f"Unexpected alert for backed-up workload {wid}"
```

---

### PBT-P2: IP-Based Access Control

**Property ID:** P2
**Validates:** Requirement 3.5

**Formal Description:**
For any source IP address attempting break-glass account authentication, the authentication must be permitted if and only if the source IP falls within the approved IP range list. Any source IP outside the approved ranges must be denied and a security alert must be generated. There is no scenario in which a non-approved IP is permitted or an approved IP is denied.

**Test Input Generators:**
- `approved_cidrs`: a list of between 1 and 5 CIDR blocks (e.g., `192.168.1.0/24`)
- `source_ip`: an IPv4 address, drawn either from within or outside the approved CIDRs

**Expected Invariants:**
- If `source_ip` is within any approved CIDR: `result.permitted == True` and `result.alert_generated == False`.
- If `source_ip` is outside all approved CIDRs: `result.permitted == False` and `result.alert_generated == True`.

**Python Hypothesis Example:**

```python
# Feature: aws-cyber-recovery-ire, Property 2: IP-based access control
from hypothesis import given, settings
from hypothesis import strategies as st
import ipaddress

approved_cidr_strategy = st.just("10.0.1.0/24")  # fixed for determinism; extend as needed

@given(
    source_ip=st.ip_addresses(v=4).map(str),
    approved_cidrs=st.just(["10.0.1.0/24", "10.0.2.0/24"])
)
@settings(max_examples=100)
def test_ip_access_control(source_ip, approved_cidrs):
    result = break_glass_manager.evaluate_login_attempt(
        source_ip=source_ip,
        approved_cidrs=approved_cidrs
    )
    in_approved = any(
        ipaddress.ip_address(source_ip) in ipaddress.ip_network(c)
        for c in approved_cidrs
    )
    if in_approved:
        assert result.permitted is True
        assert result.alert_generated is False
    else:
        assert result.permitted is False
        assert result.alert_generated is True
```

---

### PBT-P3: Re-encryption on Restore

**Property ID:** P3
**Validates:** Requirement 4.2

**Formal Description:**
For any restored workload of any type (EC2/EBS, RDS, EKS persistent volume, Lambda environment variable), the encryption key ARN on every restored resource must match the IRE account's designated CMK ARN for that workload type, and must not match any Production account CMK ARN.

**Test Input Generators:**
- `workload_type`: one of `EC2_EBS`, `RDS`, `EKS_PV`, `LAMBDA`
- `source_cmk_arn`: a Production account CMK ARN
- `ire_cmk_arns`: a dict mapping workload type to IRE CMK ARN
- `restore_result`: a simulated restore result object containing `encrypted_with` field

**Expected Invariants:**
- `restore_result.encrypted_with` must equal `ire_cmk_arns[workload_type]`.
- `restore_result.encrypted_with` must not equal `source_cmk_arn`.

**Python Hypothesis Example:**

```python
# Feature: aws-cyber-recovery-ire, Property 3: Re-encryption on restore
from hypothesis import given, settings
from hypothesis import strategies as st

workload_types = ["EC2_EBS", "RDS", "EKS_PV", "LAMBDA"]

ire_cmk_arns = {
    "EC2_EBS":  "arn:aws:kms:eu-west-1:111111111111:key/ire-ebs-cmk",
    "RDS":      "arn:aws:kms:eu-west-1:111111111111:key/ire-rds-cmk",
    "EKS_PV":   "arn:aws:kms:eu-west-1:111111111111:key/ire-eks-pv-cmk",
    "LAMBDA":   "arn:aws:kms:eu-west-1:111111111111:key/ire-lambda-cmk",
}
prod_cmk_arn = "arn:aws:kms:eu-west-1:999999999999:key/prod-backup-cmk"

@given(workload_type=st.sampled_from(workload_types))
@settings(max_examples=100)
def test_re_encryption_on_restore(workload_type):
    result = restore_service.restore_workload(
        workload_type=workload_type,
        source_cmk_arn=prod_cmk_arn,
        ire_cmk_arns=ire_cmk_arns
    )
    assert result.encrypted_with == ire_cmk_arns[workload_type], \
        f"Expected IRE CMK for {workload_type}, got {result.encrypted_with}"
    assert result.encrypted_with != prod_cmk_arn, \
        "Restored resource must not use Production CMK"
```

---

### PBT-P4: Retry Before Halt

**Property ID:** P4
**Validates:** Requirement 4.7

**Formal Description:**
For any restore failure where the retry count is strictly less than 3, the system must schedule exactly one automatic retry after a 2-minute wait, without halting the pipeline or dispatching a failure notification. For any failure where the retry count equals or exceeds 3, the pipeline must halt and a failure notification must be dispatched. Exactly one retry is scheduled per failure event — never zero, never two.

**Test Input Generators:**
- `retry_count`: an integer between 0 and 5
- `failure_event`: a simulated restore failure event object

**Expected Invariants:**
- If `retry_count < 3`: `result.retries_scheduled == 1`, `result.pipeline_halted == False`, `result.notification_dispatched == False`.
- If `retry_count >= 3`: `result.retries_scheduled == 0`, `result.pipeline_halted == True`, `result.notification_dispatched == True`.

**Python Hypothesis Example:**

```python
# Feature: aws-cyber-recovery-ire, Property 4: Retry before halt
from hypothesis import given, settings
from hypothesis import strategies as st

@given(retry_count=st.integers(min_value=0, max_value=5))
@settings(max_examples=100)
def test_retry_before_halt(retry_count):
    result = restore_service.handle_failure(retry_count=retry_count)
    if retry_count < 3:
        assert result.retries_scheduled == 1
        assert result.pipeline_halted is False
        assert result.notification_dispatched is False
    else:
        assert result.retries_scheduled == 0
        assert result.pipeline_halted is True
        assert result.notification_dispatched is True
```

---

### PBT-P5: Scan Sequencing Invariant

**Property ID:** P5
**Validates:** Requirement 9.1

**Formal Description:**
For any completed restore event, the scanning system must always invoke Tanium first, CrowdStrike second, and Qualys third. No subsequent scanner may be invoked before the preceding scanner has returned a result. This invariant holds regardless of restore event content, workload type, or workload identifier.

**Test Input Generators:**
- `restore_event`: a completed restore event with varying workload types, identifiers, and metadata fields

**Expected Invariants:**
- `scan_invocations[0].scanner == Scanner.TANIUM`
- `scan_invocations[1].scanner == Scanner.CROWDSTRIKE`
- `scan_invocations[2].scanner == Scanner.QUALYS`
- Each invocation's `invoked_after` timestamp is strictly greater than the preceding scanner's `result_received` timestamp.

**Python Hypothesis Example:**

```python
# Feature: aws-cyber-recovery-ire, Property 5: Scan sequencing invariant
from hypothesis import given, settings
from hypothesis import strategies as st
from ire.models import Scanner

restore_event_strategy = st.builds(
    RestoreEvent,
    workload_id=st.uuids().map(str),
    workload_type=st.sampled_from(["EC2_EBS", "RDS", "EKS", "LAMBDA"]),
    restore_duration_seconds=st.integers(min_value=60, max_value=3600)
)

@given(restore_event=restore_event_strategy)
@settings(max_examples=100)
def test_scan_sequencing_invariant(restore_event):
    result = orchestrator.handle_restore_complete(restore_event)
    invocations = result.scan_invocations
    assert len(invocations) == 3
    assert invocations[0].scanner == Scanner.TANIUM
    assert invocations[1].scanner == Scanner.CROWDSTRIKE
    assert invocations[2].scanner == Scanner.QUALYS
    for i in range(1, len(invocations)):
        assert invocations[i].invoked_at >= invocations[i-1].result_received_at
```

---

### PBT-P6: Quarantine on High-Severity Finding

**Property ID:** P6
**Validates:** Requirement 9.5

**Formal Description:**
For any scan result that contains at least one finding with a CVSS v3 score of 7.0 or above, or a vendor-native severity of Critical or High, the workload must be moved to the quarantine subnet, have its security group replaced with the deny-all quarantine group, and have its promotion pipeline halted — regardless of which scanner produced the finding, which workload type is involved, or what other findings were present.

**Test Input Generators:**
- `scanner`: one of `TANIUM`, `CROWDSTRIKE`, `QUALYS`
- `findings`: a list of Finding objects with varying CVSS scores and severities, where at least one has `cvss_score >= 7.0` or `severity in {CRITICAL, HIGH}`
- `workload_id`: a UUID string

**Expected Invariants:**
- `result.workload_subnet == QUARANTINE_SUBNET`
- `result.security_group == IRE_QUARANTINE_SG`
- `result.promotion_halted == True`
- `result.secops_notified == True`

**Python Hypothesis Example:**

```python
# Feature: aws-cyber-recovery-ire, Property 6: Quarantine on high-severity finding
from hypothesis import given, settings
from hypothesis import strategies as st
from ire.models import Scanner, Severity, Finding

high_severity_finding = st.builds(
    Finding,
    cvss_score=st.floats(min_value=7.0, max_value=10.0),
    severity=st.sampled_from([Severity.CRITICAL, Severity.HIGH])
)

@given(
    scanner=st.sampled_from(list(Scanner)),
    findings=st.lists(high_severity_finding, min_size=1, max_size=5),
    workload_id=st.uuids().map(str)
)
@settings(max_examples=100)
def test_quarantine_on_high_severity(scanner, findings, workload_id):
    result = orchestrator.process_scan_result(
        workload_id=workload_id,
        scanner=scanner,
        findings=findings
    )
    assert result.workload_subnet == QUARANTINE_SUBNET
    assert result.security_group == IRE_QUARANTINE_SG
    assert result.promotion_halted is True
    assert result.secops_notified is True
```

---

### PBT-P7: Promotion Gate — All Scanners Clean

**Property ID:** P7
**Validates:** Requirement 9.6

**Formal Description:**
For any workload, a promotion attempt must succeed if and only if all three scanners (Tanium, CrowdStrike, Qualys) have returned a clean status for that workload on the same scan cycle. A promotion attempt where any scanner's status is absent, failed, timed out, or contains a finding must be rejected without executing any transfer.

**Test Input Generators:**
- `tanium_status`: one of `CLEAN`, `FINDING`, `TIMEOUT`, `ERROR`, `ABSENT`
- `crowdstrike_status`: same set
- `qualys_status`: same set

**Expected Invariants:**
- If all three statuses are `CLEAN`: `result.promotion_permitted == True`.
- For any other combination: `result.promotion_permitted == False`.

**Python Hypothesis Example:**

```python
# Feature: aws-cyber-recovery-ire, Property 7: Promotion gate — all scanners clean
from hypothesis import given, settings
from hypothesis import strategies as st
from ire.models import ScanStatus

scan_status_strategy = st.sampled_from(list(ScanStatus))

@given(
    tanium=scan_status_strategy,
    crowdstrike=scan_status_strategy,
    qualys=scan_status_strategy
)
@settings(max_examples=100)
def test_promotion_gate_all_scanners_clean(tanium, crowdstrike, qualys):
    result = orchestrator.evaluate_promotion_eligibility(
        tanium_status=tanium,
        crowdstrike_status=crowdstrike,
        qualys_status=qualys
    )
    all_clean = (
        tanium == ScanStatus.CLEAN and
        crowdstrike == ScanStatus.CLEAN and
        qualys == ScanStatus.CLEAN
    )
    assert result.promotion_permitted == all_clean
```

---

### PBT-P8: DNS Allowlist Enforcement

**Property ID:** P8
**Validates:** Requirement 13.4

**Formal Description:**
For any DNS query submitted from an IRE workload to the ISO DNS Service, the response must be a successful resolution if and only if the queried DNS zone appears in the approved IRE DNS zone allowlist. For all other zones, the response must be REFUSED or NXDOMAIN. The DNS service must never silently drop or transparently forward queries for non-approved zones.

**Test Input Generators:**
- `allowlisted_zones`: a fixed set (e.g., `{ire.internal, corp.internal, scanning-tools.internal}`)
- `queried_zone`: a DNS zone name drawn from a mix of approved and unapproved zones

**Expected Invariants:**
- If `queried_zone` is in `allowlisted_zones`: `result.response_type in {RESOLVED}`.
- Otherwise: `result.response_type in {REFUSED, NXDOMAIN}` and `result.forwarded == False`.

**Python Hypothesis Example:**

```python
# Feature: aws-cyber-recovery-ire, Property 8: DNS allowlist enforcement
from hypothesis import given, settings
from hypothesis import strategies as st

ALLOWED_ZONES = frozenset(["ire.internal", "corp.internal", "scanning-tools.internal"])
all_zones = list(ALLOWED_ZONES) + [
    "production.internal", "external.com", "attacker.net", "unknown.local"
]

@given(queried_zone=st.sampled_from(all_zones))
@settings(max_examples=100)
def test_dns_allowlist_enforcement(queried_zone):
    result = iso_dns_service.resolve(zone=queried_zone)
    if queried_zone in ALLOWED_ZONES:
        assert result.response_type == "RESOLVED"
    else:
        assert result.response_type in ("REFUSED", "NXDOMAIN")
        assert result.forwarded is False
```

---

### PBT-P9: Artefact Integrity on Promotion

**Property ID:** P9
**Validates:** Requirement 11.7

**Formal Description:**
For any workload artefact transferred from IRE to the DR Environment, the SHA-256 checksum computed after transfer must equal the checksum recorded at the end of the IRE scanning phase. If any checksum does not match, the promotion must be aborted and all transferred artefacts must be rolled back. No partial promotion must be left in a confirmed state.

**Test Input Generators:**
- `artefacts`: a list of artefact objects, each with `artefact_id`, `ire_checksum` (SHA-256 hex string), and `transferred_checksum` (may differ to simulate corruption)
- At least one artefact in some test runs has `transferred_checksum != ire_checksum`

**Expected Invariants:**
- If all `transferred_checksum == ire_checksum`: `result.promotion_confirmed == True` and `result.rolled_back_artefacts == []`.
- If any `transferred_checksum != ire_checksum`: `result.promotion_confirmed == False` and `result.rolled_back_artefacts` contains all artefact IDs.

**Python Hypothesis Example:**

```python
# Feature: aws-cyber-recovery-ire, Property 9: Artefact integrity on promotion
from hypothesis import given, settings
from hypothesis import strategies as st
import hashlib

sha256_hex = st.binary(min_size=32, max_size=32).map(lambda b: b.hex())

artefact_strategy = st.builds(
    dict,
    artefact_id=st.uuids().map(str),
    ire_checksum=sha256_hex,
    transferred_checksum=sha256_hex
)

@given(artefacts=st.lists(artefact_strategy, min_size=1, max_size=10))
@settings(max_examples=100)
def test_artefact_integrity_on_promotion(artefacts):
    result = promotion_service.validate_and_promote(artefacts=artefacts)
    all_match = all(
        a["ire_checksum"] == a["transferred_checksum"] for a in artefacts
    )
    if all_match:
        assert result.promotion_confirmed is True
        assert result.rolled_back_artefacts == []
    else:
        assert result.promotion_confirmed is False
        assert set(result.rolled_back_artefacts) == {a["artefact_id"] for a in artefacts}
```


---

## 5. Test Cases by Requirement

---

### Requirement 1: Rubrik Backup of Production Workloads

---

**TC-R1-01: Happy path backup completes and records backup event**

Requirement Reference: 1.1, 1.3
Type: Integration

Preconditions:
- All four workload types (EC2/EBS, RDS, EKS, Lambda) are registered in the backup inventory.
- A scheduled Rubrik backup job is due to run.
- KMS CMKs are active in the Production Simulator Account.

Test Steps:
1. Trigger a scheduled Rubrik backup job for all registered test workloads.
2. Wait for the job to complete.
3. Retrieve the backup completion record from the Rubrik backup log.

Expected Results:
- A backup completion record exists for each registered workload.
- Each record contains: workload identifier, timestamp (second precision), backup size in bytes, and KMS CMK ARN.
- No workload in the inventory is missing from the completion records.

Pass/Fail Criteria: Pass if all workloads have a completion record with all four required fields populated. Fail if any workload is absent or any field is missing.

---

**TC-R1-02: Backup failure alert within 15 minutes**

Requirement Reference: 1.4
Type: Integration

Preconditions:
- A test workload is registered in the backup inventory.
- The workload's storage is deliberately made inaccessible (e.g., EBS volume detached) to cause backup failure.

Test Steps:
1. Trigger a Rubrik backup job for the test workload.
2. Record the exact timestamp of the backup failure event from Rubrik logs.
3. Monitor the alerting channel (SNS/email) for an alert.

Expected Results:
- An alert is received within 15 minutes of the failure event timestamp.
- The alert includes the workload identifier and a failure reason.
- The most recent successful backup for that workload is not altered or deleted.

Pass/Fail Criteria: Pass if alert arrives within 15 minutes and contains the required fields, and the prior successful backup remains intact. Fail otherwise.

---

**TC-R1-03: Missing workload from backup triggers alert**

Requirement Reference: 1.7
Type: Integration

Preconditions:
- Five workloads are registered in the backup inventory.
- One workload is intentionally excluded from the backup job scope (simulating an inventory drift).

Test Steps:
1. Run the backup job with the excluded workload.
2. Wait for job completion.
3. Monitor the alerting channel.

Expected Results:
- An alert is generated within 15 minutes of job completion.
- The alert identifies the omitted workload by its workload identifier.
- No alert is generated for workloads that were successfully backed up.

Pass/Fail Criteria: Pass if the alert is timely and precisely identifies the omitted workload. Fail if no alert is generated or if the alert misidentifies the workload.

---

**TC-R1-04: Backup data encrypted with source CMK**

Requirement Reference: 1.2
Type: Integration

Preconditions:
- A KMS CMK is active in the Production Simulator Account.
- A test workload backup has been completed.

Test Steps:
1. Retrieve the backup completion record for the test workload.
2. Extract the `kmsKeyArn` field from the record.
3. Verify the key ARN belongs to the Production Simulator Account (not the IRE account).

Expected Results:
- The `kmsKeyArn` field is populated and matches the Production Simulator Account KMS CMK ARN.
- No backup object is stored without a CMK association.

Pass/Fail Criteria: Pass if `kmsKeyArn` is present and matches the correct Production CMK. Fail if the field is absent, null, or references a different account's key.

---

**TC-R1-05: Backup retention policy enforced**

Requirement Reference: 1.5
Type: Integration

Preconditions:
- A backup policy is configured with a 30-day minimum and 365-day maximum retention.
- A test backup completed 31 days ago and a test backup completed 366 days ago exist.

Test Steps:
1. Query the Rubrik backup catalogue for the 31-day-old backup. Verify it exists.
2. Query for the 366-day-old backup. Verify it has been deleted per the retention policy.
3. Attempt to manually delete the 31-day-old backup. Verify this fails (below minimum retention).

Expected Results:
- 31-day backup is present (within retention window).
- 366-day backup is absent (expired past maximum).
- Manual deletion below minimum retention is rejected.

Pass/Fail Criteria: Pass if retention boundaries are correctly enforced in both directions. Fail if any backup outside the retention window is retained or any backup within it is prematurely deleted.

---

**TC-R1-06: Backup data in transit uses TLS 1.2 or higher**

Requirement Reference: 1.6
Type: Security

Preconditions:
- Network capture tooling configured on the path between Rubrik and the Production Simulator Account.

Test Steps:
1. Initiate a backup transfer.
2. Capture the TLS handshake on the network segment between Rubrik and the source workload.
3. Inspect the negotiated TLS version.

Expected Results:
- TLS version negotiated is 1.2 or higher on all Rubrik transfer channels.
- No fallback to TLS 1.1 or lower is observed.

Pass/Fail Criteria: Pass if all observed connections use TLS 1.2+. Fail if any connection uses a lower protocol version.


---

### Requirement 2: IRE Account Isolation and Placement

---

**TC-R2-01: SCP denies direct internet egress from IRE**

Requirement Reference: 2.2
Type: Security

Preconditions:
- The IRE Test Account is placed in the test OU with the production SCP set applied.
- An IAM role in the IRE Test Account has permissions to attempt `ec2:CreateInternetGateway`.

Test Steps:
1. Assume the IAM role in the IRE Test Account.
2. Attempt to create an Internet Gateway via the AWS CLI: `aws ec2 create-internet-gateway`.
3. Attempt to attach an Internet Gateway via `aws ec2 attach-internet-gateway`.

Expected Results:
- Both API calls return an `AccessDenied` error.
- The denial is attributed to the SCP (the error message references the applicable SCP).

Pass/Fail Criteria: Pass if both calls are denied with `AccessDenied`. Fail if either call succeeds.

---

**TC-R2-02: SCP denies VPC peering to Production**

Requirement Reference: 2.2
Type: Security

Preconditions:
- SCPs are applied at the test OU level.
- An IAM role in the IRE Test Account exists.

Test Steps:
1. Attempt to create a VPC peering connection from the IRE Test VPC to the Production Simulator VPC: `aws ec2 create-vpc-peering-connection`.

Expected Results:
- The call returns `AccessDenied`.
- No peering connection object is created.

Pass/Fail Criteria: Pass if the call is denied. Fail if a peering connection is created.

---

**TC-R2-03: Non-allowlisted traffic is denied and dropped at PA Firewall**

Requirement Reference: 2.4
Type: Security

Preconditions:
- A test EC2 instance is running in the IRE Test Restore Subnet.
- A test host outside the approved scanner/ISO CIDRs is available (simulating an arbitrary source).

Test Steps:
1. From the arbitrary source host, attempt to establish a TCP connection to the IRE test instance on port 22 (SSH).
2. Inspect PA Firewall deny logs.

Expected Results:
- The connection attempt is dropped (no TCP RST or response from the IRE instance).
- A deny log entry appears in the PA Firewall flow log within 60 seconds, recording source IP, destination IP, port, protocol, and `DENY` action.

Pass/Fail Criteria: Pass if connection is dropped and a deny log entry is generated. Fail if connection succeeds or no log entry is produced.

---

**TC-R2-04: Steady-state has no route table entries targeting Production subnets**

Requirement Reference: 2.6
Type: Configuration

Preconditions:
- No active recovery operation is in progress in the IRE Test Account.

Test Steps:
1. Retrieve all route tables associated with IRE VPC subnets via `aws ec2 describe-route-tables`.
2. Inspect each route table for any route with a destination CIDR matching any Production Simulator Account subnet range.

Expected Results:
- No route table entry targets any Production subnet CIDR.
- The only non-local routes point to the TGW attachment (directing traffic to the PA Firewall).

Pass/Fail Criteria: Pass if no Production-destined routes are present. Fail if any Production route is found.

---

**TC-R2-05: IRE Account resides in the correct OU**

Requirement Reference: 2.1
Type: Configuration

Preconditions:
- AWS Organizations management account is accessible.

Test Steps:
1. Call `aws organizations list-parents --child-id <IRE-account-id>`.
2. Verify the parent OU ID matches the designated OU containing the Production and DR accounts.

Expected Results:
- The IRE Test Account's parent OU is identical to the OU containing the Production Simulator and DR Test Accounts.

Pass/Fail Criteria: Pass if the OU IDs match. Fail if they differ.


---

### Requirement 3: Break-Glass Access Control

---

**TC-R3-01: Break-glass login with MFA succeeds**

Requirement Reference: 3.1, 3.4
Type: Integration

Preconditions:
- Break-glass user `bg-recovery-engineer-01` is provisioned in the IRE Test Account.
- An MFA device is registered and active for this user.
- The test operator's IP address is within the approved IP range.

Test Steps:
1. Initiate AWS Console login for `bg-recovery-engineer-01` from an approved IP address.
2. Enter the correct MFA token when prompted.
3. Verify the session is established and the console is accessible.

Expected Results:
- Login succeeds and an active console session is established.
- A CloudTrail event `ConsoleLogin` is recorded with the correct IAM principal and source IP.

Pass/Fail Criteria: Pass if login succeeds with MFA and a CloudTrail event is generated. Fail if login is denied or no CloudTrail event is produced.

---

**TC-R3-02: Break-glass login without MFA is denied**

Requirement Reference: 3.4
Type: Security

Preconditions:
- Break-glass user `bg-recovery-engineer-01` is provisioned.
- The MFA enforcement IAM policy is attached to the `bg-recovery-operators` group.

Test Steps:
1. Attempt to perform an AWS API action (`aws s3 ls`) using long-term credentials for `bg-recovery-engineer-01` without providing an MFA session token.

Expected Results:
- The API call returns `AccessDenied` due to the `BoolIfExists: aws:MultiFactorAuthPresent == false` condition.

Pass/Fail Criteria: Pass if the action is denied without MFA. Fail if the action succeeds without MFA.

---

**TC-R3-03: Login from non-approved IP is denied and alerts**

Requirement Reference: 3.5
Type: Security

Preconditions:
- Break-glass user configured with approved IP ranges.
- A test host with an IP outside the approved ranges is available.

Test Steps:
1. From the non-approved IP host, attempt to authenticate as `bg-recovery-engineer-01`.
2. Monitor the security alerting channel for an alert.

Expected Results:
- Authentication is denied.
- A security alert is generated and received on the alerting channel.
- A CloudTrail event captures the denied attempt with the source IP.

Pass/Fail Criteria: Pass if authentication is denied, an alert is generated, and CloudTrail records the event. Fail if any of the three conditions is not met.

---

**TC-R3-04: CloudTrail event generated on break-glass authentication**

Requirement Reference: 3.3
Type: Integration

Preconditions:
- A successful break-glass login has just been performed (see TC-R3-01).

Test Steps:
1. Query CloudTrail for `ConsoleLogin` events in the last 5 minutes.
2. Inspect the event for: IAM principal ARN, source IP address, action (`ConsoleLogin`), and timestamp.

Expected Results:
- A `ConsoleLogin` CloudTrail event exists.
- The event contains the expected IAM principal ARN, the correct source IP, and a timestamp matching the login time.

Pass/Fail Criteria: Pass if all four fields are present and correct. Fail if the event is absent or any field is missing.

---

**TC-R3-05: Credential rotation triggered within 1 hour of session end**

Requirement Reference: 3.8
Type: Integration

Preconditions:
- A break-glass session was established and has just ended (the user logged out or the session expired).
- The EventBridge rotation trigger rule is active.

Test Steps:
1. Record the session end timestamp from CloudTrail.
2. Monitor Secrets Manager for a rotation event on the break-glass secret.

Expected Results:
- A Secrets Manager rotation is initiated within 60 minutes of the session end timestamp.
- The new secret version is stored and encrypted with the IRE break-glass CMK.

Pass/Fail Criteria: Pass if rotation is initiated within 60 minutes. Fail if rotation does not occur within 70 minutes (10% tolerance).

---

**TC-R3-06: Scheduled credential rotation at 90-day boundary**

Requirement Reference: 3.7
Type: Integration

Preconditions:
- The Secrets Manager rotation schedule is configured for 90 days.
- A test break-glass secret has a `LastRotatedDate` set to 90 days in the past (simulated via Secrets Manager test rotation trigger).

Test Steps:
1. Trigger the scheduled rotation Lambda manually (simulating the 90-day schedule firing).
2. Verify a new secret version is created.
3. Verify the new `LastRotatedDate` is within 60 seconds of the trigger time.

Expected Results:
- A new secret version is created.
- The `LastRotatedDate` is updated to the current time.
- The rotation interval does not exceed 90 days.

Pass/Fail Criteria: Pass if a new version is created and the rotation interval is within the 90-day limit. Fail otherwise.

---

**TC-R3-07: MFA failure increments failed attempt counter**

Requirement Reference: 3.6
Type: Integration

Preconditions:
- Break-glass user is provisioned.
- Test MFA device is configured to produce an incorrect token on demand.

Test Steps:
1. Attempt to authenticate with an incorrect MFA token.
2. Check the failed attempt counter in the account's tracking mechanism.
3. Verify a CloudTrail event is generated recording the failure.

Expected Results:
- Authentication is denied.
- Failed attempt counter is incremented by 1.
- CloudTrail event contains IAM principal, source IP, and failure timestamp.

Pass/Fail Criteria: Pass if all three conditions are met. Fail if authentication is permitted or counter is not incremented.


---

### Requirement 4: Restore Workloads into the IRE

---

**TC-R4-01: Restore initiates within 5 minutes of recovery event declaration**

Requirement Reference: 4.1
Type: Integration

Preconditions:
- A valid backup of `test-ec2-linux-01` exists in the Rubrik test instance.
- The IRE Scanning Orchestrator is running.

Test Steps:
1. Record the exact timestamp when a recovery event is declared via the Orchestrator API.
2. Monitor the Workflow Registry (DynamoDB) for the pipeline state to transition to `RESTORE_PENDING`.
3. Check the Rubrik restore job start timestamp.

Expected Results:
- The pipeline state transitions to `RESTORE_PENDING` within 5 minutes of the event declaration timestamp.
- A Rubrik restore job is initiated with the correct backup ID and target IRE account.

Pass/Fail Criteria: Pass if the restore job is initiated within 5 minutes (up to 5 minutes 30 seconds with 10% tolerance). Fail otherwise.

---

**TC-R4-02: Source CMK decryption and IRE CMK re-encryption verified**

Requirement Reference: 4.2
Type: Integration

Preconditions:
- A test workload backup exists, encrypted with the Production Simulator CMK.
- IRE EBS CMK and IRE RDS CMK are active.

Test Steps:
1. Initiate a restore of `test-ec2-linux-01` into the IRE.
2. After restore completes, retrieve the EBS volume descriptor for the restored volume.
3. Check the `KmsKeyId` field on the volume.
4. Verify the key ARN belongs to the IRE Test Account.

Expected Results:
- The restored EBS volume's `KmsKeyId` matches the IRE EBS CMK ARN.
- The Production Simulator CMK ARN does not appear on any restored resource.

Pass/Fail Criteria: Pass if the IRE CMK ARN is present and the Production CMK ARN is absent. Fail otherwise.

---

**TC-R4-03: Restore retries up to 3 times before halting**

Requirement Reference: 4.4, 4.7
Type: Integration

Preconditions:
- Rubrik test instance is configured to simulate restore failure (backup ID flagged as unrestorable).

Test Steps:
1. Declare a recovery event for the flagged backup.
2. Monitor the Workflow Registry for retry events.
3. Record the timestamps of each retry and the final halt event.

Expected Results:
- Exactly 3 restore attempts are made (initial + 2 retries).
- Each retry occurs approximately 2 minutes after the preceding failure.
- After the 3rd failure, the pipeline state transitions to `FAILED`.
- A failure notification is dispatched within 15 minutes of the 3rd failure.

Pass/Fail Criteria: Pass if exactly 3 attempts are made with 2-minute gaps, pipeline halts after the 3rd, and a notification is dispatched within 15 minutes. Fail if any condition differs.

---

**TC-R4-04: Outbound connections from partial restore blocked until scan job registered**

Requirement Reference: 4.6
Type: Integration

Preconditions:
- A restore is in progress (the EC2 instance has been launched but the scan job has not yet been registered).

Test Steps:
1. During the restore (before `RESTORE_COMPLETE` state), attempt an outbound TCP connection from the partially restored instance to a test external host.
2. Register the scan job (allow restore to complete normally).
3. Attempt the same outbound connection again after scan job registration.

Expected Results:
- The outbound connection during partial restore is blocked (no TCP response from the target).
- After the scan job is registered, outbound connections to scanner endpoints are permitted (per security group rules).

Pass/Fail Criteria: Pass if outbound is blocked pre-registration and allowed to scanner endpoints post-registration. Fail if outbound is permitted before scan job registration.

---

**TC-R4-05: Failure notification within 15 minutes after 3 failed attempts**

Requirement Reference: 4.4
Type: Integration

Preconditions:
- Same setup as TC-R4-03.

Test Steps:
1. Record the timestamp of the 3rd restore failure.
2. Monitor the recovery team notification channel (SNS/email).

Expected Results:
- A notification is received within 15 minutes of the 3rd failure timestamp.
- The notification contains the workload identifier and failure summary.

Pass/Fail Criteria: Pass if notification arrives within 16 minutes 30 seconds (15 min + 10% tolerance). Fail otherwise.

---

**TC-R4-06: Restore records completion event with required fields**

Requirement Reference: 4.3
Type: Integration

Preconditions:
- A successful restore of `test-ec2-linux-01` has just completed.

Test Steps:
1. Query the Workflow Registry (DynamoDB) for the `RestoreCompletionRecord` for the test workflow.
2. Inspect the record for all required fields.

Expected Results:
- Record contains: `workloadId`, `backupTimestamp`, `restoreDuration`, `targetResourceIds` (list of IRE EC2/EBS resource IDs).
- `kmsKeyArn` field is present and matches the IRE EBS CMK.

Pass/Fail Criteria: Pass if all fields are present and populated. Fail if any field is absent or null.


---

### Requirement 5: EC2/EBS Workload Recovery

---

**TC-R5-01: EBS snapshot created from Rubrik backup and attached to pre-registered EC2 launch template**

Requirement Reference: 5.1
Type: Integration

Preconditions:
- `test-ec2-linux-01` has a pre-registered EC2 launch template in the IRE Test Account.
- A valid Rubrik backup exists for this workload.

Test Steps:
1. Initiate an EC2 workload restore for `test-ec2-linux-01`.
2. After restore completes, inspect the launched EC2 instance to confirm it used the pre-registered launch template.
3. Verify an EBS snapshot was created in the IRE account from the Rubrik backup.

Expected Results:
- The EC2 instance was launched from the correct pre-registered launch template.
- An EBS snapshot exists in the IRE account corresponding to the restore.
- The snapshot's source is the Rubrik backup, not a direct copy from the Production account.

Pass/Fail Criteria: Pass if the launch template reference matches and the snapshot exists. Fail otherwise.

---

**TC-R5-02: Restored EBS volumes encrypted with IRE EBS CMK**

Requirement Reference: 5.2
Type: Integration

Preconditions:
- A restore of `test-ec2-linux-01` has completed.

Test Steps:
1. Call `aws ec2 describe-volumes --filters Name=attachment.instance-id,Values=<restored-instance-id>`.
2. Check the `KmsKeyId` field on each attached volume.

Expected Results:
- Every EBS volume attached to the restored instance has `KmsKeyId` set to the IRE EBS CMK ARN.
- No volume uses the Production Simulator Account CMK.

Pass/Fail Criteria: Pass if all volumes use the IRE EBS CMK. Fail if any volume uses a different key.

---

**TC-R5-03: IRE-Restore-SG applied, blocking all inbound except scanner CIDRs**

Requirement Reference: 5.3
Type: Integration

Preconditions:
- A restore has completed and the EC2 instance is in running state.

Test Steps:
1. Retrieve the security group attached to the restored instance.
2. Verify the inbound rules permit only Tanium (TCP 17472, 17486), CrowdStrike (TCP 443), and Qualys (TCP 443, ICMP) from their respective CIDR ranges.
3. Attempt an inbound connection from a non-scanner IP on port 22 (SSH). Verify it is blocked.

Expected Results:
- The security group is `IRE-Restore-SG`.
- Only scanner CIDRs are in the inbound allow list.
- Connection from a non-scanner IP is blocked.

Pass/Fail Criteria: Pass if security group is correct and non-scanner traffic is blocked. Fail if any additional inbound rule exists or non-scanner traffic is permitted.

---

**TC-R5-04: Restored instance metadata matches backup metadata**

Requirement Reference: 5.4
Type: Integration

Preconditions:
- The source backup metadata for `test-ec2-linux-01` contains known tags, IAM instance profile reference, and volume attachment configuration.

Test Steps:
1. Retrieve the restored instance via `aws ec2 describe-instances`.
2. Compare the instance tags, IAM instance profile ARN, and volume attachment configuration against the values recorded in the backup metadata.

Expected Results:
- All tags are identical to those in the backup metadata.
- The IAM instance profile reference matches the backup metadata value.
- Volume attachment configuration (device mappings) matches the backup metadata.

Pass/Fail Criteria: Pass if all three attributes match exactly. Fail if any attribute differs.

---

**TC-R5-05: EC2 restore failure raises alert within 5 minutes**

Requirement Reference: 5.5
Type: Integration

Preconditions:
- A backup is configured to fail on restore (e.g., snapshot is intentionally corrupt).

Test Steps:
1. Initiate the EC2 restore for the corrupt backup.
2. Record the failure timestamp from CloudWatch/Workflow Registry.
3. Monitor the recovery team alert channel.

Expected Results:
- An alert is received within 5 minutes of the failure event.
- The alert contains the failure details and workload identifier.

Pass/Fail Criteria: Pass if alert arrives within 5 minutes 30 seconds. Fail otherwise.

---

**TC-R5-06: Successful EC2 restore writes audit log entry**

Requirement Reference: 5.6
Type: Integration

Preconditions:
- A restore of `test-ec2-linux-01` has completed successfully.

Test Steps:
1. Query the centralised S3 audit log bucket for a restore completion record matching the test workload's workflow ID.
2. Inspect the record for: restored instance identifier, restore timestamp, and source backup identifier.

Expected Results:
- A restore completion record exists in the centralised audit log.
- The record contains all three required fields.

Pass/Fail Criteria: Pass if the record exists with all required fields. Fail if the record is absent or incomplete.


---

### Requirement 6: RDS Workload Recovery

---

**TC-R6-01: RDS snapshot restore via native RDS API**

Requirement Reference: 6.1
Type: Integration

Preconditions:
- `test-rds-mysql-01` has a valid Rubrik backup that includes an RDS snapshot.

Test Steps:
1. Initiate a recovery event for `test-rds-mysql-01`.
2. Monitor Rubrik and AWS CloudTrail for the `RestoreDBInstanceFromDBSnapshot` API call.

Expected Results:
- A `RestoreDBInstanceFromDBSnapshot` event appears in CloudTrail.
- The call targets the IRE Test Account, not the Production Simulator Account.
- The restored instance appears in the IRE Test Account's RDS console.

Pass/Fail Criteria: Pass if the native RDS API is used and the instance is created in the IRE account. Fail if a different restore method is used.

---

**TC-R6-02: IRE RDS CMK applied before instance reaches available state**

Requirement Reference: 6.2
Type: Integration

Preconditions:
- A restore of `test-rds-mysql-01` is in progress.

Test Steps:
1. Poll the RDS instance status every 10 seconds from restore initiation.
2. When the instance first transitions from `creating` to `available`, immediately query the `StorageEncrypted` and `KmsKeyId` fields.

Expected Results:
- `StorageEncrypted == true` and `KmsKeyId` matches the IRE RDS CMK ARN at the moment the instance first reaches `available` state.
- The Production Simulator CMK ARN does not appear.

Pass/Fail Criteria: Pass if CMK is applied before `available` state. Fail if the instance reaches `available` with a different or absent CMK.

---

**TC-R6-03: Public accessibility disabled before instance available**

Requirement Reference: 6.4
Type: Integration

Preconditions:
- Same as TC-R6-02.

Test Steps:
1. When the RDS instance first reaches `available` state, call `aws rds describe-db-instances` and check `PubliclyAccessible`.

Expected Results:
- `PubliclyAccessible == false` at the moment the instance first reaches `available` state.

Pass/Fail Criteria: Pass if `PubliclyAccessible` is false. Fail if it is ever true.

---

**TC-R6-04: RDS security group permits only scanner CIDR ranges on DB port**

Requirement Reference: 6.3
Type: Integration

Preconditions:
- A restored RDS instance is running in the IRE Test Account.

Test Steps:
1. Retrieve the security group attached to the restored RDS instance.
2. Inspect inbound rules for the MySQL port (3306) or PostgreSQL port (5432).
3. Attempt a connection to the DB port from a non-scanner test IP.

Expected Results:
- Inbound rules on the DB port list only the approved scanner CIDR ranges.
- Connection from a non-scanner IP is refused or timed out.

Pass/Fail Criteria: Pass if only scanner CIDRs are in the inbound rules and non-scanner connections are blocked. Fail otherwise.

---

**TC-R6-05: Engine version incompatibility alert within 5 minutes**

Requirement Reference: 6.5
Type: Integration

Preconditions:
- A backup snapshot is labelled with engine version `MySQL 8.4` but the IRE Test Account's RDS configuration only supports up to `MySQL 8.0`, triggering an incompatibility.

Test Steps:
1. Initiate a restore for the incompatible snapshot.
2. Record the failure timestamp from the pipeline log.
3. Monitor the recovery team alert channel.

Expected Results:
- An alert is received within 5 minutes of failure detection.
- The alert contains: instance identifier, source snapshot identifier, detected engine version, and incompatibility reason.

Pass/Fail Criteria: Pass if the alert arrives within 5 minutes 30 seconds with all four required fields. Fail otherwise.

---

**TC-R6-06: Parameter group and option group names recorded in restore metadata**

Requirement Reference: 6.6
Type: Integration

Preconditions:
- `test-rds-mysql-01` was backed up when using a custom parameter group `mysql-custom-params` and option group `mysql-custom-options`.

Test Steps:
1. After restore, query the restore metadata record via the recovery job identifier.
2. Check for `parameterGroupName` and `optionGroupName` fields.

Expected Results:
- `parameterGroupName == "mysql-custom-params"`.
- `optionGroupName == "mysql-custom-options"`.
- Both values are retrievable via the recovery job identifier.

Pass/Fail Criteria: Pass if both fields are present and match the source values. Fail if either field is missing or incorrect.


---

### Requirement 7: EKS Workload Recovery

---

**TC-R7-01: PV snapshots and manifests restored within 60 minutes**

Requirement Reference: 7.1
Type: Integration

Preconditions:
- `test-eks-cluster-01` has a valid Rubrik backup including PV snapshots and Kubernetes manifests.
- The IRE Test Account has a pre-provisioned EKS cluster.

Test Steps:
1. Declare a recovery event for `test-eks-cluster-01`.
2. Record the restore initiation timestamp from the Workflow Registry.
3. Monitor the EKS cluster for pod and PV availability.
4. Record the timestamp when all pods reach a `Running` state and all PVs are bound.

Expected Results:
- All PVs are bound and all pods are running within 60 minutes of restore initiation.

Pass/Fail Criteria: Pass if the 60-minute threshold is met. Fail if pods are not running within 66 minutes (10% tolerance).

---

**TC-R7-02: PV encrypted with IRE EKS CMK before pods reach Running state**

Requirement Reference: 7.2
Type: Integration

Preconditions:
- An EKS restore is in progress.

Test Steps:
1. At the moment the first pod transitions to `Running`, call `aws ec2 describe-volumes` for all PVs attached to EKS nodes.
2. Check `KmsKeyId` on each PV-backing EBS volume.

Expected Results:
- All PV-backing volumes have `KmsKeyId` equal to the IRE EKS PV CMK ARN when the first pod reaches `Running`.
- No pod reaches `Running` state with an unencrypted or Production-CMK-encrypted PV.

Pass/Fail Criteria: Pass if all volumes use the IRE EKS PV CMK at pod `Running` time. Fail otherwise.

---

**TC-R7-03: Network policies allow only scanner endpoints inbound**

Requirement Reference: 7.3
Type: Integration

Preconditions:
- An EKS workload has been restored.

Test Steps:
1. Retrieve all Kubernetes NetworkPolicy objects applied to the restored namespace.
2. Verify they contain only inbound allow rules for scanner endpoint IP ranges.
3. Attempt a connection to a pod from a non-scanner test IP.

Expected Results:
- Network policies list only scanner endpoint CIDRs in ingress rules.
- Connection from a non-scanner IP is denied.

Pass/Fail Criteria: Pass if policies are correct and non-scanner traffic is blocked. Fail otherwise.

---

**TC-R7-04: Production registry unreachable within 30 seconds triggers image substitution**

Requirement Reference: 7.5
Type: Integration

Preconditions:
- A Kubernetes manifest references a container image in the Production Simulator registry.
- The Production Simulator registry is unreachable from the IRE network (simulated by blocking the DNS name or IP).

Test Steps:
1. Initiate an EKS restore with the manifest referencing the Production registry image.
2. After 30 seconds without a successful pull, check whether the IRE internal registry is substituted.
3. Verify the substitution is logged with the original and substitute registry references.

Expected Results:
- The image pull source is substituted with the IRE-approved internal registry after 30 seconds.
- A log entry records the original registry reference and the substitute registry reference.

Pass/Fail Criteria: Pass if substitution occurs after 30 seconds (within 33 seconds tolerance) and the log entry contains both references. Fail otherwise.

---

**TC-R7-05: External LoadBalancer services disabled on restore**

Requirement Reference: 7.6
Type: Integration

Preconditions:
- The backed-up EKS workload had external LoadBalancer services defined.

Test Steps:
1. After restore completes, list all services in the restored namespace.
2. Check the type and status of any LoadBalancer-type services.

Expected Results:
- All external LoadBalancer services are disabled (type changed to `ClusterIP` or service has no external IP assigned).
- No external IP is allocated until post-promotion.

Pass/Fail Criteria: Pass if no external LoadBalancer IP is present on the restored workload. Fail if any external IP is allocated.

---

**TC-R7-06: PV snapshot failure triggers rollback and error report**

Requirement Reference: 7.7
Type: Integration

Preconditions:
- One PV snapshot in the test backup is intentionally corrupted to cause restore failure.

Test Steps:
1. Initiate the EKS restore.
2. Allow the PV restore to fail.
3. Check the EKS cluster for any partially restored resources.
4. Check the Workflow Registry for a failure report.

Expected Results:
- All partially restored resources (pods, PVs, services) are removed from the cluster.
- A failure report in the Workflow Registry specifies which PV failed.
- The EKS cluster is in its pre-restore state.

Pass/Fail Criteria: Pass if all partial resources are cleaned up and a failure report is generated. Fail if any partial resource remains in the cluster.


---

### Requirement 8: Lambda Workload Recovery

---

**TC-R8-01: Function deployed with all attributes**

Requirement Reference: 8.1
Type: Integration

Preconditions:
- `test-lambda-01` has a Rubrik backup.
- The backup captures: runtime `python3.11`, handler `index.handler`, memory `512MB`, timeout `30s`, one layer, and three environment variables.

Test Steps:
1. Initiate a Lambda restore for `test-lambda-01`.
2. After deployment, call `aws lambda get-function-configuration --function-name <restored-name>`.
3. Compare each attribute against the backup record.

Expected Results:
- `Runtime`, `Handler`, `MemorySize`, `Timeout`, `Layers`, and `Environment.Variables` all match the backup record values.

Pass/Fail Criteria: Pass if all six attributes match. Fail if any attribute differs.

---

**TC-R8-02: Environment variables encrypted with IRE Lambda CMK**

Requirement Reference: 8.2
Type: Integration

Preconditions:
- `test-lambda-01` restore has completed.

Test Steps:
1. Call `aws lambda get-function-configuration` and check the `KMSKeyArn` field.

Expected Results:
- `KMSKeyArn` matches the IRE Lambda CMK ARN.
- No environment variable is stored in plaintext.

Pass/Fail Criteria: Pass if `KMSKeyArn` equals the IRE Lambda CMK ARN. Fail otherwise.

---

**TC-R8-03: Production ARN in env var disables function and flags for review**

Requirement Reference: 8.6
Type: Integration

Preconditions:
- `test-lambda-prod-arn` has a backup. One environment variable contains a Production account ARN pattern.

Test Steps:
1. Initiate a restore of `test-lambda-prod-arn`.
2. After deployment, check the function's `State` field.
3. Check the restore metadata for flagged variable names.

Expected Results:
- Function `State` is `Disabled`.
- The restore metadata records the name(s) of environment variables containing Production ARN patterns.
- The function remains disabled until the flagged variables are updated or acknowledged.

Pass/Fail Criteria: Pass if the function is disabled and the flagged variable names are recorded. Fail if the function is enabled or the flagged names are not recorded.

---

**TC-R8-04: IRE-scoped execution role assigned as active role**

Requirement Reference: 8.5
Type: Integration

Preconditions:
- `test-lambda-01` restore has completed.
- The source Lambda had an execution role `arn:aws:iam::999999999999:role/prod-lambda-role`.

Test Steps:
1. Call `aws lambda get-function-configuration` and retrieve the `Role` field.
2. Verify the role ARN belongs to the IRE Test Account.
3. Verify the role's policy is scoped to IRE account resources only.

Expected Results:
- The active execution role ARN belongs to the IRE Test Account (account ID `111111111111`).
- The role policy references only IRE account resource ARNs.

Pass/Fail Criteria: Pass if the active role is IRE-scoped and the Production role is not applied. Fail otherwise.

---

**TC-R8-05: Original execution role ARN recorded in metadata only**

Requirement Reference: 8.4
Type: Integration

Preconditions:
- Same as TC-R8-04.

Test Steps:
1. Query the restore metadata record for `test-lambda-01`.
2. Verify the `originalExecutionRoleArn` field is present and contains the Production execution role ARN.
3. Verify this ARN does not appear in the `Role` field of the deployed function.

Expected Results:
- `originalExecutionRoleArn` in metadata equals `arn:aws:iam::999999999999:role/prod-lambda-role`.
- The function's active `Role` field does not equal the Production role ARN.

Pass/Fail Criteria: Pass if both conditions are met. Fail otherwise.

---

**TC-R8-06: Lambda deploy failure leaves resources disabled and logs failure**

Requirement Reference: 8.7
Type: Integration

Preconditions:
- A Lambda backup is configured to fail mid-deployment (e.g., the deployment package is intentionally corrupted).

Test Steps:
1. Initiate the Lambda restore for the corrupted backup.
2. Allow the deployment to fail.
3. Check AWS Lambda for any partially deployed function state.
4. Check the Workflow Registry for a failure record.

Expected Results:
- Any partially deployed function resources have state `Disabled`.
- The failure record contains the function name and failure reason.
- A recovery engineer is notified of the failure.

Pass/Fail Criteria: Pass if partial resources are disabled and the failure is logged with name and reason. Fail otherwise.


---

### Requirement 9: Security Scanning of Restored Workloads

---

**TC-R9-01: Scanning sequence is Tanium → CrowdStrike → Qualys**

Requirement Reference: 9.1
Type: Integration

Preconditions:
- `test-ec2-linux-01` has been restored and is in `RESTORE_COMPLETE` state.
- All scanner sandboxes are configured to return clean results after a simulated scan.

Test Steps:
1. Allow the Scanning Orchestrator to proceed automatically.
2. Record the API call timestamps from the orchestrator logs for each scanner invocation.
3. Record the result receipt timestamps for each scanner.

Expected Results:
- Tanium API is called first.
- CrowdStrike API is called only after Tanium returns a result.
- Qualys API is called only after CrowdStrike returns a result.

Pass/Fail Criteria: Pass if all three invocations occur in the correct order with no overlap. Fail if any scanner is invoked before the preceding scanner has returned a result.

---

**TC-R9-02: Tanium scan completes within 2 hours with clean result**

Requirement Reference: 9.2
Type: Integration

Preconditions:
- `test-ec2-linux-01` is restored and healthy (no malware).
- Tanium sandbox is configured to perform a scan and return a clean result.

Test Steps:
1. Note Tanium scan initiation timestamp from the orchestrator log.
2. Wait for Tanium to return a scan result.
3. Record the result receipt timestamp.
4. Check the result status.

Expected Results:
- Tanium returns a result within 2 hours of scan initiation.
- Result status is `CLEAN` (zero threats detected).

Pass/Fail Criteria: Pass if result arrives within 2 hours and status is `CLEAN`. Fail otherwise.

---

**TC-R9-03: CrowdStrike scan completes within 2 hours with clean result**

Requirement Reference: 9.3
Type: Integration

Preconditions:
- Same as TC-R9-02, but after Tanium has returned clean.

Test Steps:
1. Note CrowdStrike scan initiation timestamp.
2. Wait for CrowdStrike to return a result.

Expected Results:
- CrowdStrike returns a result within 2 hours.
- Result status is `CLEAN` (zero detections of any severity).

Pass/Fail Criteria: Pass if result arrives within 2 hours and status is `CLEAN`. Fail otherwise.

---

**TC-R9-04: Qualys scan completes within 4 hours, zero CVSS 7.0+ findings**

Requirement Reference: 9.4
Type: Integration

Preconditions:
- `test-ec2-linux-01` is restored and has no CVSS 7.0+ vulnerabilities.

Test Steps:
1. Note Qualys scan initiation timestamp.
2. Wait for Qualys to return a vulnerability report.
3. Inspect the report for any findings with CVSS v3 score ≥ 7.0.

Expected Results:
- Qualys returns a report within 4 hours.
- Zero findings rated CVSS v3 ≥ 7.0.

Pass/Fail Criteria: Pass if report arrives within 4 hours and contains zero CVSS 7.0+ findings. Fail otherwise.

---

**TC-R9-05: CVSS 7.0+ finding quarantines workload and notifies within 15 minutes**

Requirement Reference: 9.5
Type: Integration

Preconditions:
- `test-ec2-infected-01` is restored (contains a simulated CVSS 8.0 vulnerability visible to Qualys).

Test Steps:
1. Allow scanning to proceed to Qualys.
2. When Qualys returns a CVSS 8.0 finding, record the detection timestamp.
3. Check the workload's subnet and security group.
4. Monitor the security operations team notification channel.

Expected Results:
- Workload moves to the Quarantine Subnet (`10.100.10.0/24`).
- Security group is replaced with `IRE-Quarantine-SG` (deny all).
- Pipeline state transitions to `QUARANTINED`.
- Security operations notification received within 15 minutes of detection.

Pass/Fail Criteria: Pass if all four conditions are met within the time limit. Fail if any condition is not met.

---

**TC-R9-06: Scanner timeout treats workload as failed and notifies within 15 minutes**

Requirement Reference: 9.7
Type: Integration

Preconditions:
- Tanium sandbox is configured to not return a result (simulating a timeout).
- The scan timeout is set to 2 hours.

Test Steps:
1. Initiate a restore and allow scanning to begin.
2. Allow 2 hours to elapse without Tanium returning a result.
3. Record the timeout detection timestamp.
4. Monitor the security operations notification channel.

Expected Results:
- Orchestrator detects the timeout and marks the workload as `SCAN_FAILED`.
- Pipeline promotion is halted.
- Security operations notification received within 15 minutes of timeout detection.

Pass/Fail Criteria: Pass if the workload is marked as failed and a notification arrives within 15 minutes. Fail otherwise.

---

**TC-R9-07: Promotion blocked until all 3 scanners return clean**

Requirement Reference: 9.6
Type: Integration

Preconditions:
- A workload has completed Tanium and CrowdStrike scans (both clean) but Qualys has not yet returned a result.

Test Steps:
1. Attempt to manually trigger a promotion from the Orchestrator API while Qualys result is pending.

Expected Results:
- The promotion attempt is rejected with an error indicating Qualys scan is not complete.
- No artefact transfer is initiated.

Pass/Fail Criteria: Pass if promotion is blocked. Fail if any artefact transfer proceeds.

---

**TC-R9-08: Scan audit records stored in immutable audit log**

Requirement Reference: 9.9
Type: Integration

Preconditions:
- A complete scan cycle has been performed for `test-ec2-linux-01`.

Test Steps:
1. Query the centralised S3 audit log bucket for `ScanResult` records matching the workflow ID.
2. Verify records exist for Tanium, CrowdStrike, and Qualys.
3. Attempt to delete one of the scan audit records.

Expected Results:
- Three `ScanResult` records exist (one per scanner), each containing scan initiation time, scanner identity, completion time, and outcome.
- Deletion of the record is rejected by S3 Object Lock.

Pass/Fail Criteria: Pass if records are present, contain required fields, and deletion is blocked. Fail otherwise.


---

### Requirement 10: Workload Cleansing

---

**TC-R10-01: Tanium automated remediation removes threat and logs result**

Requirement Reference: 10.1
Type: Integration

Preconditions:
- `test-ec2-infected-01` is restored with a known EICAR test file detectable by Tanium.
- Tanium sandbox is configured to attempt automated remediation.

Test Steps:
1. Allow Tanium to detect the threat.
2. Allow automated remediation to run.
3. Query the centralised audit log for the remediation record.

Expected Results:
- Tanium reports `remediation_status = success`.
- Audit log record contains: tool name (`TANIUM`), threat identifier, and remediation status.
- The EICAR file is no longer present on the instance.

Pass/Fail Criteria: Pass if remediation succeeds, the threat is removed, and the audit log record contains all three required fields. Fail otherwise.

---

**TC-R10-02: CrowdStrike automated remediation isolates process and logs result**

Requirement Reference: 10.2
Type: Integration

Preconditions:
- `test-ec2-infected-01` is restored with a simulated malicious process detectable by CrowdStrike.

Test Steps:
1. Allow CrowdStrike to detect and isolate the affected process.
2. Allow automated remediation to run.
3. Query the centralised audit log for the remediation record.

Expected Results:
- The affected process is isolated.
- Audit log record contains: tool name (`CROWDSTRIKE`), threat identifier, and remediation status.

Pass/Fail Criteria: Pass if isolation occurs and audit log contains required fields. Fail otherwise.

---

**TC-R10-03: Remediation timeout (30 minutes) triggers quarantine**

Requirement Reference: 10.3
Type: Integration

Preconditions:
- Tanium sandbox is configured to initiate remediation but not complete within 30 minutes (simulated via delayed response).

Test Steps:
1. Allow Tanium to detect a threat and initiate remediation.
2. Allow 30 minutes to elapse without remediation completion.
3. Check the workload state and subnet.

Expected Results:
- After 30 minutes of unresolved remediation, the workload is moved to `QUARANTINED` state.
- The workload is in the Quarantine Subnet with `IRE-Quarantine-SG`.
- An escalation to manual security review is raised.

Pass/Fail Criteria: Pass if quarantine occurs within 33 minutes (10% tolerance) of remediation initiation. Fail otherwise.

---

**TC-R10-04: Post-remediation rescan cycles all three scanners before promotion**

Requirement Reference: 10.4
Type: Integration

Preconditions:
- Tanium has completed successful automated remediation on `test-ec2-infected-01`.

Test Steps:
1. Confirm remediation success.
2. Monitor the Orchestrator for scan job triggers.
3. Record the order and completion of all subsequent scans.

Expected Results:
- A full three-scanner cycle (Tanium → CrowdStrike → Qualys) is triggered after remediation.
- Promotion is only permitted after this post-remediation cycle returns clean from all three scanners.
- The prior partial scan cycle results are not reused.

Pass/Fail Criteria: Pass if a full new scan cycle is triggered and promotion requires clean results from this new cycle. Fail if promotion is permitted based on pre-remediation scan results.

---

**TC-R10-05: Quarantine event audit logged with workload ID, tool, and timestamp**

Requirement Reference: 10.5
Type: Integration

Preconditions:
- A workload has been quarantined (either via TC-R9-05 or TC-R10-03).

Test Steps:
1. Query the centralised audit log for the quarantine event record.
2. Inspect the record fields.

Expected Results:
- Quarantine event record exists.
- Record contains: workload identifier, triggering tool name, and quarantine timestamp.

Pass/Fail Criteria: Pass if all three fields are present and accurate. Fail otherwise.


---

### Requirement 11: Promotion of Validated Workloads to DR Environment

---

**TC-R11-01: Happy path promotion after all scans pass**

Requirement Reference: 11.1, 11.2
Type: End-to-End

Preconditions:
- `test-ec2-linux-01` has completed all three scan cycles with clean results.
- A designated recovery authority is available to provide approval.
- DR Test Account has pre-provisioned infrastructure for this workload.

Test Steps:
1. Observe the Orchestrator publish an approval request to the SNS topic.
2. Recovery authority provides approval via the pre-signed approval URL.
3. Monitor the Promotion Service for artefact transfer.
4. After completion, verify artefact presence in the DR Test Account.

Expected Results:
- Promotion begins after approval is received.
- EBS snapshot is transferred to the DR Test Account using TLS 1.2+.
- Snapshot is re-encrypted with the DR EBS CMK ARN.
- Promotion audit record is written to the centralised log.

Pass/Fail Criteria: Pass if artefact is in the DR account, encrypted with the DR CMK, and the audit record is present. Fail otherwise.

---

**TC-R11-02: Approval gate expires after 4 hours and notifies recovery team**

Requirement Reference: 11.3
Type: Integration

Preconditions:
- A workload has passed all scans and an approval request has been issued.
- The approval request is deliberately not actioned.

Test Steps:
1. Note the approval request timestamp.
2. Allow 4 hours to elapse without any approval or rejection.
3. Monitor the pipeline state and notification channel.

Expected Results:
- Pipeline state transitions to `APPROVAL_EXPIRED` after 4 hours.
- Recovery team receives a notification of the expiry.
- No promotion proceeds.
- The pipeline must be re-initiated from the `RequestApproval` step.

Pass/Fail Criteria: Pass if all four conditions are met. Fail otherwise.

---

**TC-R11-03: Checksum mismatch aborts promotion and rolls back artefacts**

Requirement Reference: 11.7
Type: Integration

Preconditions:
- A promotion is in progress.
- The test environment is configured to corrupt one artefact during transfer (simulated by modifying the transferred snapshot).

Test Steps:
1. Allow the promotion transfer to complete.
2. When the Promotion Service computes the post-transfer checksum, it detects a mismatch.
3. Observe the pipeline response.
4. Check the DR Test Account for any artefacts.

Expected Results:
- Promotion is aborted.
- All transferred artefacts are rolled back (deleted from the DR account).
- Recovery team is notified of the integrity failure.
- No promotion success confirmation is issued.

Pass/Fail Criteria: Pass if promotion is aborted, artefacts are rolled back, and the team is notified. Fail if any artefact remains in the DR account after rollback.

---

**TC-R11-04: Infrastructure type mismatch aborts promotion**

Requirement Reference: 11.6
Type: Integration

Preconditions:
- A Lambda workload has passed all scans.
- The DR Test Account has been provisioned with EC2 infrastructure (wrong type) for this workload's slot.

Test Steps:
1. Initiate the promotion workflow.
2. The Promotion Service validates the DR infrastructure type.

Expected Results:
- Promotion is aborted with a type mismatch error.
- The mismatch reason is logged.
- Recovery team is notified.

Pass/Fail Criteria: Pass if promotion is aborted and the mismatch is logged. Fail if the promotion proceeds to transfer.

---

**TC-R11-05: Promotion audit log entry recorded with approver identity**

Requirement Reference: 11.4
Type: Integration

Preconditions:
- A successful promotion has just completed (see TC-R11-01).

Test Steps:
1. Query the centralised audit log for the `PromotionAuditRecord` matching the workflow ID.
2. Inspect all required fields.

Expected Results:
- Record contains: workload identifier, IRE scan results summary (per scanner), approving authority IAM principal ARN, and promotion timestamp.

Pass/Fail Criteria: Pass if all four fields are present. Fail if any field is absent or null.

---

**TC-R11-06: TLS 1.2+ used for all artefact transfers**

Requirement Reference: 11.2
Type: Security

Preconditions:
- Network capture tooling is active on the cross-account artefact transfer path.

Test Steps:
1. Initiate a promotion transfer.
2. Capture TLS handshake on the transfer channel.
3. Inspect the negotiated TLS version.

Expected Results:
- All transfer channels use TLS 1.2 or higher.

Pass/Fail Criteria: Pass if TLS 1.2+ is confirmed on all channels. Fail if any lower version is observed.

---

**TC-R11-07: DR CMK re-encryption applied on artefact transfer**

Requirement Reference: 11.2
Type: Integration

Preconditions:
- A promotion transfer has completed.

Test Steps:
1. Describe the transferred EBS snapshot in the DR Test Account.
2. Check the `KmsKeyId` field.

Expected Results:
- `KmsKeyId` matches the DR EBS CMK ARN (not the IRE CMK ARN).

Pass/Fail Criteria: Pass if the DR CMK is used. Fail if the IRE CMK or any other key is present.

---

**TC-R11-08: Promotion failure triggers rollback and 15-minute notification**

Requirement Reference: 11.5
Type: Integration

Preconditions:
- A promotion transfer is configured to fail mid-transfer (e.g., cross-account copy API error injected).

Test Steps:
1. Initiate a promotion.
2. Allow the transfer to fail.
3. Check the DR account for any partially transferred artefacts.
4. Monitor the recovery team notification channel.

Expected Results:
- All partially transferred artefacts are deleted from the DR account.
- Failure reason is logged.
- Recovery team notification received within 15 minutes of failure.

Pass/Fail Criteria: Pass if rollback is complete and notification arrives in time. Fail otherwise.


---

### Requirement 12: DR Environment Workload Activation

---

**TC-R12-01: Promoted workload importable into pre-provisioned DR infrastructure**

Requirement Reference: 12.1, 12.2
Type: Integration

Preconditions:
- `test-ec2-linux-01` has been successfully promoted to the DR Test Account.
- DR Test Account has pre-provisioned EC2 launch template and VPC infrastructure.

Test Steps:
1. Trigger DR infrastructure validation via the DR Account's import check API.
2. Verify the workload artefact (EBS snapshot) can be attached to the pre-provisioned launch template.
3. Check the workload activation-ready status.

Expected Results:
- Validation succeeds: the artefact is importable into the target infrastructure.
- Workload status is set to `ACTIVATION_READY`.

Pass/Fail Criteria: Pass if validation succeeds and status is `ACTIVATION_READY`. Fail otherwise.

---

**TC-R12-02: Activation event logged in CloudTrail with workload ID and IAM principal**

Requirement Reference: 12.4
Type: Integration

Preconditions:
- A workload in `ACTIVATION_READY` state is manually activated by a recovery engineer.

Test Steps:
1. Recovery engineer triggers workload activation via the DR Environment API.
2. Query CloudTrail in the DR Test Account for the activation event.
3. Inspect the event fields.

Expected Results:
- A CloudTrail event exists for the activation action.
- The event contains: workload identifier, activation timestamp, and the IAM principal ARN of the recovery engineer.

Pass/Fail Criteria: Pass if all three fields are present in the CloudTrail event. Fail if the event is absent or any field is missing.

---

**TC-R12-03: DR Environment data encrypted with DR CMK**

Requirement Reference: 12.3
Type: Integration

Preconditions:
- A workload has been promoted to the DR Test Account and imported into DR infrastructure.

Test Steps:
1. Describe the DR EBS volumes, RDS instance, EKS PVs, or Lambda function (depending on workload type).
2. Check `KmsKeyId` on each resource.

Expected Results:
- All resources use the DR Test Account's KMS CMK ARN.
- No IRE CMK ARN appears on any DR resource.

Pass/Fail Criteria: Pass if all resources use the DR CMK. Fail otherwise.


---

### Requirement 13: ISO Environment Foundational Services

---

**TC-R13-01: AD authentication from IRE workload succeeds via ISO AD**

Requirement Reference: 13.1, 13.3
Type: Integration

Preconditions:
- A test IRE workload EC2 instance is running.
- An IRE-scoped service account is provisioned in the ISO Test AD (in the IRE service account scope).

Test Steps:
1. From the IRE test instance, attempt an LDAPS bind to the ISO AD server on port 636.
2. Authenticate with the IRE-scoped service account credentials.

Expected Results:
- LDAPS bind succeeds.
- Authentication is completed using an IRE-scoped service account (not a Production service account).

Pass/Fail Criteria: Pass if authentication succeeds with an IRE-scoped account. Fail if authentication uses a Production service account or fails unexpectedly.

---

**TC-R13-02: DNS query for approved zone resolves correctly**

Requirement Reference: 13.4
Type: Integration

Preconditions:
- The IRE VPC DHCP option set points to the ISO DNS server.
- `ire.internal` zone is on the DNS allowlist with a known A record.

Test Steps:
1. From an IRE test instance, run `nslookup testhost.ire.internal`.
2. Observe the response.

Expected Results:
- The query resolves successfully with the correct IP address.
- Response type is `RESOLVED`.

Pass/Fail Criteria: Pass if the name resolves correctly. Fail if the query returns REFUSED, NXDOMAIN, or an incorrect IP.

---

**TC-R13-03: DNS query for non-approved zone returns REFUSED or NXDOMAIN**

Requirement Reference: 13.4
Type: Integration

Preconditions:
- `production.internal` zone is not on the IRE DNS allowlist.

Test Steps:
1. From an IRE test instance, run `nslookup testhost.production.internal`.
2. Observe the response.

Expected Results:
- Response is `REFUSED` or `NXDOMAIN`.
- The query is not silently dropped or forwarded to an external resolver.

Pass/Fail Criteria: Pass if response is REFUSED or NXDOMAIN. Fail if the query resolves or is silently dropped.

---

**TC-R13-04: Log write from IRE to ISO monitoring endpoint succeeds**

Requirement Reference: 13.5
Type: Integration

Preconditions:
- ISO monitoring endpoint is active and accepting log writes from the IRE service account.

Test Steps:
1. From an IRE test instance, send a test log entry to the ISO monitoring ingest endpoint (HTTPS POST).
2. Verify the log entry appears in the ISO monitoring system.

Expected Results:
- The POST request succeeds (HTTP 200 or equivalent).
- The log entry is visible in the ISO monitoring system.

Pass/Fail Criteria: Pass if the write succeeds. Fail if the request is rejected.

---

**TC-R13-05: Log read attempt from IRE to ISO monitoring endpoint is denied**

Requirement Reference: 13.5
Type: Security

Preconditions:
- ISO monitoring endpoint has write-only IAM permissions configured for the IRE service account.

Test Steps:
1. From an IRE test instance, attempt to read or list logs from the ISO monitoring endpoint (HTTP GET / list API).
2. Observe the response.

Expected Results:
- The read request is rejected with an authorization error.
- No Production log data is returned.

Pass/Fail Criteria: Pass if the read is denied. Fail if any log data is returned.

---

**TC-R13-06: ISO unavailability falls back to cached DNS and cert; alert within 5 minutes**

Requirement Reference: 13.6
Type: Integration

Preconditions:
- The ISO Test Account VPC attachment to the Transit Gateway is temporarily suspended (simulating ISO unavailability).
- The IRE local DNS resolver has cached entries (populated before the outage).
- Pre-staged certificates are stored in the IRE S3 fallback bucket.

Test Steps:
1. Suspend the ISO connectivity.
2. From an IRE test instance, query a DNS name that was previously cached.
3. Verify the cached result is returned.
4. Attempt certificate validation using the local S3 fallback.
5. Monitor the operations team alert channel.

Expected Results:
- Cached DNS entries are returned (up to 24-hour TTL).
- Certificate validation succeeds using the pre-staged local certificates.
- An operations team alert is received within 5 minutes of outage detection.

Pass/Fail Criteria: Pass if fallback DNS and certs work and the alert arrives within 5 minutes 30 seconds. Fail otherwise.

---

**TC-R13-07: IRE service accounts are separate from Production service accounts**

Requirement Reference: 13.3
Type: Configuration

Preconditions:
- The ISO Test AD is configured with both a Production scope and an IRE service account scope.

Test Steps:
1. List service accounts in the IRE scope.
2. List service accounts in the Production scope.
3. Verify there is no overlap.

Expected Results:
- No IRE-scoped service account exists in the Production scope.
- No Production service account can authenticate against IRE AD resources.

Pass/Fail Criteria: Pass if scopes are fully separated. Fail if any cross-scope account exists.

---

**TC-R13-08: PA Firewall restricts IRE-to-ISO traffic to approved ports only**

Requirement Reference: 13.2
Type: Security

Preconditions:
- PA Firewall zone policy is applied between the IRE and ISO zones.

Test Steps:
1. From the IRE test instance, attempt connections to the ISO environment on approved ports (TCP 636, 53, 443, 80).
2. Attempt connections on a non-approved port (e.g., TCP 22, TCP 8080).

Expected Results:
- Connections on approved ports succeed (subject to service availability).
- Connections on non-approved ports are denied and dropped by the PA Firewall.
- Deny entries appear in PA Firewall flow logs.

Pass/Fail Criteria: Pass if approved ports are reachable and non-approved ports are denied. Fail otherwise.


---

### Requirement 14: KMS Encryption at Rest

---

**TC-R14-01: Distinct CMKs exist per workload type in IRE and DR accounts**

Requirement Reference: 14.1, 14.2
Type: Configuration

Preconditions:
- IRE Test Account and DR Test Account CMKs have been provisioned.

Test Steps:
1. List all KMS CMKs in the IRE Test Account via `aws kms list-keys`.
2. Retrieve key metadata for each: alias, description.
3. List all KMS CMKs in the DR Test Account.
4. Compare the key ARNs between the two accounts.

Expected Results:
- IRE Test Account has distinct CMKs for: EBS, RDS, EKS PV, Lambda env vars, break-glass, and log encryption.
- DR Test Account has distinct CMKs for: EBS, RDS, EKS PV, and Lambda env vars.
- No key ARN appears in both accounts.

Pass/Fail Criteria: Pass if distinct CMKs exist per workload type in each account with no shared ARNs. Fail otherwise.

---

**TC-R14-02: CMK key policy denies all non-authorised principals**

Requirement Reference: 14.4
Type: Security

Preconditions:
- A non-authorised IAM role exists in the IRE Test Account (e.g., a generic developer role).

Test Steps:
1. Assume the non-authorised role.
2. Attempt `aws kms encrypt --key-id <ire-ebs-cmk-arn> --plaintext "test"`.

Expected Results:
- The call returns `AccessDenied`.
- The deny is attributed to the CMK key policy's explicit deny statement for non-authorised principals.

Pass/Fail Criteria: Pass if `AccessDenied` is returned. Fail if the encryption succeeds.

---

**TC-R14-03: CloudTrail KMS data events enabled and log Encrypt/Decrypt/GenerateDataKey**

Requirement Reference: 14.5
Type: Configuration

Preconditions:
- CloudTrail KMS data event logging is enabled in the IRE Test Account.

Test Steps:
1. Perform an authorised `aws kms encrypt` call using the IRE EBS CMK.
2. Perform an authorised `aws kms decrypt` call.
3. Query CloudTrail for the corresponding data events.

Expected Results:
- A CloudTrail data event exists for `Encrypt`, containing: key ARN, calling principal ARN, and action.
- A CloudTrail data event exists for `Decrypt` with the same fields.
- Events are delivered to the Centralised Logging Account S3 bucket.

Pass/Fail Criteria: Pass if events are captured with all required fields. Fail if events are absent or fields are missing.

---

**TC-R14-04: CMK deletion alert delivered within 24 hours; 30-day waiting period enforced**

Requirement Reference: 14.6
Type: Integration

Preconditions:
- An EventBridge rule monitors `ScheduleKeyDeletion` API calls.

Test Steps:
1. Attempt to schedule deletion of the IRE EBS CMK with a 30-day waiting period: `aws kms schedule-key-deletion --key-id <arn> --pending-window-in-days 30`.
2. Monitor the security team notification channel.
3. Attempt to schedule deletion with fewer than 30 days (e.g., 7 days) to verify the minimum waiting period.

Expected Results:
- An alert is delivered to the security team within 24 hours of the scheduling event.
- The attempt with fewer than 30 days is rejected (AWS KMS enforces a 7-day minimum; the 30-day policy is enforced via SCP or key policy).

Pass/Fail Criteria: Pass if the alert arrives within 24 hours. Fail otherwise.

---

**TC-R14-05: Automatic annual key rotation is enabled**

Requirement Reference: 14.3
Type: Configuration

Preconditions:
- IRE and DR CMKs are provisioned.

Test Steps:
1. For each CMK, call `aws kms get-key-rotation-status --key-id <arn>`.
2. Check `KeyRotationEnabled` field.

Expected Results:
- `KeyRotationEnabled == true` for all CMKs in IRE and DR accounts.

Pass/Fail Criteria: Pass if rotation is enabled on all CMKs. Fail if any CMK has rotation disabled.


---

### Requirement 15: Network Isolation and Traffic Control

---

**TC-R15-01: No direct VPC peering between IRE and Production**

Requirement Reference: 15.2
Type: Configuration

Preconditions:
- IRE Test VPC and Production Simulator VPC exist.

Test Steps:
1. Call `aws ec2 describe-vpc-peering-connections` in the IRE Test Account.
2. Check for any peering connection involving the Production Simulator VPC ID.

Expected Results:
- No VPC peering connections exist between the IRE VPC and the Production Simulator VPC.

Pass/Fail Criteria: Pass if no peering connection is found. Fail if any peering connection to the Production VPC exists.

---

**TC-R15-02: All IRE traffic routes through Transit Gateway and PA Firewall**

Requirement Reference: 2.3, 15.1
Type: Configuration

Preconditions:
- IRE Test VPC route tables are configured.

Test Steps:
1. Retrieve all route tables for IRE VPC subnets.
2. Verify that the default route (0.0.0.0/0) points to the TGW attachment.
3. Trace a packet from an IRE instance to the ISO environment using VPC flow logs to confirm it passes through the PA Firewall VPC.

Expected Results:
- All non-local routes in the IRE VPC point to the TGW attachment.
- VPC flow logs confirm traffic traverses the PA Firewall VPC before reaching any destination.

Pass/Fail Criteria: Pass if default route points to TGW and PA Firewall is confirmed in the traffic path. Fail otherwise.

---

**TC-R15-03: PA Firewall blocks IRE → Production during active recovery**

Requirement Reference: 15.3
Type: Security

Preconditions:
- An active recovery pipeline is running (IRE is in `RECOVERY_ACTIVE` state).
- A test destination IP in the Production Simulator CIDR range is available.

Test Steps:
1. From an IRE test instance, attempt a TCP connection to the Production Simulator IP on any port.
2. Inspect PA Firewall deny logs.

Expected Results:
- The connection is blocked.
- A deny entry appears in the PA Firewall flow log with the action `DENY`.

Pass/Fail Criteria: Pass if the connection is blocked and the deny log entry is present. Fail if the connection succeeds or no log entry is generated.

---

**TC-R15-04: PA Firewall logs all allowed and denied flows**

Requirement Reference: 15.4
Type: Configuration

Preconditions:
- PA Firewall logging is configured.
- A mix of allowed and denied test connections have been made.

Test Steps:
1. Query PA Firewall flow logs from the last 5 minutes.
2. Verify that each log entry contains: source IP, destination IP, port, protocol, and action (ALLOW or DENY).
3. Verify that logs are retained for at least 90 days.

Expected Results:
- All flow log entries contain the five required fields.
- Log entries from 90 days ago are still accessible.

Pass/Fail Criteria: Pass if all fields are present and 90-day retention is confirmed. Fail otherwise.

---

**TC-R15-05: No Elastic IPs or public endpoints on restored workloads**

Requirement Reference: 15.5
Type: Configuration

Preconditions:
- `test-ec2-linux-01` has been restored into the IRE.

Test Steps:
1. Call `aws ec2 describe-instances --instance-ids <restored-id>` and check `PublicIpAddress` and `PublicDnsName`.
2. Call `aws ec2 describe-addresses` to list Elastic IPs in the IRE account and verify none are associated with restored instances.

Expected Results:
- `PublicIpAddress` is null.
- `PublicDnsName` is null.
- No Elastic IP is associated with the restored instance.

Pass/Fail Criteria: Pass if no public IP or Elastic IP is present. Fail if any public address is found.

---

**TC-R15-06: Security group applied to workload within 60 seconds of running state**

Requirement Reference: 15.6
Type: Integration

Preconditions:
- A restore is in progress and the instance is about to reach `running` state.

Test Steps:
1. Record the timestamp when the instance transitions to `running`.
2. Poll `describe-instances` every 5 seconds to check the security group assignment.

Expected Results:
- `IRE-Restore-SG` is attached to the instance within 60 seconds of the `running` state transition.
- The security group is applied without manual intervention.

Pass/Fail Criteria: Pass if the SG is applied within 66 seconds (10% tolerance). Fail otherwise.

---

**TC-R15-07: SG application failure stops workload and alerts**

Requirement Reference: 15.7
Type: Integration

Preconditions:
- The SG assignment mechanism is configured to fail (e.g., the Lambda applying the SG is disabled).

Test Steps:
1. Initiate a restore and allow the SG application to fail.
2. Check the instance state.
3. Monitor the operations team alert channel.

Expected Results:
- The instance is halted (stopped state).
- An alert is sent to the operations team.
- The instance remains stopped until the SG is successfully applied.

Pass/Fail Criteria: Pass if instance is stopped and alert is sent. Fail if the instance continues running without a security group.

---

**TC-R15-08: VPC endpoint policy denies requests outside IRE account boundary**

Requirement Reference: 15.8
Type: Security

Preconditions:
- The IRE Test VPC has a VPC endpoint for S3 with an endpoint policy restricting access to the IRE Test Account's buckets.

Test Steps:
1. From an IRE test instance, attempt to access an S3 bucket in a different AWS account via the VPC endpoint: `aws s3 ls s3://external-account-bucket`.

Expected Results:
- The request is denied by the VPC endpoint policy.
- A denial entry is generated in CloudTrail.

Pass/Fail Criteria: Pass if access is denied. Fail if the request succeeds.


---

### Requirement 16: Compliance, Auditability, and Logging

---

**TC-R16-01: CloudTrail enabled with multi-region logging**

Requirement Reference: 16.1
Type: Configuration

Preconditions:
- CloudTrail trail is expected to be configured in the IRE Test Account.

Test Steps:
1. Call `aws cloudtrail describe-trails`.
2. Check `IsMultiRegionTrail`, `IncludeManagementEvents`, and `S3BucketName`.
3. Verify that the S3 bucket resides in the Centralised Logging Test Account.

Expected Results:
- `IsMultiRegionTrail == true`.
- Management events and KMS/S3/Lambda data events are captured.
- Logs are delivered to the Centralised Logging Test Account S3 bucket.

Pass/Fail Criteria: Pass if all three conditions are met. Fail otherwise.

---

**TC-R16-02: AWS Config enabled and recording resource changes**

Requirement Reference: 16.2
Type: Configuration

Preconditions:
- AWS Config recorder is expected to be active.

Test Steps:
1. Call `aws configservice describe-configuration-recorders`.
2. Verify `recordingGroup.allSupported == true` (or explicit resource types are listed).
3. Call `aws configservice describe-delivery-channels` and verify the delivery channel targets the Centralised Logging Account S3 bucket.

Expected Results:
- Config recorder is active for all resource types.
- Configuration history is delivered to the Centralised Logging Account.

Pass/Fail Criteria: Pass if recorder is active and delivery channel is correct. Fail otherwise.

---

**TC-R16-03: Recovery audit report produced within 30 minutes of pipeline completion**

Requirement Reference: 16.3
Type: Integration

Preconditions:
- A full recovery pipeline (restore → scan → approve → promote) has just completed.

Test Steps:
1. Record the pipeline completion timestamp from the Workflow Registry.
2. Monitor the Centralised Logging Account S3 bucket for the audit report.

Expected Results:
- A `RecoveryAuditReport` JSON file appears in the S3 bucket within 30 minutes of pipeline completion.
- The report contains: `workflowId`, `workloadId`, backup source, restore timestamp, scan results per tool, cleansing actions, approval record, promotion record, and DR activation status.

Pass/Fail Criteria: Pass if the report is present within 33 minutes (10% tolerance) with all required fields. Fail otherwise.

---

**TC-R16-04: Out-of-band IRE resource modification generates alert within 5 minutes**

Requirement Reference: 16.4
Type: Security

Preconditions:
- No active recovery pipeline is running for the test workload.
- A non-orchestration IAM principal exists in the IRE Test Account.

Test Steps:
1. Using the non-orchestration IAM principal, perform a resource modification (e.g., change a security group on a restored instance) outside of any pipeline.
2. Record the modification timestamp from CloudTrail.
3. Monitor the security operations team notification channel.

Expected Results:
- A high-severity alert is sent to the security operations team within 5 minutes of the modification.

Pass/Fail Criteria: Pass if the alert arrives within 5 minutes 30 seconds. Fail otherwise.

---

**TC-R16-05: CloudTrail logs retained 365 days with S3 Object Lock**

Requirement Reference: 16.5, 16.7
Type: Configuration

Preconditions:
- The Centralised Logging Account S3 log bucket has S3 Object Lock enabled in compliance mode.

Test Steps:
1. Check the S3 bucket's Object Lock configuration: `aws s3api get-object-lock-configuration`.
2. Verify `ObjectLockEnabled == Enabled`, `Mode == COMPLIANCE`, and `Days >= 365`.
3. Attempt to delete a log object from the bucket.

Expected Results:
- Object Lock is enabled in compliance mode with a retention of at least 365 days.
- Deletion attempt is rejected.

Pass/Fail Criteria: Pass if Object Lock is correctly configured and deletion is rejected. Fail otherwise.

---

**TC-R16-06: GuardDuty findings forwarded to SIEM within 5 minutes**

Requirement Reference: 16.6
Type: Integration

Preconditions:
- GuardDuty is enabled in the IRE Test Account.
- An EventBridge → Kinesis Data Firehose → SIEM pipeline is configured.
- A SIEM test endpoint is monitoring for incoming events.

Test Steps:
1. Generate a synthetic GuardDuty finding using the GuardDuty sample findings API: `aws guardduty create-sample-findings`.
2. Record the finding creation timestamp.
3. Monitor the SIEM test endpoint for the incoming finding.

Expected Results:
- The finding appears in the SIEM within 5 minutes of the GuardDuty finding creation timestamp.

Pass/Fail Criteria: Pass if the finding arrives within 5 minutes 30 seconds. Fail otherwise.


---

### Requirement 17: Recovery Time and Recovery Point Objectives

---

**TC-R17-01: RPO validation — last successful backup within 24 hours of recovery event**

Requirement Reference: 17.1
Type: Performance

Preconditions:
- All registered test workloads have backup schedules configured with a maximum 24-hour interval.
- At least one complete backup cycle has run.

Test Steps:
1. Declare a recovery event for `test-ec2-linux-01`.
2. Record the recovery event declaration timestamp.
3. Identify the most recent successful backup timestamp for this workload from the backup completion records.
4. Calculate the interval: `recovery_event_time - last_backup_time`.

Expected Results:
- The calculated interval is ≤ 24 hours.

Pass/Fail Criteria: Pass if the interval is ≤ 24 hours for all registered workloads. Fail if any workload's last backup is more than 24 hours before the recovery event.

---

**TC-R17-02: RTO validation — full EC2 pipeline completes within 8 hours**

Requirement Reference: 17.2
Type: Performance

Preconditions:
- A complete clean pipeline is expected (no scan failures, prompt approval, no retries).
- `test-ec2-linux-01` backup is available.

Test Steps:
1. Declare a recovery event for `test-ec2-linux-01` and record the `InitiateRestore` timestamp from the Workflow Registry.
2. Allow the full pipeline to complete (restore → scan → approve → promote).
3. Record the `PROMOTED` state timestamp from the Workflow Registry.
4. Calculate total elapsed time.

Expected Results:
- Total elapsed time from `InitiateRestore` to `PROMOTED` is ≤ 8 hours.

Pass/Fail Criteria: Pass if elapsed time is ≤ 8 hours. Fail if it exceeds 8 hours.

---

**TC-R17-03: RTO validation — full RDS pipeline completes within 8 hours**

Requirement Reference: 17.2
Type: Performance

Preconditions:
- `test-rds-mysql-01` backup is available.

Test Steps:
1. Same steps as TC-R17-02 but for `test-rds-mysql-01`.

Expected Results:
- Total elapsed time from `InitiateRestore` to `PROMOTED` is ≤ 8 hours.

Pass/Fail Criteria: Pass if elapsed time is ≤ 8 hours. Fail otherwise.

---

**TC-R17-04: RTO validation — full EKS pipeline completes within 8 hours**

Requirement Reference: 17.2
Type: Performance

Preconditions:
- `test-eks-cluster-01` backup is available.

Test Steps:
1. Same steps as TC-R17-02 but for `test-eks-cluster-01`.

Expected Results:
- Total elapsed time from `InitiateRestore` to `PROMOTED` is ≤ 8 hours.

Pass/Fail Criteria: Pass if elapsed time is ≤ 8 hours. Fail otherwise.

---

**TC-R17-05: RTO validation — full Lambda pipeline completes within 8 hours**

Requirement Reference: 17.2
Type: Performance

Preconditions:
- `test-lambda-01` backup is available.

Test Steps:
1. Same steps as TC-R17-02 but for `test-lambda-01`.

Expected Results:
- Total elapsed time from `InitiateRestore` to `PROMOTED` is ≤ 8 hours.

Pass/Fail Criteria: Pass if elapsed time is ≤ 8 hours. Fail otherwise.

---

**TC-R17-06: 6-hour checkpoint notification sent when pipeline not at validation stage**

Requirement Reference: 17.3
Type: Integration

Preconditions:
- A recovery pipeline is running with deliberately slow scanning (simulated via delayed scanner responses).
- The pipeline has not reached `QualysClean` state at the 6-hour mark.

Test Steps:
1. Record the `InitiateRestore` timestamp.
2. Allow 6 hours to elapse without the pipeline reaching `QualysClean`.
3. Monitor the recovery team and incident manager notification channels.

Expected Results:
- A notification is sent at T+6 hours (±5 minutes) to both the recovery team and incident manager.
- The notification includes the current pipeline state and an estimated completion time.

Pass/Fail Criteria: Pass if notification is sent at approximately T+6 hours with the required content. Fail otherwise.

---

**TC-R17-07: Concurrent workload pipelines each maintain independent 8-hour RTO**

Requirement Reference: 17.4
Type: Performance

Preconditions:
- Five test workload backups are available (one EC2, one RDS, one EKS, one Lambda, one EC2).

Test Steps:
1. Declare five concurrent recovery events with 5-minute staggered start times.
2. Allow all five pipelines to run concurrently to completion.
3. For each pipeline, calculate the elapsed time from its individual `InitiateRestore` timestamp to `PROMOTED`.

Expected Results:
- Each pipeline completes within 8 hours of its individual `InitiateRestore` timestamp.
- No pipeline's RTO is affected by other concurrent pipelines.

Pass/Fail Criteria: Pass if all five pipelines meet their individual 8-hour RTO. Fail if any pipeline exceeds 8 hours due to resource contention.


---

### Requirement 18: Identity and Access Management

---

**TC-R18-01: IAM permission boundary applied to all roles in IRE account**

Requirement Reference: 18.1
Type: Configuration

Preconditions:
- All IAM roles in the IRE Test Account have been provisioned.

Test Steps:
1. List all IAM roles in the IRE Test Account: `aws iam list-roles`.
2. For each role, call `aws iam get-role --role-name <name>` and check `PermissionsBoundary`.
3. Verify the boundary policy explicitly denies IAM, S3 log bucket, and CloudTrail management actions.

Expected Results:
- Every IAM role in the IRE account has a `PermissionsBoundary` attached.
- The boundary policy's deny statements cover: `iam:*`, `s3:*` on log buckets, and `cloudtrail:*` for non-orchestration roles.

Pass/Fail Criteria: Pass if every role has a boundary with the required denies. Fail if any role lacks a boundary or the boundary is missing required denies.

---

**TC-R18-02: Wildcard IAM actions denied in all role policies**

Requirement Reference: 18.2
Type: Configuration

Preconditions:
- All managed and inline policies attached to IRE roles are accessible.

Test Steps:
1. List all managed policies and inline policies attached to all IRE roles.
2. Inspect each policy document for any statement with `"Action": "*"` or `"Action": "service:*"`.

Expected Results:
- No policy document attached to any IRE role contains a wildcard action statement.

Pass/Fail Criteria: Pass if no wildcard actions are found. Fail if any wildcard action statement exists.

---

**TC-R18-03: API calls to workload resources from outside IRE VPC are denied**

Requirement Reference: 18.3
Type: Security

Preconditions:
- An IAM role in the IRE Test Account has a condition key `aws:SourceVpc` set to the IRE VPC ID.
- A test host outside the IRE VPC is available.

Test Steps:
1. Assume the IAM role from the external test host.
2. Attempt `aws ec2 describe-instances` targeting IRE workload resources.

Expected Results:
- The call returns `AccessDenied` due to the `aws:SourceVpc` condition not matching.

Pass/Fail Criteria: Pass if the call is denied from outside the IRE VPC. Fail if it succeeds.

---

**TC-R18-04: New IAM role creation notifies security team within 15 minutes**

Requirement Reference: 18.4
Type: Integration

Preconditions:
- An EventBridge rule monitors `CreateRole` CloudTrail events.

Test Steps:
1. Create a test IAM role in the IRE Test Account: `aws iam create-role --role-name test-new-role ...`.
2. Record the creation timestamp.
3. Monitor the security operations team notification channel.

Expected Results:
- A notification is received within 15 minutes of the role creation timestamp.
- The notification references the new role ARN or name.

Pass/Fail Criteria: Pass if notification arrives within 16 minutes 30 seconds. Fail otherwise.

Note: Delete the test role after the test to maintain a clean environment.

---

**TC-R18-05: Root account has no access keys and MFA enforced**

Requirement Reference: 18.5
Type: Configuration

Preconditions:
- AWS Organizations management-level access is available.

Test Steps:
1. Check root account credential report: `aws iam generate-credential-report` then `aws iam get-credential-report`.
2. For the root account row, inspect: `access_key_1_active`, `access_key_2_active`, and `mfa_active`.

Expected Results:
- `access_key_1_active == false`.
- `access_key_2_active == false`.
- `mfa_active == true`.

Pass/Fail Criteria: Pass if both access key fields are false and MFA is active. Fail if any access key is active or MFA is inactive.

---

**TC-R18-06: SSO federation disabled at IRE account level**

Requirement Reference: 3.1
Type: Configuration

Preconditions:
- The IRE Test Account is accessible.

Test Steps:
1. Attempt to access the IRE Test Account via the corporate SSO portal.
2. Attempt to create an IAM Identity Center permission set that targets the IRE Test Account.

Expected Results:
- The SSO portal does not list the IRE Test Account as an available account.
- The attempt to assign a permission set targeting the IRE account is rejected.

Pass/Fail Criteria: Pass if SSO access to the IRE account is not available. Fail if SSO authentication succeeds.


---

## 6. Security Test Cases

---

**SECSCAN-01: Attempt Production resource access from IRE workload — denied**

Objective: Verify that an IRE workload cannot read, modify, or enumerate Production account resources directly.

Preconditions: A restored EC2 instance is running in the IRE Test Account. The Production Simulator Account contains tagged EC2 instances and S3 buckets.

Test Steps:
1. From the IRE instance, attempt `aws ec2 describe-instances` targeting the Production Simulator Account ID.
2. Attempt `aws s3 ls s3://production-simulator-bucket`.

Expected Results: Both calls are denied. No Production resource data is returned. Deny entries are generated in CloudTrail.

Pass/Fail Criteria: Pass if both calls are denied. Fail if any Production data is accessible.

---

**SECSCAN-02: Attempt IRE-to-internet connection — denied at SG/NACL level**

Objective: Verify that no direct internet egress path exists from the IRE.

Preconditions: A restored EC2 instance is running in the IRE Test Restore Subnet. An internet-accessible destination IP is available for testing.

Test Steps:
1. From the IRE instance, attempt `curl --connect-timeout 10 https://1.1.1.1`.
2. Attempt `curl --connect-timeout 10 https://example.com`.

Expected Results: Both connection attempts time out without receiving any response. VPC flow logs show the traffic is blocked at the NACL or security group level. No internet gateway or NAT gateway route is present.

Pass/Fail Criteria: Pass if both connections are blocked. Fail if either connection succeeds.

---

**SECSCAN-03: Attempt to modify CloudTrail logs from IRE — denied by S3 Object Lock**

Objective: Verify that CloudTrail log objects in the Centralised Logging Account cannot be modified or deleted from the IRE.

Preconditions: A CloudTrail log object exists in the Centralised Logging Account S3 bucket. S3 Object Lock is enabled in compliance mode.

Test Steps:
1. From an IRE role, attempt `aws s3 rm s3://ire-centralized-logs-bucket/cloudtrail/...`.
2. Attempt `aws s3 cp modified-log.json s3://ire-centralized-logs-bucket/cloudtrail/...` (overwrite).

Expected Results: Both operations return `AccessDenied` or `InvalidRequest`. The original log object is unmodified.

Pass/Fail Criteria: Pass if both operations fail and the object is intact. Fail otherwise.

---

**SECSCAN-04: Attempt to promote unscanned workload — blocked by Orchestrator**

Objective: Verify that the Promotion Service cannot be invoked for a workload that has not completed all three scans.

Preconditions: A workload is in `RESTORE_COMPLETE` state (scans have not started).

Test Steps:
1. Directly invoke the Promotion Service Lambda for the unscanned workload (bypassing the normal Orchestrator flow).

Expected Results: The Promotion Service rejects the request, returning a status indicating that scan results are absent or incomplete. No artefact transfer is initiated.

Pass/Fail Criteria: Pass if the promotion is rejected. Fail if any transfer proceeds.

---

**SECSCAN-05: Attempt to use corporate SSO credentials in IRE — SSO federation disabled**

Objective: Verify that corporate SSO identity federation cannot be used to access the IRE account.

Preconditions: A valid corporate SSO identity exists (non-break-glass user).

Test Steps:
1. Attempt to assume any IAM role in the IRE Test Account using a federated SSO identity (via AWS Identity Center or SAML assertion).

Expected Results: The role assumption fails. The IRE account does not appear as an available account in the SSO portal. No federated session is established.

Pass/Fail Criteria: Pass if SSO access is denied at the account level. Fail if any federated session is established.

---

**SECSCAN-06: Attempt to access IRE KMS CMK from non-authorised role — denied by key policy**

Objective: Verify that the KMS key policy's explicit deny statement blocks non-authorised principals.

Preconditions: A non-authorised IAM role exists in the IRE Test Account (not listed in the CMK key policy's authorised principals).

Test Steps:
1. Assume the non-authorised role.
2. Attempt `aws kms encrypt --key-id <ire-ebs-cmk-arn> --plaintext "test"`.
3. Attempt `aws kms decrypt` with a known ciphertext.

Expected Results: Both operations return `AccessDenied`. The denial is attributed to the explicit deny statement in the CMK key policy.

Pass/Fail Criteria: Pass if both operations are denied. Fail if either succeeds.

---

**SECSCAN-07: Inject malware into test workload — verify quarantine before promotion**

Objective: Verify the end-to-end security control that prevents a malware-infected workload from being promoted to DR.

Preconditions: `test-ec2-infected-01` (with a known simulated malware indicator) is ready for restore.

Test Steps:
1. Restore `test-ec2-infected-01` through the normal pipeline.
2. Allow all three scanners to run. At least one scanner should detect the malware indicator.
3. Observe the pipeline state.
4. Verify that no promotion is attempted.

Expected Results: The workload is quarantined. Pipeline state is `QUARANTINED`. Security operations is notified within 15 minutes. No artefact is transferred to the DR Test Account.

Pass/Fail Criteria: Pass if the workload is quarantined before any promotion and no artefact reaches the DR account. Fail if the workload bypasses the quarantine and a promotion is attempted.

---

**SECSCAN-08: Attempt IAM privilege escalation — denied by SCP**

Objective: Verify that SCPs prevent IAM privilege escalation actions within the IRE account.

Preconditions: An IAM role in the IRE Test Account attempts to escalate its own privileges.

Test Steps:
1. From an IRE role, attempt `aws iam create-policy-version` to introduce a wildcard allow policy.
2. Attempt `aws iam attach-role-policy` to attach an existing admin policy to the role.

Expected Results: Both calls return `AccessDenied` due to the OU-level SCP denying privilege escalation actions.

Pass/Fail Criteria: Pass if both operations are denied. Fail if either succeeds.


---

## 7. Performance Tests

---

**PERF-01: RPO measurement — backup completes within 24-hour window**

Objective: Confirm that the backup cadence meets the 24-hour RPO target for all registered workloads.

Method: Over a 7-day period, record all backup completion timestamps for each registered workload. For each pair of consecutive backup completion timestamps, calculate the interval. Verify that no interval exceeds 24 hours.

Metric: Maximum consecutive backup interval per workload.
Target: ≤ 24 hours for every workload.

Pass/Fail Criteria: Pass if no workload has a backup gap exceeding 24 hours over the 7-day observation period. Fail if any gap exceeds 24 hours.

---

**PERF-02: RTO measurement — full EC2 recovery pipeline completes within 8 hours**

Objective: Measure the wall-clock time for a complete EC2 workload recovery pipeline under clean conditions.

Workload: `test-ec2-linux-01`
Start event: `InitiateRestore` state in Workflow Registry.
End event: `PROMOTED` state in Workflow Registry.

Method: Execute the full pipeline three times (on separate days to avoid caching effects). Record the elapsed time for each run. Report the maximum and average elapsed time.

Target: Maximum elapsed time ≤ 8 hours across all three runs.

Pass/Fail Criteria: Pass if all three runs complete within 8 hours. Fail if any run exceeds 8 hours.

---

**PERF-03: RTO measurement — full RDS recovery pipeline completes within 8 hours**

Objective: Same as PERF-02 for an RDS workload.

Workload: `test-rds-mysql-01`
Target: Maximum elapsed time ≤ 8 hours.

Pass/Fail Criteria: Pass if all three runs complete within 8 hours. Fail otherwise.

---

**PERF-04: RTO measurement — full EKS recovery pipeline completes within 8 hours**

Objective: Same as PERF-02 for an EKS workload.

Workload: `test-eks-cluster-01`
Target: Maximum elapsed time ≤ 8 hours.

Pass/Fail Criteria: Pass if all three runs complete within 8 hours. Fail otherwise.

---

**PERF-05: RTO measurement — full Lambda recovery pipeline completes within 8 hours**

Objective: Same as PERF-02 for a Lambda workload.

Workload: `test-lambda-01`
Target: Maximum elapsed time ≤ 8 hours.

Pass/Fail Criteria: Pass if all three runs complete within 8 hours. Fail otherwise.

---

**PERF-06: Concurrent pipeline test — 5 workloads running concurrently, each within 8-hour RTO**

Objective: Verify that the 8-hour RTO is maintained for each workload when five recovery pipelines run simultaneously.

Workloads: `test-ec2-linux-01`, `test-rds-mysql-01`, `test-eks-cluster-01`, `test-lambda-01`, `test-ec2-windows-01`.

Method:
1. Declare five concurrent recovery events within a 5-minute window.
2. Allow all five pipelines to run to completion.
3. For each pipeline, calculate elapsed time from its individual `InitiateRestore` timestamp to `PROMOTED`.
4. Record Step Functions concurrent execution metrics and DynamoDB throughput during the test.

Target: Every pipeline completes within 8 hours of its individual start time.

Pass/Fail Criteria: Pass if all five pipelines meet their individual 8-hour RTO. Fail if any pipeline exceeds 8 hours.

---

**PERF-07: Scan timing — Tanium scan completes within 2 hours**

Objective: Measure the time from Tanium scan initiation to result receipt under load.

Method: Execute Tanium scans for five workloads concurrently. Record initiation and result timestamps for each.

Target: All five Tanium scan results are received within 2 hours of their individual initiation timestamps.

Pass/Fail Criteria: Pass if all scans return results within 2 hours. Fail if any scan exceeds 2 hours.

---

**PERF-08: Scan timing — CrowdStrike scan completes within 2 hours**

Objective: Same as PERF-07 for CrowdStrike.

Target: All five CrowdStrike scan results received within 2 hours of their individual initiation timestamps.

Pass/Fail Criteria: Pass if all scans return results within 2 hours. Fail otherwise.

---

**PERF-09: Scan timing — Qualys scan completes within 4 hours**

Objective: Same as PERF-07 for Qualys.

Target: All five Qualys vulnerability reports received within 4 hours of their individual initiation timestamps.

Pass/Fail Criteria: Pass if all scans return results within 4 hours. Fail otherwise.

---

**PERF-10: Restore initiation latency — within 5 minutes of event declaration**

Objective: Verify that restore initiation latency meets the 5-minute SLA under concurrent load.

Method: Declare five concurrent recovery events. For each, record the time between event declaration and the `RESTORE_PENDING` state transition.

Target: Each restore initiates within 5 minutes of its event declaration.

Pass/Fail Criteria: Pass if all five initiations are within 5 minutes. Fail if any exceeds 5 minutes.


---

## 8. Test Traceability Matrix

The following table maps every acceptance criterion from Requirements 1–18 to its covering test case(s). Full traceability is required before test exit criteria can be satisfied.

| Requirement | Acceptance Criterion | Test Case(s) |
|---|---|---|
| 1 | 1.1 — Protect all in-scope workloads | TC-R1-01 |
| 1 | 1.2 — Encrypt backups with source CMK | TC-R1-04 |
| 1 | 1.3 — Backup completion record with required fields | TC-R1-01 |
| 1 | 1.4 — Backup failure alert within 15 minutes | TC-R1-02 |
| 1 | 1.5 — Retention 30–365 days | TC-R1-05 |
| 1 | 1.6 — TLS 1.2+ for backup data in transit | TC-R1-06 |
| 1 | 1.7 — Alert for omitted workload within 15 minutes | TC-R1-03, PBT-P1 |
| 2 | 2.1 — IRE in same OU as Production and DR | TC-R2-05 |
| 2 | 2.2 — SCPs deny internet egress, VPC peering, IAM escalation | TC-R2-01, TC-R2-02, SECSCAN-08 |
| 2 | 2.3 — All IRE traffic through TGW and PA Firewall | TC-R15-02 |
| 2 | 2.4 — PA Firewall allowlist-only rules | TC-R2-03, TC-R15-01 |
| 2 | 2.5 — No inbound/outbound internet at network layer | SECSCAN-02 |
| 2 | 2.6 — Steady-state: no Production routes or sessions | TC-R2-04 |
| 3 | 3.1 — Break-glass only; SSO federation disabled | SECSCAN-05, TC-R18-06 |
| 3 | 3.2 — Break-glass creds in Secrets Manager, encrypted with CMK | TC-R3-01 |
| 3 | 3.3 — CloudTrail audit event on break-glass use/rotation | TC-R3-04 |
| 3 | 3.4 — MFA required for break-glass login | TC-R3-01, TC-R3-02 |
| 3 | 3.5 — Non-approved IP denied with alert | TC-R3-03, PBT-P2 |
| 3 | 3.6 — MFA failure increments counter and generates audit event | TC-R3-07 |
| 3 | 3.7 — Credential rotation ≤ 90 days | TC-R3-06 |
| 3 | 3.8 — Rotation within 1 hour of session end | TC-R3-05 |
| 4 | 4.1 — Restore initiates within 5 minutes | TC-R4-01, PERF-10 |
| 4 | 4.2 — Source CMK decrypt, IRE CMK re-encrypt | TC-R4-02, PBT-P3 |
| 4 | 4.3 — Restore completion record with required fields | TC-R4-06 |
| 4 | 4.4 — Halt after 3 retries with notification within 15 minutes | TC-R4-03, TC-R4-05 |
| 4 | 4.5 — Restore into IRE subnets with no public IPs | TC-R15-05 |
| 4 | 4.6 — Outbound blocked until scan job registered | TC-R4-04 |
| 4 | 4.7 — Auto-retry with 2-minute wait; no halt below 3 attempts | TC-R4-03, PBT-P4 |
| 5 | 5.1 — EC2 restore via EBS snapshot and pre-registered launch template | TC-R5-01 |
| 5 | 5.2 — EBS volumes encrypted with IRE EBS CMK | TC-R5-02 |
| 5 | 5.3 — IRE-Restore-SG applied; only scanner CIDRs inbound | TC-R5-03 |
| 5 | 5.4 — Tags, IAM profile, volume config match backup metadata | TC-R5-04 |
| 5 | 5.5 — EC2 restore failure alert within 5 minutes | TC-R5-05 |
| 5 | 5.6 — Audit log entry on successful EC2 restore | TC-R5-06 |
| 6 | 6.1 — RDS restore via native snapshot API | TC-R6-01 |
| 6 | 6.2 — IRE RDS CMK applied before instance available | TC-R6-02 |
| 6 | 6.3 — SG permits only scanner CIDRs on DB port | TC-R6-04 |
| 6 | 6.4 — Public accessibility disabled before available | TC-R6-03 |
| 6 | 6.5 — Engine incompatibility alert within 5 minutes | TC-R6-05 |
| 6 | 6.6 — Parameter group and option group names in restore metadata | TC-R6-06 |
| 7 | 7.1 — PVs and manifests restored within 60 minutes | TC-R7-01 |
| 7 | 7.2 — PVs encrypted with IRE EKS CMK before pods Running | TC-R7-02 |
| 7 | 7.3 — Network policies allow only scanner endpoints inbound | TC-R7-03 |
| 7 | 7.4 — Original namespace, quotas, service accounts restored | TC-R7-01 |
| 7 | 7.5 — Production registry substitute after 30s timeout | TC-R7-04 |
| 7 | 7.6 — External LoadBalancer services disabled on restore | TC-R7-05 |
| 7 | 7.7 — PV/manifest failure triggers rollback and report | TC-R7-06 |
| 8 | 8.1 — Lambda deployed with all attributes | TC-R8-01 |
| 8 | 8.2 — Env vars encrypted with IRE Lambda CMK | TC-R8-02 |
| 8 | 8.3 — Lambda deployed inside VPC, outbound restricted to scanner endpoints | TC-R8-04 |
| 8 | 8.4 — Original execution role ARN in metadata only | TC-R8-05 |
| 8 | 8.5 — IRE-scoped execution role as active role | TC-R8-04 |
| 8 | 8.6 — Production ARN in env var disables function and flags for review | TC-R8-03 |
| 8 | 8.7 — Restore failure leaves partial resources disabled | TC-R8-06 |
| 9 | 9.1 — Scan sequence: Tanium → CrowdStrike → Qualys | TC-R9-01, PBT-P5 |
| 9 | 9.2 — Tanium scan result within 2 hours; CLEAN = zero threats | TC-R9-02, PERF-07 |
| 9 | 9.3 — CrowdStrike scan result within 2 hours; CLEAN = zero detections | TC-R9-03, PERF-08 |
| 9 | 9.4 — Qualys scan result within 4 hours; CLEAN = zero CVSS 7.0+ | TC-R9-04, PERF-09 |
| 9 | 9.5 — CVSS 7.0+ quarantines workload; alert within 15 minutes | TC-R9-05, PBT-P6, SECSCAN-07 |
| 9 | 9.6 — Promotion blocked until all 3 scanners clean | TC-R9-07, PBT-P7, SECSCAN-04 |
| 9 | 9.7 — Scanner timeout treated as failure; alert within 15 minutes | TC-R9-06 |
| 9 | 9.8 — Scanner endpoints pre-approved through PA Firewall | TC-R2-03 |
| 9 | 9.9 — Scan audit records in immutable log for 365 days | TC-R9-08 |
| 10 | 10.1 — Tanium remediation removes threat and logs result | TC-R10-01 |
| 10 | 10.2 — CrowdStrike remediation isolates and logs result | TC-R10-02 |
| 10 | 10.3 — Remediation failure/timeout triggers quarantine within 30 minutes | TC-R10-03 |
| 10 | 10.4 — Post-remediation full rescan before promotion | TC-R10-04 |
| 10 | 10.5 — Quarantine event logged with workload ID, tool, timestamp | TC-R10-05 |
| 11 | 11.1 — Promotion initiated after all checks pass | TC-R11-01 |
| 11 | 11.2 — TLS 1.2+ for transfer; DR CMK re-encryption | TC-R11-06, TC-R11-07 |
| 11 | 11.3 — Human approval gate; expiry after 4 hours with notification | TC-R11-02 |
| 11 | 11.4 — Promotion audit record with approver identity | TC-R11-05 |
| 11 | 11.5 — Promotion failure: rollback and 15-minute notification | TC-R11-08 |
| 11 | 11.6 — DR infrastructure mismatch aborts promotion | TC-R11-04 |
| 11 | 11.7 — Checksum mismatch aborts and rolls back; PBT | TC-R11-03, PBT-P9 |
| 12 | 12.1 — DR has pre-provisioned infrastructure | TC-R12-01 |
| 12 | 12.2 — Promoted artefact validated as importable before activation-ready | TC-R12-01 |
| 12 | 12.3 — DR data encrypted with DR CMK | TC-R12-03 |
| 12 | 12.4 — Activation event in CloudTrail with workload ID and IAM principal | TC-R12-02 |
| 13 | 13.1 — IRE permits outbound to ISO for approved services only | TC-R13-01, TC-R13-08 |
| 13 | 13.2 — PA Firewall restricts IRE-to-ISO to approved ports | TC-R13-08 |
| 13 | 13.3 — ISO uses IRE-scoped service accounts, separate from Production | TC-R13-01, TC-R13-07 |
| 13 | 13.4 — ISO DNS resolves only allowlisted zones; REFUSED/NXDOMAIN otherwise | TC-R13-02, TC-R13-03, PBT-P8 |
| 13 | 13.5 — ISO monitoring: write-only for IRE; no read access to Production logs | TC-R13-04, TC-R13-05 |
| 13 | 13.6 — ISO unavailable: DNS cache fallback, cert fallback; alert within 5 min | TC-R13-06 |
| 14 | 14.1 — Distinct CMK per workload type in IRE | TC-R14-01 |
| 14 | 14.2 — Distinct CMK per workload type in DR | TC-R14-01 |
| 14 | 14.3 — Automatic annual key rotation | TC-R14-05 |
| 14 | 14.4 — KMS key policy: explicit deny for non-authorised principals | TC-R14-02, SECSCAN-06 |
| 14 | 14.5 — CloudTrail KMS data events for Encrypt/Decrypt/GenerateDataKey | TC-R14-03 |
| 14 | 14.6 — CMK deletion alert within 24 hours; 30-day waiting period | TC-R14-04 |
| 15 | 15.1 — PA Firewall inspects and denies non-matching traffic | TC-R2-03, TC-R15-01 |
| 15 | 15.2 — No direct VPC peering IRE ↔ Production | TC-R15-01 |
| 15 | 15.3 — PA Firewall blocks IRE → Production during recovery | TC-R15-03 |
| 15 | 15.4 — PA Firewall logs all flows; retained 90 days | TC-R15-04 |
| 15 | 15.5 — No Elastic IPs or public endpoints on restored workloads | TC-R15-05 |
| 15 | 15.6 — Security group applied within 60 seconds of running state | TC-R15-06 |
| 15 | 15.7 — SG application failure stops workload and alerts | TC-R15-07 |
| 15 | 15.8 — VPC endpoint policy denies out-of-account requests | TC-R15-08 |
| 16 | 16.1 — CloudTrail multi-region, management and data events | TC-R16-01 |
| 16 | 16.2 — AWS Config recording all resource changes | TC-R16-02 |
| 16 | 16.3 — End-to-end audit report within 30 minutes of pipeline completion | TC-R16-03 |
| 16 | 16.4 — Out-of-band modification alert within 5 minutes | TC-R16-04, SECSCAN-01 |
| 16 | 16.5 — Logs retained 365 days | TC-R16-05 |
| 16 | 16.6 — GuardDuty findings to SIEM within 5 minutes | TC-R16-06 |
| 16 | 16.7 — S3 Object Lock compliance mode; deletion denied | TC-R16-05, SECSCAN-03 |
| 17 | 17.1 — RPO ≤ 24 hours | TC-R17-01, PERF-01 |
| 17 | 17.2 — RTO ≤ 8 hours per workload type | TC-R17-02, TC-R17-03, TC-R17-04, TC-R17-05, PERF-02, PERF-03, PERF-04, PERF-05 |
| 17 | 17.3 — 6-hour checkpoint notification | TC-R17-06 |
| 17 | 17.4 — Concurrent pipelines: independent 8-hour RTO | TC-R17-07, PERF-06 |
| 18 | 18.1 — IAM permission boundaries on all IRE roles | TC-R18-01 |
| 18 | 18.2 — Wildcard IAM actions denied | TC-R18-02 |
| 18 | 18.3 — API calls from outside IRE VPC denied (aws:SourceVpc) | TC-R18-03 |
| 18 | 18.4 — New IAM role creation notifies security team within 15 minutes | TC-R18-04 |
| 18 | 18.5 — Root account: no access keys, MFA enforced | TC-R18-05 |


---

## 9. Test Exit Criteria

A test cycle is considered complete and the solution is ready for production deployment only when all of the following exit criteria are satisfied.

### 9.1 Property-Based Tests

- All nine PBT cases (PBT-P1 through PBT-P9) have executed successfully with a minimum of 100 iterations each.
- No Hypothesis shrunk counterexample is present for any property.
- All property tests are tagged with the correct design property reference comment.

### 9.2 Functional Test Cases

- All test cases in Section 5 (TC-Rn-nn) have been executed and carry a Pass verdict.
- Zero test cases carry a Fail or Blocked verdict.
- Any test case that was skipped due to environment constraints is documented with a written risk acceptance and a re-test plan.

### 9.3 Security Test Cases

- All eight security test cases in Section 6 (SECSCAN-01 through SECSCAN-08) carry a Pass verdict.
- No security test has exposed an unmitigated control bypass.

### 9.4 Performance Tests

- All ten performance tests in Section 7 (PERF-01 through PERF-10) have been executed.
- All timing targets have been met:
  - RPO: ≤ 24 hours for all workloads (PERF-01).
  - RTO: ≤ 8 hours for all workload types under single and concurrent load (PERF-02 to PERF-06).
  - Tanium scan: ≤ 2 hours (PERF-07).
  - CrowdStrike scan: ≤ 2 hours (PERF-08).
  - Qualys scan: ≤ 4 hours (PERF-09).
  - Restore initiation latency: ≤ 5 minutes (PERF-10).

### 9.5 Defect Status

- Zero open defects with severity Critical.
- Zero open defects with severity High.
- All Medium and Low defects have an accepted resolution plan with target fix date.

### 9.6 Traceability

- The Test Traceability Matrix in Section 8 has 100% coverage of all acceptance criteria from Requirements 1–18.
- Every acceptance criterion maps to at least one executed test case with a Pass verdict.
- No untested acceptance criterion remains.

### 9.7 Test Reporting

- A final test execution report has been produced summarising:
  - Total test cases executed, passed, failed, and skipped.
  - PBT iteration counts and any shrunk counterexamples.
  - Performance measurements with timestamps.
  - Defect log with status.
  - Sign-off from the security operations lead, recovery engineering lead, and compliance officer.

