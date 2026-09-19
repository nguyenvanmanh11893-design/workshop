---
title: "Workshop Overview"
date: 2026-09-18T23:00:00+07:00
weight: 1
chapter: false
pre: "<b>5.1. </b>"
---



---

### 1. Context and Technical Problem Statement

In the digital era, storing and managing personal files in the cloud is an indispensable requirement. However, building an architecture similar to Google Drive introduces significant engineering and operational challenges:
* **Server Overhead from Large File Ingestion**: Streaming large binary payloads through traditional API gateways quickly exhausts server RAM and creates network I/O contention under high concurrency.
* **Malicious Files and Extension Spoofing**: Applications cannot rely on file extensions (`.jpg`, `.pdf`) or client-declared `Content-Type` headers alone. An independent, automated mechanism is required to inspect binary magic bytes before files are promoted to persistent storage.
* **Storage and Database State Drift**: In distributed environments, network drops or application crashes during uploads, deletions, or moves cause inconsistencies between database metadata and physical storage on Amazon S3, leading to orphaned files or incorrect user quota accounting.
* **Multi-tenant Security and Session Isolation**: Applications must enforce strict data boundaries to prevent unauthorized cross-tenant access, mitigate Session Hijacking, and defend against Cross-Site Request Forgery (CSRF).

The **Cloud File Manager** project provides a comprehensive solution by pairing a modern React web application with a Node.js/Express backend API, MySQL relational database, and Amazon S3 object storage verified via serverless AWS Lambda functions.

---

### 2. Learning Objectives

Upon completing this workshop, participants will master the following technical competencies:
1. **Direct-to-S3 Upload Patterns**: Understand how Presigned POST Policies work, how the backend generates cryptographically signed upload policies, and how clients upload files directly to S3 without burdening API servers.
2. **Quota Reservation & Concurrency Controls**: Learn how the application atomically reserves quota ahead of time, prevents race conditions during concurrent uploads, and rolls back reserved bytes on failures or expiration.
3. **Serverless Event-Driven Architectures on AWS**: Implement an AWS Lambda function triggered by S3 `s3:ObjectCreated` events to inspect binary magic bytes and emit structured JSON reports.
4. **Durable Background Worker Patterns**: Master database-backed job queuing, lease fencing for concurrency control, jittered exponential backoff for retries, and idempotent job execution.
5. **Lifecycle Management & Data Reconciliation**: Implement Trash, Restore, delayed permanent purges, version-aware S3 batch deletion (`(Key, VersionId)` tuples), and automated reconciliation routines to self-heal orphaned objects and quota drift.
6. **Advanced Application Security**: Establish database-backed Opaque Sessions delivered via HttpOnly cookies, dynamic CSRF token validation, and invitation-only user registration.
7. **Infrastructure as Code (IaC) & CI/CD Fundamentals**: Analyze Terraform configurations (isolated subnets, S3 Gateway Endpoints, CloudWatch monitoring) and multi-stage verification pipelines in GitHub Actions.

---

### 3. Target Deliverables

Participants will produce the following tangible outputs:
* **Fully Functional Local Environment**: Concurrently running React Frontend, Express API, MySQL database, and background Worker.
* **Seeded Initial Dataset**: Provisioned administrator account, invitation tokens, hierarchical folder structures, and successfully verified file uploads.
* **Complete Test Verification**: Run and pass 82 backend unit/integration tests and 6 frontend test suites with 100% pass rates.
* **Architectural Trade-off Analysis**: Clear rationale for design decisions regarding security, performance, cost, and reliability.

---

### 4. Workshop Scope & 6-Week Roadmap Alignment

The workshop modules align directly with the skills accumulated throughout the 6-week training roadmap:

| Week | Focus Area | Mapping to Cloud File Manager Workshop | Workshop Module |
| :---: | :--- | :--- | :---: |
| **Week 1** | AWS Fundamentals, Console, CLI, IAM, EC2 | Architecture design, security models, and cloud service responsibilities. | **Modules 5.1 & 5.2** |
| **Week 2** | Cloud Storage, Databases, and Networking | MySQL setup, Sequelize migrations, Opaque Sessions, HttpOnly cookies, and folder hierarchy. | **Modules 5.3 & 5.4.1** |
| **Week 3** | Serverless, File Ingestion, and S3 Integration | Direct S3 Upload (Presigned POST), Quota Reservation, and AWS Lambda magic bytes validation. | **Modules 5.4.2 & 5.4.3** |
| **Week 4** | Asynchronous Processing, Queues, and Lifecycles | Durable Worker, Lease Fencing, Retries, Trash, Version-aware S3 Purge, and Reconciliation. | **Modules 5.4.4 & 5.5** |
| **Week 5** | Testing, IaC Automation & CI/CD | Comprehensive automated test execution, Terraform configuration analysis, CI/CD pipeline verification. | **Modules 5.5 & 5.6** |
| **Week 6** | Operational Monitoring, Risk Management & Wrap-up | CloudWatch Logs/Metrics/Alarms setup, 75 USD Billing Alarm with SNS, artifact packaging, and resource cleanup. | **Modules 5.6 & 5.7** |
