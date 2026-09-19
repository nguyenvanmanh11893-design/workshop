---
title: "Validation and Worker Finalization"
date: 2026-09-18T23:00:00+07:00
weight: 3
chapter: false
pre: "<b>5.4.3 </b>"
---



---

### 1. Objective
* Understand S3 `s3:ObjectCreated` event notification mechanics triggering AWS Lambda functions.
* Master binary magic bytes inspection algorithms in Lambda to prevent file extension spoofing and executable uploads.
* Observe the Durable Worker scanning the `Jobs` table, claiming tasks via lease fencing, and executing atomic database transactions to promote files to `READY`.

---

### 2. Asynchronous Processing Sequence Diagram

<!-- > [IMAGE 5-05: Sequence diagram of Direct Upload, Lambda Validation, and Worker Finalization — illustrating interactions between S3, Lambda, SQS, MySQL, and Worker.] -->

```
[Amazon S3]                  [AWS Lambda]                 [MySQL DB]             [Durable Worker]
     |                            |                           |                          |
     | (Stored in incoming/)      |                           |                          |
     | 1. S3 Event Notification   |                           |                          |
     |--------------------------->|                           |                          |
     |                            | (Read first 512 bytes)    |                          |
     |                            | (Validate Magic Bytes)    |                          |
     | 2. Write JSON Report       |                           |                          |
     |    reports/{uploadId}.json |                           |                          |
     |<---------------------------|                           |                          |
     |                            | (Push SQS DLQ on error)   |                          |
     |                            +-------------------------->| (Create FINALIZE Job)    |
     |                                                        |                          |
     |                                                        | 3. Poll Jobs (Due)       |
     |                                                        |<-------------------------|
     |                                                        | 4. Fenced Lease Claim    |
     |                                                        |    (Update lease_token)  |
     |                                                        |------------------------->|
     |                                                                                   |
     | 5. Read reports/{uploadId}.json                                                   |
     |<----------------------------------------------------------------------------------|
     |                                                                                   |
     | 6. S3 CopyObject (Exact version from incoming/ to objects/)                       |
     |<----------------------------------------------------------------------------------|
     |                                                                                   |
     |                                                        | 7. Fenced DB Transaction:|
     |                                                        |    - Insert File row     |
     |                                                        |      (status = READY)    |
     |                                                        |    - Settle User Quota   |
     |                                                        |    - Insert AuditLog row |
     |                                                        |    - Complete Job        |
     |                                                        |<-------------------------|
```

---

### 3. Lambda Content Validation Algorithm

The Lambda function in `lambdas/s3-file-validator/index.js` inspects leading binary headers:

```javascript
// Binary signature extraction (Magic Bytes Check)
function detectMimeType(buffer) {
  // PDF: %PDF- (0x25 0x50 0x44 0x46 0x2D)
  if (buffer.length >= 5 && buffer.toString('ascii', 0, 5) === '%PDF-') {
    return 'application/pdf';
  }
  // PNG: \x89PNG\r\n\x1a\n (0x89 0x50 0x4E 0x47 0x0D 0x0A 0x1A 0x0A)
  if (buffer.length >= 8 &&
      buffer[0] === 0x89 && buffer[1] === 0x50 && 
      buffer[2] === 0x4e && buffer[3] === 0x47) {
    return 'image/png';
  }
  // JPEG: \xFF\xD8\xFF
  if (buffer.length >= 3 && 
      buffer[0] === 0xff && buffer[1] === 0xd8 && buffer[2] === 0xff) {
    return 'image/jpeg';
  }
  // TXT: Valid UTF-8 / ASCII code points
  if (isValidUtf8Text(buffer)) {
    return 'text/plain';
  }
  return 'application/octet-stream';
}
```

#### Canonical JSON Validation Report (`reports/{uploadId}.json`):
```json
{
  "report_id": "rep_9f83a21b",
  "upload_id": "upl_4a12c8e0",
  "user_id": 1,
  "s3_key": "incoming/1/upl_4a12c8e0",
  "s3_version_id": "3/L4kqtJlcpXroDTDmJ+rmSpXd3dIbrHY",
  "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "detected_mime_type": "application/pdf",
  "verdict": "PASSED",
  "validator_version": "1.0.0",
  "validated_at": "2026-09-18T23:05:00.000Z"
}
```

---

### 4. Durable Worker Finalization Protocol

The standalone background process `worker.js` guarantees consistency across 4 steps:

#### Step 1: Database-Backed Lease Fencing
The worker executes an atomic SQL update against the `Jobs` table to prevent race conditions across parallel worker nodes:
```sql
UPDATE Jobs 
SET lease_token = 'worker-node-1-uuid', 
    lease_expires_at = DATE_ADD(NOW(), INTERVAL 60 SECOND),
    attempts = attempts + 1
WHERE id = 101 
  AND status = 'PENDING'
  AND (lease_expires_at IS NULL OR lease_expires_at < NOW());
```

#### Step 2: Evaluate Report and Execute Exact-Version S3 Copy
The worker loads the JSON report from S3. If `verdict === 'PASSED'`, it invokes S3 `CopyObject`:
* **Source**: `incoming/1/upl_4a12c8e0` (with exact source `VersionId`).
* **Destination**: `objects/1/file_77b31a29`.
* Stores the newly produced destination `VersionId`.

#### Step 3: Fenced Database Transaction
If the S3 copy succeeds, the worker wraps state mutations in a MySQL transaction:
1. Inserts a record into `Files` with `status = 'READY'` and target S3 version metadata.
2. Updates `UploadSessions` status to `COMPLETED`.
3. Settles user quota: `Users.used_bytes = Users.used_bytes + file_size`.
4. Appends an audit trail entry into `AuditLogs` with action `FILE_UPLOAD`.
5. Marks the job in `Jobs` as `COMPLETED`.

#### Step 4: Rejection Handling
If `verdict === 'REJECTED'` (e.g., an executable payload masked as a `.pdf`):
* The binary object is never copied to `objects/`.
* The worker transitions `UploadSessions` to `FAILED`.
* The reserved quota is rolled back without deducting user capacity.

---

### 5. Verification Steps

#### Check 1: Worker Execution Logs
In Terminal 2 (running `npm run worker`), verify the structured logs:
```text
[worker] Claimed job ID: 101, type: FINALIZE_UPLOAD, lease_token: worker-node-1-uuid
[worker] Read validation report for upload upl_4a12c8e0: PASSED (application/pdf)
[worker] Copied object to objects/1/file_77b31a29 with destination VersionId: abc123xyz
[worker] Database transaction committed. File ID: 45 is now READY.
[worker] Quota updated: +2097152 bytes for user 1.
[worker] Job 101 marked as COMPLETED.
```

#### Check 2: Verify UI Update in Browser
Return to `http://localhost:5173/drive`:
* The file list automatically displays `report.pdf` with the corresponding PDF icon.
* File size, upload date, and owner attributes populate accurately.
* The quota usage indicator in the sidebar increments by the file size.
