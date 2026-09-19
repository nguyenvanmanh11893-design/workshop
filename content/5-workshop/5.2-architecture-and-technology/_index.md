---
title: "Architecture and Technology"
date: 2026-09-18T23:00:00+07:00
weight: 2
chapter: false
pre: "<b>5.2. </b>"
---



---

### 1. Overall System Architecture

The **Cloud File Manager** architecture cleanly decouples metadata management from binary object storage.

<!-- > [IMAGE 5-01: Cloud File Manager overall system architecture — showing components and verified data flows.] -->

```
+----------------------------------------------------------------------------------------------------+
|                                         CLIENT LAYER                                               |
|  React 18 + TypeScript + Vite + Ant Design 5 (Single Page Application)                             |
|  - HttpOnly Cookie (Opaque Session ID) | Custom Header: X-CSRF-Token                               |
+-----------------------------------+----------------------------------------+-----------------------+
                                    |                                        |
                 1. Init session    | 2. Receive S3 Presigned POST           | 3. Direct Upload
                    & reserve quota |    Policy & Credentials                |    (Multipart Form)
                                    v                                        v
+-----------------------------------+--------------------+   +---------------+-----------------------+
|                 BACKEND API SERVER                     |   |               AMAZON S3               |
|  Node.js (>=20) / Express.js (ES Modules)              |   |  Bucket: Private, Versioned, TLS-only |
|  - Middleware: Opaque Session, CSRF, Error, RequestId  |   |                                       |
|  - Controllers: Auth, Folder, File, Quota, Activity    |   |  Prefix Structure:                    |
|  - Services: Presigned URL, Quota, Lifecycle           |   |  - incoming/{userId}/{uploadId}       |
|  - Sequelize ORM (Migrations, Models)                  |   |  - objects/{userId}/{fileId}          |
+-----------------------------------+--------------------+   |  - reports/{uploadId}.json            |
                                    |                        +---------------+-----------------------+
                                    | 4. Record session                      |
                                    v    in UploadSessions table             | 5. s3:ObjectCreated
+-----------------------------------+--------------------+                   v    (incoming/ prefix only)
|                 DATABASE LAYER                         |   +---------------+-----------------------+
|  MySQL (InnoDB Engine)                                 |   |            AWS LAMBDA                 |
|  - Tables: Users, Sessions, Invitations                |   |  s3-file-validator (Outside VPC)      |
|  - Tables: Folders, Files, UploadSessions              |   |  - Inspect magic bytes                |
|  - Tables: Jobs, AuditLogs, ReconciliationFindings     |   |  - Output reports/{uploadId}.json     |
+-----------------------------------+--------------------+   +---------------+-----------------------+
                                    ^                                        |
                                    | 7. Fenced DB Transaction               | 6. SQS Failure
                                    |    (READY / REJECTED)                  |    Destination (DLQ)
+-----------------------------------+--------------------+                   v
|             DURABLE BACKGROUND WORKER                  |   +---------------+-----------------------+
|  Standalone process: npm run worker                    |   |            AMAZON SQS                 |
|  - Polls Jobs table: FINALIZE_UPLOAD, PURGE_FILE, ...  |<--+  validator-dlq (SSE Encrypted, 14-day)|
|  - Lease fencing: lease token & expiration timeout     |   +---------------------------------------+
|  - S3 exact-version copy (incoming -> objects)         |
|  - Promotes File to READY, settles quota & audit log   |
+--------------------------------------------------------+
```

---

### 2. Component Roles and Responsibilities

#### 2.1 Frontend 
* **Technology**: React 18, TypeScript, Vite, Ant Design 5, React Router v6.
* **Responsibilities**:
  * Delivers an intuitive drive UI featuring hierarchical folder trees, breadcrumb navigation, and modal dialogs for folder creation, renaming, and moves.
  * The file upload modal (`UploadModal.tsx`) supports drag-and-drop, client-side validation (PDF, JPEG, PNG, TXT), and size constraints (< 50 MiB).
  * Manages transparent session cookies and automatically attaches the `X-CSRF-Token` header on all mutating HTTP requests (POST, PUT, DELETE).
  * Dispatches multipart form data directly to the Amazon S3 endpoint with real-time progress indicators.

#### 2.2 Backend API Server
* **Technology**: Node.js (>=20), Express.js (ES Modules), Sequelize ORM.
* **Responsibilities**:
  * **Auth & Authorization**: Verifies registration invite codes; manages server-side Opaque Sessions stored in MySQL; strictly verifies object ownership via `req.user.id`.
  * **S3 Policy Generation**: Generates cryptographically signed S3 Presigned POST Policies with strict boundary conditions (50 MiB max, exact `incoming/{userId}/{uploadId}` destination).
  * **Quota Management**: Atomically pre-reserves user quota (`used_bytes + size <= max_bytes`) before an upload starts, mitigating over-allocation.
  * **Lifecycle APIs**: Exposes endpoints for soft deletion (`/trash`), recovery (`/restore`), permanent deletion (`/purge`), and activity audit inspection (`/activity`).

#### 2.3 Amazon S3
* **Configuration**: Private bucket, S3 Object Versioning enabled, TLS enforcement (HTTPS only), and Block Public Access enabled by default.
* **Prefix Architecture**:
  * `incoming/{userId}/{uploadId}`: Quarantine holding area for raw uploads awaiting async validation.
  * `reports/{uploadId}.json`: JSON validation results emitted by the Lambda validator.
  * `objects/{userId}/{fileId}`: Persistent production location for verified, ready files.

#### 2.4 AWS Lambda Validator
* **Function**: `lambdas/s3-file-validator` (Node.js 20/22 runtime).
* **Responsibilities**:
  * Deployed outside VPC for sub-second cold starts and zero NAT Gateway data processing charges.
  * Triggered by `s3:ObjectCreated` events on the `incoming/` prefix.
  * Inspects binary magic bytes:
    * PDF: Matches `%PDF-` signature.
    * PNG: Matches 8-byte signature `\x89PNG\r\n\x1a\n`.
    * JPEG: Matches 3-byte signature `\xFF\xD8\xFF`.
    * TXT: Verifies valid UTF-8 / ASCII encoding.
  * Emits canonical JSON reports to `reports/{uploadId}.json`.
  * On unhandled exceptions or failures, events route to **Amazon SQS Dead-Letter Queue (DLQ)**.

#### 2.5 Durable Background Worker
* **Process**: Standalone process launched via `npm run worker`.
* **Responsibilities**:
  * Scans active jobs in the MySQL `Jobs` table using database-backed lease fencing.
  * Parses validation reports from `reports/{uploadId}.json`:
    * If **PASSED**: Performs an exact-version S3 copy from `incoming/` to `objects/`, executes a fenced database transaction setting file status to `READY`, settles quota, and records audit logs.
    * If **REJECTED**: Updates file status to `REJECTED`, rolls back reserved quota, and logs security findings.
  * Executes scheduled maintenance: Expiring abandoned upload sessions (`EXPIRE_UPLOAD_SESSION`), purging trashed files after 7 days (`PURGE_FILE`), and running reconciliation routines (`RECONCILE`).

---

### 3. Security and Network Boundaries

Standardized deployment architecture in Singapore (`ap-southeast-1`):

<!-- > [IMAGE 5-02: AWS network and security architecture designed in Terraform in Singapore (ap-southeast-1).] -->

```
+----------------------------------------------------------------------------------------------------+
|                                    AWS REGION: ap-southeast-1 (Singapore)                          |
|                                                                                                    |
|  +----------------------------------------------------------------------------------------------+  |
|  | VPC (10.0.0.0/16)                                                                            |  |
|  |                                                                                              |  |
|  |  +-------------------------------------+  +-----------------------------------------------+  |  |
|  |  | Public Subnet (10.0.1.0/24)          |  | Private DB Subnets (10.0.11.0/24, 10.0.12.0/24) |  |
|  |  |                                     |  | (No Internet Gateway or NAT Gateway route)    |  |  |
|  |  |  [EC2 Instance]                    |  |                                               |  |  |
|  |  |  - Nginx Reverse Proxy (443 TLS)    |  |  [Amazon RDS MySQL]                           |  |  |
|  |  |  - Express API (127.0.0.1:3000)     |  |  - Port 3306 restricted to EC2 SG only        |  |  |
|  |  |  - Durable Worker                   |  |  - KMS Storage Encryption                    |  |  |
|  |  |  - Mandatory IMDSv2                 |  |  - 7-day automated backups                   |  |  |
|  |  |  - SSH (Port 22) closed             |  |                                               |  |  |
|  |  +------------------+------------------+  +-----------------------+-----------------------+  |  |
|  |                     |                                             |                          |  |
|  |                     +----------------------+----------------------+                          |  |
|  |                                            |                                                 |  |
|  |                     +----------------------v----------------------+                          |  |
|  |                     | S3 Gateway Endpoint (Free, Private VPC)     |                          |  |
|  |                     +----------------------+----------------------+                          |  |
|  |  +-----------------------------------------|-------------------------------------------------+  |
|                                               |                                                    |
|  +--------------------------------------------v-------------------------------------------------+  |
|  | Outside VPC Services                                                                         |  |
|  |                                                                                              |  |
|  |  [Amazon S3 Bucket]               [AWS Lambda]                   [Amazon SQS]                |  |
|  |  - Private, Versioned, CORS       - s3-file-validator            - validator-dlq (Encrypted) |  |
|  |  - incoming/, objects/, reports/  - Least privilege IAM          - 14-day message retention  |  |
|  +----------------------------------------------------------------------------------------------+  |
+----------------------------------------------------------------------------------------------------+
```

#### Core Security Invariants:
1. **Zero Public SSH**: Server administration is performed exclusively via AWS Systems Manager (SSM Session Manager), eliminating port 22 exposure.
2. **Fully Isolated Database**: RDS MySQL sits in private subnets with no route to an Internet Gateway, accepting connections on port 3306 strictly from the EC2 Security Group.
3. **S3 Gateway Endpoint**: S3 traffic flows entirely over AWS internal networks via VPC Gateway Endpoints, removing public internet exposure and egress fees.
4. **IMDSv2 Cloud Identity Protection**: EC2 enforces Instance Metadata Service v2 (`http_tokens = "required"`), thwarting SSRF attacks targeting instance IAM roles.

---

### 4. Implementation Status: Local vs. AWS Cloud

To maintain strict technical transparency, the boundaries between local implementation and AWS cloud design are clarified below:

| Dimension | Local Development Environment | AWS Production Design  |
| :--- | :--- | :--- |
| **Actual Status** | **COMPLETED & VERIFIED** | **CONFIGURED & STATICALLY TESTED** |
| **Web Client** | Served via Vite dev server (`http://localhost:5173`) | Compiled static assets served through Nginx over HTTPS |
| **API & Worker** | Node.js processes running locally on workstation | Managed systemd services running on EC2 Linux hosts |
| **Database** | Local MySQL Server on port 3306 | Multi-Subnet Amazon RDS MySQL with KMS encryption and 7-day backups |
| **Object Store** | Real S3 bucket or verified S3 mock test adapters | Amazon S3 Singapore with Versioning, TLS-only, CORS, and Gateway Endpoints |
| **Validation** | Service adapters or packaged Lambda tests | Event-triggered AWS Lambda with Amazon SQS Dead-Letter Queue |
| **Secrets Management**| Local `.env` file (gitignored, development only) | AWS SSM Parameter Store (`SecureString`), injected into memory |
