---
title: "Proposal"
date: 2026-09-18T23:00:00+07:00
weight: 2
chapter: false
pre: "<b>2. </b>"
---

# Cloud-Native Personal File Storage and Management System 

---

### 1. Executive Summary

**Cloud File Manager** is a personal cloud storage and file management solution (inspired by Google Drive), architected to deliver an intuitive, secure, and resource-optimized user experience.

The system combines the responsiveness of modern web technologies (**React 18, TypeScript, Vite, Ant Design**) with a service-oriented backend (**Node.js, Express, Sequelize, MySQL**) and production-grade **Amazon Web Services (AWS)** primitives—including **Amazon S3, AWS Lambda, Amazon SQS, Amazon CloudWatch**, alongside Infrastructure as Code (**Terraform**).

The design specifically tackles traditional performance and security bottlenecks through:
* **Direct-to-S3 Uploads**: Files stream directly from client browsers to S3 via Presigned POST Policies, completely eliminating I/O bottlenecks and memory exhaustion on the API server.
* **Rigorous Quota Reservation**: Atomic pre-allocation and validation of storage limits before uploads commence.
* **Outside-VPC Lambda Validator**: Fast, cost-efficient binary content validation (magic bytes inspection) triggered by S3 events.
* **Durable Background Worker**: An autonomous processing pipeline equipped with database-backed lease fencing, jittered exponential backoff, and idempotent job transitions.
* **Comprehensive Lifecycle Management**: Multi-stage Trash and Restore workflows, version-aware S3 deletion, and automated reconciliation routines to eliminate orphaned objects and state drift.

---

### 2. Problem Statement & Context

#### 2.1 Real-World Challenges
In conventional file storage architectures, incoming binary streams pass directly through the backend API server before being persisted to local disks or relayed to object storage. This approach creates severe operational liabilities:
1. **Server I/O & Memory Bottlenecks**: High concurrent uploads of medium-to-large files (e.g., 50 MiB) quickly saturate network interfaces and exhaust server RAM.
2. **Double Bandwidth Costs (Double Egress/Ingress)**: Streaming data first to the API host and subsequently to object storage incurs redundant network hops and elevated latency.
3. **Malicious Files & Extension Spoofing**: Solely relying on file extensions (`.pdf`, `.png`) or client-declared `Content-Type` headers is vulnerable to bypass without deep binary magic bytes inspection.
4. **State Drift Between Database and Storage**: Network drops or process crashes during upload or deletion create inconsistencies between database records and underlying S3 objects, resulting in orphaned storage or corrupted quotas.
5. **Session Security & Multi-tenant Data Leakage**: In multi-user setups, weak session handling or insufficient ownership verification risks unauthorized cross-tenant data access.

#### 2.2 Project Objectives
* Build an intuitive, responsive personal drive web interface with hierarchical directory navigation.
* Implement a secure Direct-to-S3 upload mechanism without overloading backend server resources.
* Guarantee data consistency and transactional integrity between MySQL and Amazon S3 under all failure scenarios (network interruptions, process crashes).
* Enforce multi-layered security boundaries: Invitation-only registration, Opaque Sessions stored in HttpOnly cookies, CSRF mitigation, and strict per-user tenant isolation.

#### 2.3 Target Audience & 6-Week Scope
* **Target Audience**: Individuals, research groups, or engineers requiring a controlled, secure environment for document and asset storage with strict quota enforcement.
* **6-Week Scope**: Complete the core application architecture, secure authentication, Direct S3 upload, durable background workers, file lifecycle operations, comprehensive verification, CloudWatch monitoring, and hands-on workshop modules. Terraform infrastructure definitions and CI/CD pipelines are packaged for local verification and configuration readiness.

---

### 3. Proposed Solution Architecture

The system implements a 3-tier distributed architecture integrated with AWS serverless and queuing services:

<!-- > [IMAGE 2-01: Proposed solution architecture for Cloud File Manager — showing Frontend, Backend API, MySQL, Amazon S3, Lambda Validator, and Durable Worker.] -->

```
+-----------------------------------------------------------------------------------+
|                                  CLIENT LAYER                                     |
|  React 18 + TypeScript + Vite + Ant Design 5 (SPA)                                |
|  - HttpOnly Cookie (Opaque Session)                                               |
|  - X-CSRF-Token Header                                                            |
+-------------------+---------------------------------------+-----------------------+
                    |                                       |
       1. Request   | 2. Presigned POST                     | 3. Direct Upload
          Session   |    Policy + Quota                     |    (Multipart form)
                    v                                       v
+-------------------+-------------------+   +---------------+-----------------------+
|            BACKEND API                |   |               AMAZON S3               |
|  Node.js / Express.js / Sequelize     |   |  Bucket: Private, Versioned, TLS-only |
|  - Auth & Invitation verification     |   |                                       |
|  - Quota reservation (ACTIVE)         |   |  Prefixes:                            |
|  - Folder & File metadata management  |   |  - incoming/{userId}/{uploadId}       |
|  - Generate S3 Presigned POST         |   |  - objects/{userId}/{fileId}          |
+-------------------+-------------------+   |  - reports/{uploadId}.json            |
                    |                       +---------------+-----------------------+
                    | 4. Metadata sync                      |
                    v                                       | 5. s3:ObjectCreated
+-------------------+-------------------+                   v    (incoming/ only)
|              DATABASE                 |   +---------------+-----------------------+
|  MySQL (InnoDB Engine)                |   |            AWS LAMBDA                 |
|  - Users, Sessions, Invitations       |   |  s3-file-validator (Outside VPC)      |
|  - Folders, Files, Jobs, AuditLogs    |   |  - Check magic bytes (PDF, Image, TXT)|
|  - ReconciliationFindings             |   |  - Write JSON validation report       |
+-------------------+-------------------+   +---------------+-----------------------+
                    ^                                       |
                    | 7. Fenced DB Transaction              | 6. SQS Dead-Letter
                    |    (READY / REJECTED)                 |    Queue on failure
                    |                                       v
+-------------------+-------------------+   +---------------+-----------------------+
|           DURABLE WORKER              |   |            AMAZON SQS                 |
|  Separate Node.js Process             |   |  validator-dlq (Encrypted, 14-day)    |
|  - Claim job via Lease & Fencing      |<--+---------------------------------------+
|  - Exact-version S3 copy/delete       |
|  - Exponential Jittered Retry         |
|  - Quota settlement & Reconciliation  |
+---------------------------------------+
```

#### Core Architectural Components:
1. **Frontend Client (React SPA)**: Manages UI rendering, hierarchical folder navigation, auth flows, and direct-to-S3 multi-part file uploads with real-time progress indicators.
2. **Backend API (Express.js)**: Handles business logic, validates sessions, issues constrained S3 Presigned POST policies (50 MiB limit, strict `incoming/` prefix, specific MIME types), and atomically reserves quota.
3. **Amazon S3 Storage**: Hierarchical object store organized into designated prefixes:
   * `incoming/`: Quarantine zone for raw user uploads awaiting asynchronous validation.
   * `objects/`: Primary repository for verified, production-ready files.
   * `reports/`: JSON validation outputs generated by AWS Lambda.
4. **AWS Lambda Validator**: A serverless function triggered by `s3:ObjectCreated` events in `incoming/`. It inspects file magic bytes (PDF `%PDF-`, PNG `\x89PNG`, JPEG `\xFF\xD8\xFF`, valid UTF-8 text) and publishes reports to `reports/`.
5. **Durable Background Worker**: An autonomous Node.js process executing background jobs from the `Jobs` table, utilizing lease fencing, evaluating Lambda reports, moving verified objects to `objects/`, updating statuses to `READY`, and logging audit trails.
6. **Amazon SQS (Dead-Letter Queue)**: Captures unprocessable or failing Lambda events to ensure zero event loss and support operational investigation.

---

### 4. Technology Stack & Design Decisions

| Technology | Role in System | Key Technical Rationale |
| :--- | :--- | :--- |
| **React 18 + TypeScript + Vite** | Frontend SPA Client | Strong compile-time type checking, fast HMR build cycles, and standardized Ant Design 5 UI components. |
| **Express.js (Node >=20)** | RESTful API Server | Modular layered architecture (Routes, Controllers, Services), Helmet HTTP headers, centralized authentication middleware. |
| **MySQL + Sequelize ORM** | Relational Database | Enforces relational integrity (Foreign Keys, Indexes), provides ACID transaction support for status transitions and quota consistency. |
| **Amazon S3** | Primary Object Store | Enabled S3 Object Versioning, Block Public Access enabled by default, restricted access through time-bounded (15 min) presigned URLs. |
| **AWS Lambda** | File Content Validator | Positioned outside VPC for minimal cold starts and cost efficiency; granted least-privilege access to read `incoming/` and write `reports/`. |
| **Amazon SQS** | Dead-Letter Queue | Retains failed upload validation messages with server-side encryption (SSE) and a 14-day retention window. |
| **Terraform (v1.10.5)** | Cloud Infrastructure as Code | Structured AWS modular configuration in Singapore (`ap-southeast-1`) protected by invariant security controls (no public DB, IMDSv2, TLS enforcement). |
| **GitHub Actions** | Automated CI/CD Pipeline | Multi-stage verification: Linting, Unit tests, Mock DB tests, Terraform static analysis, and cryptographic artifact packaging (SHA-256). |

---

### 5. Implementation Roadmap (6-Week Program)
 
The project implementation roadmap is aligned with the 6-week training schedule:

* **Week 1: Research & Foundational Cloud Setup**
  * Practical orientation with AWS Console, AWS CLI, IAM security models, and EC2 compute instances.
  * Define storage system requirements, draft end-to-end architecture, and design relational schemas (Users, Sessions, Folders, Files).
* **Week 2: Authentication & Hierarchical Folder Management**
  * Implement database-backed Opaque Session management via HttpOnly cookies.
  * Build dynamic CSRF validation (`X-CSRF-Token`) and invitation-only account provisioning.
  * Develop hierarchical folder CRUD APIs and integrated React Drive UI.
* **Week 3: Direct-to-S3 Upload Pipeline & Lambda Validator**
  * Architect Presigned POST generation coupled with atomic quota reservations.
  * Develop the `s3-file-validator` Lambda function inspecting file magic bytes across supported types.
  * Connect S3 Event Notifications to invoke Lambda and write canonical JSON reports to `reports/{uploadId}.json`.
* **Week 4: Durable Worker & File Lifecycle Management**
  * Construct autonomous background worker processes with database lease fencing and jittered exponential retries.
  * Implement Trash, Restore, delayed permanent purge (7-day retention), and version-aware S3 batch deletion (`(Key, VersionId)` pairs).
  * Build automated reconciliation routines to identify orphaned objects and self-heal storage quota drift.
* **Week 5: Comprehensive Verification, Terraform IaC & Workshop Documentation**
  * Execute automated test suites: 82 backend tests, 6 frontend tests, and Terraform HCL security checks.
  * Finalize modular Terraform infrastructure templates and CI/CD pipelines with manual approval gates.
  * Compile comprehensive workshop lab documentation and the final technical project report.
* **Week 6: CloudWatch Operational Monitoring, Project Packaging & Final Report**
  * Configure Amazon CloudWatch Logs, custom metrics, and a 75 USD Billing Alarm integrated with SNS notifications.
  * Verify fault tolerance, SQS DLQ exception handling, and database lease fencing concurrency under failure scenarios.
  * Finalize the comprehensive hands-on Workshop guide for Cloud File Manager, package all source artifacts, and complete the final internship report.

---

### 6. Infrastructure Cost Estimation



| Service Component | Proposed Configuration | Estimated Cost / Month (USD) | Notes & Assumptions |
| :--- | :--- | :--- | :--- |
| **Amazon EC2** | 1 x `t4g.small` (or `t3.micro`), 20 GB gp3 EBS Encrypted | ~$15.00 - $18.00 | Hosts Nginx reverse proxy, Node.js API, and the background worker process. |
| **Amazon RDS (MySQL)** | 1 x `db.t4g.micro`, Single-AZ, 20 GB gp3 Storage | ~$17.00 - $20.00 | Primary database, 7-day automated backup retention, isolated in private subnets. |
| **Amazon S3** | Standard Storage (~10 - 20 GB), PUT/GET requests | ~$0.50 - $1.50 | Binary object storage and API call operations; Versioning enabled. |
| **AWS Lambda** | 128 MB RAM, ~10,000 invocations / month | $0.00 (Free Tier) | Falls well within the AWS Free Tier threshold (1M requests/month). |
| **Amazon SQS & CloudWatch** | 1 DLQ Queue, CloudWatch Logs & Metrics | ~$1.00 - $3.00 | Structured JSON logging, host metrics (RAM/Disk), and operational alarms. |
| **Data Transfer & Gateway EP** | S3 Gateway Endpoint (Free), Low Egress | ~$1.00 - $2.00 | In-region S3 traffic routed over S3 Gateway Endpoints incurs no data transfer fees. |
| **Total Estimated Cost** | | **~$35.00 - ~$45.00 / month** | *Budget alert threshold set at 75.00 USD/month via CloudWatch Billing Alarms.* |

*Note:* The system is pre-configured with cost alerts via CloudWatch Alarms and an SNS Topic. For production deployment, detailed cost projections from the **AWS Pricing Calculator** should be incorporated based on actual traffic requirements.

---

### 7. Risk Assessment & Mitigation Strategies

| Technical Risk | Impact | Probability | Engineered Mitigation Strategy |
| :--- | :---: | :---: | :--- |
| **Malicious or Extension-Spoofed Uploads** | High | Medium | Independent magic bytes validation via Lambda prior to promoting files to `objects/`. Invalid files are rejected immediately and reserved quota is rolled back. |
| **Abandoned Uploads Leading to Ghost Quotas** | Medium | High | Enforce a strict 15-minute TTL on upload sessions. The automated Reconciliation worker regularly purges expired active sessions and reclaims quota. |
| **Versioning Conflicts During S3 Deletions** | High | Low | Implement explicit version-aware batch deletion targeting exact `(Key, VersionId)` tuples (including delete markers), verified via inventory listing prior to setting DB status to `PURGED`. |
| **Worker Process Crash During Job Execution** | High | Low | Database-backed lease fencing (`lease_token`, `lease_expires_at`). In the event of a worker failure, the lock expires and a peer worker safely resumes processing. |
| **Cross-Site Request Forgery (CSRF)** | Medium | Medium | All state-mutating requests (POST, PUT, DELETE) require a cryptographically validated `X-CSRF-Token` header matching the active server session. |
| **Uncontrolled Cloud Infrastructure Costs** | Medium | Low | Configure automated 75 USD CloudWatch Billing Alarms integrated with SNS alerts; S3 Lifecycle rules automatically expire temporary incoming files after 30 days. |

---

### 8. Expected Outcomes

1. **Architectural & Technical Rigor**:
   * Validates a modern tiered cloud storage design, showcasing the performance gains of Direct S3 uploads compared to traditional proxy routing.
   * Establishes a fault-tolerant asynchronous processing pipeline with lease fencing and self-healing data reconciliation.
2. **User Experience & Functionality**:
   * Delivers a polished, responsive web application enabling seamless directory management, direct uploading, audit tracking, and trash management.
3. **Educational & Workshop Impact**:
   * Serves as a reference implementation for engineers mastering full-stack cloud development, advanced AWS service integration, Terraform IaC, and CI/CD pipelines.
