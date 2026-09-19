---
title: "Testing and Evaluation"
date: 2026-09-18T23:00:00+07:00
weight: 6
chapter: false
pre: "<b>5.6. </b>"
---



---

### 1. Testing Strategy and Verification Matrix

The project enforces a multi-layered verification strategy spanning Static Analysis, Unit Testing, Mocked Integration Testing, IaC Validation, and Negative Security Mutation Checks:

| Test Suite / Tool | Target Under Test | Outcome | Pass Rate | Technical Notes |
| :--- | :--- | :---: | :---: | :--- |
| **ESLint (`npm run lint`)** | All JS/TS Source Code | **PASS** | 100% | Clean lint run; removes unused variables without altering runtime behavior. |
| **Backend Tests (`npm test`)** | API, Auth, Quota, Worker, Lifecycle, S3 | **PASS** | **82 Pass / 0 Fail** | 82 passed; 2 live MySQL integration tests skipped due to isolated test environments. |
| **Frontend Tests (`npm test`)** | UI Components, Formatters, API Client | **PASS** | **6 Pass / 0 Fail** | Validates icons, byte formatters, and CSRF token propagation. |
| **Frontend Build (`npm run build`)** | React TypeScript compile & bundle | **PASS** | 100% | Bundled to `frontend/dist/` successfully. |
| **Dependency Audit (`npm audit`)** | Vulnerability scanning | **PASS** | 0 High / 0 Crit | AWS SDK updated; zero High/Critical vulnerabilities remaining. |
| **Terraform Validate (`terraform`)** | HCL configurations in `infra/terraform/` | **PASS** | 100% | Valid HCL 1.10.5 syntax; AWS Provider 5.100.0 locked. |
| **HCL Security Mutation (`check_infra.py`)** | Negative mutation security invariants | **PASS** | **13/13 Cases** | Blocks 13 dangerous mutation cases: public DBs, plaintext secrets, IMDSv1, SSH, NAT. |
| **Launcher Tests (`test_launch.py`)** | Host bootstrap & secret injection | **PASS** | **3/3 Tests** | Verifies SSM SecureString fetching and fails closed on missing parameters. |
| **Packaging & Rollback Tests** | Release verification & recovery | **PASS** | **6/6 Tests** | Enforces SHA-256 validation, rejects `.env` files, triggers automated rollbacks. |

---

### 2. Backend Test Breakdown (82 Tests Passed)

The test suite leverages the native Node.js test runner (`node --test`), covering all critical evolutionary phases:

* **Phase 1 (Foundations & S3 Mocking)**: Validates basic file APIs, S3 mock adapters, and network exception handlers.
* **Phase 2 (Folder Hierarchy)**: Tests parent-child folder structures, renaming, moving, and blocking non-empty folder deletions.
* **Phase 3 (Session Security & CSRF)**: Verifies DB-backed Opaque Sessions, HttpOnly cookies, and strict rejection of requests missing `X-CSRF-Token`.
* **Phase 4 (Storage Quota)**: Tests byte allocation arithmetic, blocking over-quota requests, and total quota summation.
* **Phase 5 (Direct Upload & Presigned POST)**: Validates policy generation, size boundaries, and mandatory `incoming/` prefixes.
* **Phase 6 (Durable Worker & Lease Fencing)**: Tests race conditions across parallel workers, lease expiration, renewals, and exponential retries.
* **Phase 7 (Lifecycles & Reconciliation)**: Verifies Trash, Restore, version-aware S3 batch deletion (`DeleteObjects`), orphan scanning, and self-healing quota alignment.
* **Phase 10A (Observability & Health Probes)**: Tests sensitive data redaction in logs, `requestId` correlation, and `/health/live` / `/health/ready` endpoints.

---

### 3. Testing Boundaries and Verification Limitations

To ensure objective technical reporting, operational boundaries are documented explicitly:

1. **Not Deployed to AWS Production**:
   * Cloud resources (VPC, Subnets, EC2, RDS, Lambda, SQS, CloudWatch) have been statically verified via Terraform (`validate` and `check_infra.py`).
   * No `terraform apply` commands have been executed on live AWS accounts.
2. **Mocked Integration Testing**:
   * Unit tests utilize local mocks and stubs for Amazon S3, AWS Lambda, and Amazon SQS.
   * Local passes do not verify live AWS IAM policies, bucket policies, or physical cloud latency.
3. **Real MySQL Concurrency Not Stressed**:
   * 2 live MySQL integration tests (`phase6b real lock race` and `phase7 mysql integration`) were skipped in automated pipelines due to dedicated database requirements.
   * Real-world testing under thousands of concurrent connections remains to be conducted on live Aurora/RDS clusters.
4. **No Full Browser End-to-End Testing**:
   * Automated browser frameworks (Playwright/Cypress) have not been executed against a deployed web host.
   * Project status: **Phase 10A code complete; Phase 10B (Live Acceptance on AWS) remains open**.

---

### 4. Troubleshooting Guide

Common issues encountered during workshop labs and their resolutions:

| Error Code / Symptom | Root Cause | Remediation Procedure |
| :--- | :--- | :--- |
| **`400 Quota Exceeded`** | File size exceeds the user's available storage quota. | Query quota via `/api/users/quota`; purge Trash or increase `max_quota_bytes` in the `Users` table. |
| **`403 Invalid CSRF Token`** | Mutating request lacks or supplies an invalid `X-CSRF-Token`. | Verify client invokes `GET /api/auth/csrf` to acquire a valid token before submitting forms. |
| **`400 Invalid Invitation Code`** | Token is missing, expired, or fully consumed (`uses_count >= max_uses`). | Inspect the `Invitations` table and seed a new token as outlined in Module 5.3. |
| **File stays in `UPLOADED`, never transitions to `READY`** | Worker process is inactive, or Lambda failed to write `reports/{uploadId}.json`. | Ensure `npm run worker` is running in Terminal 2; check S3 `reports/` for validation output. |
| **`503 Service Unavailable` on `/health/ready`** | MySQL server is stopped or `.env` connection parameters are invalid. | Start MySQL Server; verify `DB_HOST`, `DB_PORT`, `DB_USER`, and `DB_PASSWORD`. |
| **`SignatureDoesNotMatch` on Direct S3 Upload** | Misconfigured AWS credentials or workstation system clock skew. | Synchronize workstation clock; verify `AWS_REGION` and local AWS credentials. |
