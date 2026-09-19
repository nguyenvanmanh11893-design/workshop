---
title: "Technical Highlights"
date: 2026-09-18T23:00:00+07:00
weight: 5
chapter: false
pre: "<b>5.5. </b>"
---



---

### 1. Opaque Session Security, CSRF Defense, and Invitation-Only Auth

#### 1.1 Opaque Server-Side Sessions vs. Stateless JWT
Rather than storing JSON Web Tokens (JWT) in client `localStorage` (vulnerable to XSS extraction), Cloud File Manager implements an **Opaque Session** architecture:
* **Server-Side State**: Each session is persisted in the MySQL `Sessions` table with a cryptographically random high-entropy `session_id`, `user_id`, expiration timestamp, and payload.
* **HttpOnly & SameSite Cookies**: The browser receives only an encrypted session cookie marked `HttpOnly` (preventing script access) and `SameSite=Lax` (blocking cross-site request leakage).
* **Instant Session Revocation**: Suspicious sessions can be instantly terminated server-side by deleting the corresponding database row—a capability impractical with stateless JWTs without complex distributed blacklists.

#### 1.2 Dual-Layer CSRF Mitigation
Because session cookies attach automatically to browser requests, strict CSRF defenses are enforced:
* Upon successful login, the client requests a dynamic token via `/api/auth/csrf` and retains it in React in-memory state.
* All state-changing HTTP requests (`POST`, `PUT`, `DELETE`, `PATCH`) must include the `X-CSRF-Token` header.
* The `csrf.middleware.js` module verifies this header against the active server session and validates incoming `Origin`/`Referer` headers. Unauthorized requests are rejected with HTTP `403 Forbidden`.

#### 1.3 Invitation-Only Registration Model
* The system mitigates automated bot abuse by requiring a valid invitation token (`token` in the `Invitations` table).
* Each token specifies maximum uses (`max_uses`), consumed uses (`uses_count`), and expiration dates (`expires_at`), providing granular control over user onboarding.

---

### 2. Direct-to-S3 Uploads and Fail-Closed Quota Reservation

#### 2.1 Eliminating Server I/O Bottlenecks
In traditional systems, streaming a 50 MiB file forces the API server to buffer 50 MiB in memory before relaying it to object storage:
* **Consequences**: Doubled bandwidth charges, event-loop blocking, and high crash vulnerability under concurrent traffic.
* **Cloud File Manager Solution**: The backend strictly computes a **Presigned POST Policy**. The user's browser streams binary data directly to Amazon S3. The API server never touches the binary payload.

#### 2.2 Concurrency-Safe Quota Reservation
A key design challenge of direct uploads is: *How can an application prevent users from exceeding quotas before files are registered in the database?*

Cloud File Manager implements atomic **Quota Reservation**:
1. When the client initializes an upload session (`POST /api/files/upload-session`), the system evaluates:
   $$\text{used\_bytes} + \text{reserved\_bytes} + \text{requested\_file\_size} \le \text{max\_quota\_bytes}$$
2. If valid, an `UploadSessions` record is created in `ACTIVE` state, holding the requested bytes in `reserved_bytes`.
3. Parallel uploads across multiple tabs are throttled if their combined sizes exceed available capacity.
4. Abandoned or failed upload sessions are automatically reclaimed after 15 minutes by the background reconciliation worker.

---

### 3. Durable Background Worker Architecture

The standalone background process `worker.js` provides high availability, fault tolerance, and idempotency:

#### 3.1 Database-Backed Lease Fencing
Multiple worker instances run concurrently without race conditions through **Lease Tokens**:
* A worker claims a task by generating a unique `lease_token` (UUID) and setting an expiration timestamp `lease_expires_at = NOW() + 60s`.
* When committing results to MySQL, state transitions enforce:
  `WHERE id = :jobId AND lease_token = :myToken AND lease_expires_at >= NOW()`
* If a worker stalls and its lease expires, peer workers can claim the task, while the stalled worker is safely fenced out from overwriting fresh state.

#### 3.2 Jittered Exponential Backoff
When transient network issues affect Amazon S3 interactions:
* Workers increment `attempts` and calculate the next execution delay using:
  $$\text{delay} = \min(\text{MAX\_DELAY}, \text{BASE\_DELAY} \times 2^{\text{attempts}}) \pm \text{jitter}$$
* Randomized jitter prevents Thundering Herd contention when multiple failed tasks retry concurrently.
* If attempts exceed the configured threshold (e.g., 5 retries), the task transitions to `FAILED` for operational inspection.

---

### 4. File Lifecycles and Version-Aware S3 Deletion

#### 4.1 Trash and 7-Day Delayed Purge
* Moving files to Trash avoids immediate data destruction, protecting against accidental user actions.
* The system schedules a `PURGE_FILE` task set to execute after 7 days (`due_at = NOW() + 7 days`).
* Users can restore files at any point during this window. Restoring locks the background job before updating the file record to preserve state consistency.

#### 4.2 Exact-Version S3 Deletion
On versioned S3 buckets, standard `DeleteObject` calls do not delete physical data; they simply write a *Delete Marker*, continuing storage billing.

Cloud File Manager resolves this completely:
1. The worker invokes `ListObjectVersions` to discover all historical versions, current versions, and delete markers for the key.
2. It aggregates explicit identifier pairs: `[ { Key: 'objects/...', VersionId: 'v1' }, ... ]`.
3. It issues `DeleteObjects` to permanently eradicate the physical versions from S3.
4. An inventory query verifies zero remaining versions before `used_bytes` is deducted in MySQL and the file is marked `PURGED`.

---

### 5. Self-Healing Storage Reconciliation

To combat state drift in distributed architectures, the reconciliation routine (`reconciliation.service.js`) provides automated self-healing:
* **Stalled Quota Reclamation**: Releases reserved capacity for abandoned upload sessions older than 15 minutes.
* **Storage Quota Realignment**: Periodically sums byte counts from physical records in `Files` to reset `Users.used_bytes`, eliminating cumulative drift.
* **Orphan Object Detection**: Scans S3 prefixes for objects lacking database metadata, logging findings to `ReconciliationFindings`. Automated cleanup requires explicit configuration (`ORPHAN_CLEANUP_MODE=delete`) and enforces a 24-hour observation grace period.

---

### 6. Infrastructure as Code (IaC) and CI/CD Automation

#### 6.1 Terraform Infrastructure (v1.10.5)
The infrastructure definition in `infra/terraform/` specifies:
* **Singapore VPC (`ap-southeast-1`)**: Distinct segmentation between 1 Public Subnet (EC2) and 2 Private DB Subnets (RDS MySQL, zero internet routes).
* **S3 Gateway Endpoints**: Routes S3 traffic across private AWS backbones with zero NAT egress fees.
* **HCL Security Guardrails**: The `infra/tests/check_infra.py` suite enforces 13 invariant security policies (blocking public DB exposure, plaintext credentials, IMDSv1, SSH port 22, and unauthorized data mutations).

#### 6.2 Manual-Gated CI/CD Delivery
Defined in `.github/workflows/`:
* **CI Workflow (`ci.yml`)**: Triggered on Pull Requests: runs ESLint, 82 backend tests, 6 frontend tests, dependency audits (`npm audit`), Terraform validation, and SHA-256 artifact packaging.
* **CD Workflow (`cd.yml`)**: Authenticates via AWS GitHub OIDC (zero static credentials), requires manual approval gates, verifies artifact checksums, and orchestrates deployment via AWS Systems Manager.
