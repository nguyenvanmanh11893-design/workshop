---
title: "Lifecycle, Trash, and Reconciliation"
date: 2026-09-18T23:00:00+07:00
weight: 4
chapter: false
pre: "<b>5.4.4 </b>"
---



---

### 1. Objective
* Securely download stored objects via time-bounded (15 min) S3 Presigned GET URLs rather than public bucket endpoints.
* Move files to Trash and understand the 7-day delayed purge lifecycle.
* Restore files back to active status and execute explicit permanent deletions.
* Master version-aware S3 batch deletions targeting exact `(Key, VersionId)` tuples and automated data reconciliation routines.

---

### 2. Step-by-Step Hands-on Workflow

#### Step 1: Secure Direct S3 File Download
1. Locate `report.pdf` in `/drive` and click the **Actions** menu (three dots).
2. Click **Download**.

*Technical Flow:*
* The browser issues `GET /api/files/45/download`.
* The API verifies ownership (`req.user.id === file.user_id`) and confirms the file is `READY`.
* Using the AWS SDK, the backend generates an S3 **Presigned GET URL** with a 15-minute TTL, embedding the header `response-content-disposition: attachment; filename="report.pdf"`.
* The browser streams the object directly from S3, consuming zero API server bandwidth.

#### Step 2: Move File to Trash
1. Click the file options menu for `report.pdf` and select **Move to Trash**.
2. An Ant Design notification confirms: *"File moved to trash"*.
3. The item leaves the Drive explorer and appears in **Trash** (`http://localhost:5173/trash`).

<!-- > [IMAGE 5-06: Trash interface displaying deleted files, automated purge timelines, and action buttons.] -->

*Business Logic & Quota Behavior:*
* File status updates to `status = 'TRASHED'` and `trashed_at = NOW()`.
* **Important Quota Rule**: Trashed files **continue to count toward user quota** (`used_bytes` remains unchanged) to prevent abuse of the trash bin as free storage.
* The system enqueues a background task in `Jobs`:
  * `job_type`: `PURGE_FILE`
  * `due_at`: `DATE_ADD(NOW(), INTERVAL 7 DAY)` (7-day delayed purge).

#### Step 3: Restore File from Trash
1. Navigate to `/trash`.
2. Click **Restore** next to `report.pdf`.
3. The file instantly returns to its original parent folder in Drive with status `READY`.
4. The scheduled purge task in `Jobs` is safely cancelled or skipped via strict locking order (Lock job before file).

#### Step 4: Immediate Permanent Purge
1. Move `report.pdf` back to Trash.
2. In `/trash`, click **Delete Permanently** and confirm the dialog.

*Asynchronous Purge Flow:*
1. **Immediate API Acknowledgment**: The backend sets file status to `PURGE_PENDING` and returns HTTP `202 Accepted`. In this transitional state, restore requests are rejected.
2. **Durable Worker Execution**:
   * The worker claims the `PURGE_FILE` task.
   * **Version-Aware S3 Deletion**: Because S3 Versioning is enabled, regular delete commands merely create a *Delete Marker* without reclaiming bytes. The worker:
     * Calls `ListObjectVersions` to retrieve all versions and delete markers for `objects/1/file_77b31a29` (including `null` versions).
     * Dispatches `DeleteObjects` targeting exact `(Key, VersionId)` tuples.
     * Queries S3 inventory to verify all versions for that key are gone.
   * **Quota Settlement & Tombstone**: After S3 confirms deletion, the worker executes a SQL transaction:
     * Deducts the physical file size from `Users.used_bytes`.
     * Retains a tombstone record with status `PURGED` and timestamp `purged_at` for audit traceability.

---

### 3. Scheduled Data Reconciliation Service

To guarantee absolute consistency between the database and S3, the automated reconciliation routine (`reconciliation.service.js`) executes 4 core operations:

```
+------------------------------------------------------------------------------------+
|                             RECONCILIATION SERVICE                                 |
+------------------------------------------------------------------------------------+
| 1. Expire Stalled Sessions: Cancel abandoned UploadSessions > 15 min -> Reclaim    |
| 2. Recover Missing Finalize: Auto-heal orphaned PASSED reports due to crashes      |
| 3. Quota Self-Healing: Recompute used_bytes from actual READY + TRASHED files      |
| 4. Orphan Object Detection: Identify unindexed objects on S3 outside DB records    |
+------------------------------------------------------------------------------------+
```

#### Detailed Operations:
1. **Stalled Upload Session Expiry**: Identifies `UploadSessions` in `ACTIVE` state older than 15 minutes. It rolls back reserved quota and transitions the session to `EXPIRED`.
2. **Recover Missing Finalize Work**: Detects uploads possessing a `PASSED` report on S3 that lack corresponding `Files` records due to worker crashes, re-queuing `FINALIZE_UPLOAD` jobs idempotently.
3. **Storage Quota Self-Healing**: Calculates actual storage consumption from the `Files` table (`SELECT SUM(size) WHERE user_id = ? AND status IN ('READY', 'TRASHED')`). Any divergence from `Users.used_bytes` is corrected and logged.
4. **Orphan Object Detection**: Scans the S3 `objects/` prefix:
   * Objects present in S3 but absent in MySQL are recorded in `ReconciliationFindings`.
   * **24-Hour Safety Rule**: Orphaned objects are only deleted if:
     * Deletion mode is explicitly enabled: `ORPHAN_CLEANUP_MODE=delete` (default is `report-only`).
     * The object has been observed persistently across a minimum 24-hour grace period.

---

### 4. Verification Steps

#### Check 1: Quota Settlement After Permanent Purge
Visit `/drive` or call `/api/users/quota`:
```bash
curl -i -b "connect.sid=..." http://localhost:3000/api/users/quota
```
*Expected Result:* `used_bytes` drops by the exact byte count of the purged file, and the sidebar quota bar reflects the reclaimed capacity.

#### Check 2: Audit Log Trail
Navigate to the **Activity** view in the sidebar (`http://localhost:5173/activity`):
* The audit log displays the complete immutable sequence:
  * `FILE_UPLOAD`: Successful ingest.
  * `FILE_TRASH`: Moved to trash.
  * `FILE_RESTORE`: Restored to drive.
  * `FILE_PURGE`: Permanently deleted.
