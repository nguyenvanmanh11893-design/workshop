---
title: "Hands-on Exercises"
date: 2026-09-18T23:00:00+07:00
weight: 4
chapter: false
pre: "<b>5.4. </b>"
---



---

### Step-by-Step Hands-on Roadmap

Each exercise includes: **Objective**, **Prerequisites**, **Step-by-Step Commands**, **Expected Outcome**, and **Verification Procedure**.

* **[5.4.1. Bootstrap and Verify Services](5.4.1-bootstrap-services/)**:
  * Launch Frontend and Backend API development servers (`npm run dev`).
  * Run the standalone background Worker process (`npm run worker`).
  * Test liveness and readiness health probes: `/health/live` and `/health/ready`.
* **[5.4.2. Authentication and Direct S3 Upload](5.4.2-auth-and-direct-upload/)**:
  * Register a new user account with an invitation token.
  * Log in, inspect the Opaque Session Cookie, and retrieve the CSRF token.
  * Create custom directories and navigate folder hierarchies.
  * Initialize an upload session, evaluate quota reservation, and stream files directly to the S3 `incoming/` prefix via Presigned POST Policies.
* **[5.4.3. Validation with Lambda and Worker Finalization](5.4.3-validation-and-finalization/)**:
  * Trace `s3:ObjectCreated` notifications invoking the `s3-file-validator` Lambda function.
  * Inspect binary magic bytes and generate canonical JSON reports at `reports/{uploadId}.json`.
  * Observe the Durable Worker claiming jobs via lease fencing, safely copying objects to `objects/`, updating file states to `READY`, and settling quotas.
* **[5.4.4. File Lifecycle, Trash, and Reconciliation](5.4.4-lifecycle-and-reconciliation/)**:
  * Generate time-bounded (15 min) S3 Presigned GET URLs for secure downloads.
  * Move files to Trash and schedule a 7-day delayed purge.
  * Restore files back to the active drive before the purge executes.
  * Trigger immediate purge requests (`PURGE_PENDING` -> HTTP 202) and execute version-aware S3 batch deletions targeting exact `(Key, VersionId)` tuples.
  * Run the reconciliation scanner to detect orphaned storage and self-heal quota discrepancies.
