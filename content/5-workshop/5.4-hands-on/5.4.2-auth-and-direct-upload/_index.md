---
title: "Authentication and Direct S3 Upload"
date: 2026-09-18T23:00:00+07:00
weight: 2
chapter: false
pre: "<b>5.4.2 </b>"
---



---

### 1. Objective
* Register a new user account securely via an invitation code (`WORKSHOP-INVITE-2026`).
* Establish an authenticated session using Opaque Session Cookies and extract the CSRF token.
* Create a hierarchical directory structure within the Drive web interface.
* Execute direct-to-S3 uploads from the browser using Presigned POST Policies and verify storage quota reservations.

---

### 2. Prerequisites
* Both development processes (`npm run dev` and `npm run worker`) running concurrently.
* The invitation code `WORKSHOP-INVITE-2026` seeded in the database as completed in **[Module 5.3](../../5.3-prerequisites/)**.
* A sample test file (PDF, PNG, or JPEG) under 50 MiB ready on your machine.

---

### 3. Execution Steps

#### Step 1: User Account Registration
1. Navigate to `http://localhost:5173/register`.
2. Input the registration details:
   * **Display Name**: `John Doe`
   * **Email**: `user1@example.com`
   * **Password**: `Password@123`
   * **Invitation Token**: `WORKSHOP-INVITE-2026`
3. Click **Register**.

*System Flow*: The API checks the token in `Invitations`. If valid and unused (`uses_count < max_uses`), it hashes the password with `bcryptjs` (salt rounds = 10), creates a `Users` record, increments `uses_count`, and provisions a default 1 GiB quota (`1073741824 bytes`).

<!-- > [IMAGE 5-03: Drive login and file management interface — displaying folder trees, file grids, and storage quota widgets.] -->

#### Step 2: Log In and Inspect Session State
1. Visit `http://localhost:5173/login`, input credentials, and click **Log In**.
2. Upon authentication, you will be redirected to the main `/drive` workspace.

*Security Analysis in DevTools (F12 -> Application -> Cookies):*
* The session cookie `connect.sid` is flagged with `HttpOnly` and `SameSite=Lax`, protecting it against XSS exfiltration.
* The client invokes `GET /api/auth/csrf` to retrieve an anti-CSRF token, attaching it as `X-CSRF-Token` in all subsequent mutating HTTP requests.

#### Step 3: Create Hierarchical Directories
1. Within the Drive interface, click **+ New Folder**.
2. Name the folder `Workshop Documents` and click **Confirm**.
3. Double-click the folder to navigate into it. The breadcrumb trail updates to `Drive > Workshop Documents`.

#### Step 4: Initialize Direct S3 Upload Session
1. Click **Upload** to open the file upload modal.
2. Select or drag-and-drop your test file (e.g., `report.pdf`, ~2 MiB).

<!-- > [IMAGE 5-04: File upload modal and Direct S3 Upload progress bar — illustrating browser streaming directly to the S3 endpoint.] -->

---

### 4. Technical Upload Flow Analysis

The direct upload executes across 3 coordinated phases:

```
[Client Browser]                       [Backend API]                    [Amazon S3]
        |                                     |                               |
        | 1. POST /api/files/upload-session   |                               |
        |    { filename, size, mime, folder } |                               |
        |------------------------------------>|                               |
        |                                     | (Evaluate Quota: used + size) |
        |                                     | (Create UploadSession: ACTIVE)|
        |                                     | (Generate Presigned POST Data)|
        | 2. Return Presigned POST Policy     |                               |
        |<------------------------------------|                               |
        |                                                                     |
        | 3. POST multipart/form-data (directly to S3)                        |
        |    [Fields: key, policy, X-Amz-Signature, file payload]            |
        |-------------------------------------------------------------------->|
        |                                                                     |
        | 4. HTTP 204 No Content (S3 ingest success)                          |
        |<--------------------------------------------------------------------|
```

1. **Phase 1: Request Upload Session**:
   * The client dispatches metadata to `POST /api/files/upload-session`.
   * The API verifies quota: If `used_bytes + reserved_bytes + size > max_quota`, it responds with `400 Quota Exceeded`.
   * If valid, an `UploadSessions` row is created with status `ACTIVE` and an S3 Presigned POST Policy is generated (15 min TTL, enforcing key prefix `incoming/${userId}/${uploadId}`).
2. **Phase 2: Direct-to-S3 Multipart Upload**:
   * The browser receives the S3 endpoint URL and required cryptographic form fields.
   * The client constructs a `FormData` payload and transmits the binary stream directly to S3 via `XMLHttpRequest`, reflecting real-time progress.
3. **Phase 3: S3 Confirmation**:
   * Amazon S3 verifies the signature, stores the object at `incoming/${userId}/${uploadId}`, and returns HTTP `204 No Content`.

---

### 5. Verification Steps

#### Check 1: Quota Reservation Record in MySQL
Inspect the active session record in MySQL:

```sql
SELECT id, user_id, filename, file_size, status, created_at 
FROM UploadSessions 
ORDER BY id DESC LIMIT 1;
```
*Expected Result:*
The latest row displays status `ACTIVE`, verifying that storage capacity was reserved atomically.

#### Check 2: Raw Uploaded Object on Amazon S3
Inspect the quarantined `incoming/` prefix using the AWS CLI:

```bash
aws s3 ls s3://example-cloud-file-manager-bucket-local/incoming/
```
*Expected Result:*
The raw object appears under `incoming/{userId}/{uploadId}`, ready for Lambda validation.
