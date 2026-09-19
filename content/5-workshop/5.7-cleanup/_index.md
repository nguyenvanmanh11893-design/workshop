---
title: "Resource Cleanup and Conclusion"
date: 2026-09-18T23:00:00+07:00
weight: 7
chapter: false
pre: "<b>5.7. </b>"
---



---

### 1. Scope of Resources to Clean Up

Following workshop completion, provisioned resources fall into the following scopes:

| Scope | Resource | Impact | Recommended Action |
| :--- | :--- | :---: | :--- |
| **Local** | Node.js processes (Dev server, API, Worker) | Consumes workstation RAM / CPU | Terminate processes via `Ctrl + C`. |
| **Local** | Build artifacts in `frontend/dist/` | Consumes local disk space | Delete folder if no packaging is required. |
| **Database** | Database `cloud_file_manager` and seeded data | Occupies local MySQL storage | Retain for reference or drop test tables. |
| **Amazon S3** | Objects in `incoming/`, `objects/`, `reports/` | Incurs S3 storage fees | Purge sample objects uploaded during the workshop. |
| **AWS Cloud** | *(If Terraform was tested on a live AWS account)* | Incurs EC2, RDS, and CloudWatch charges | Execute Terraform destruction workflow. |

---

### 2. Local Environment Teardown

#### Step 1: Terminate Running Processes
* In **Terminal 1** (running `npm run dev`): Press `Ctrl + C` and type `Y` to terminate both Express API and Vite dev servers.
* In **Terminal 2** (running `npm run worker`): Press `Ctrl + C` to stop the background worker. The worker initiates a graceful shutdown, completing pending tasks and releasing active database leases.

#### Step 2: Clean Up Compiled Static Assets
If you ran `npm run build`, delete the output directory to reclaim disk space:
```bash
rm -rf frontend/dist
```

#### Step 3: Reset Local Database (Optional)
To purge all workshop test data in MySQL and reset state:
```sql
DROP DATABASE cloud_file_manager;
CREATE DATABASE cloud_file_manager CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```
Then re-run migrations: `npm run db:migrate`.

---

### 3. AWS Cloud Teardown Instructions

#### Step 1: Purge All Objects and Versions in S3 Bucket
Because S3 Versioning is enabled, deleting a bucket fails if objects or delete markers remain. You must remove all versions first:
```bash
# Delete all object versions and delete markers from S3
aws s3api delete-objects \
    --bucket example-cloud-file-manager-bucket \
    --delete "$(aws s3api list-object-versions \
        --bucket example-cloud-file-manager-bucket \
        --query='{Objects: Versions[].{Key:Key,VersionId:VersionId}}' \
        --output json)"
```

#### Step 2: Destroy Infrastructure via Terraform
Navigate to the infrastructure directory and initiate destruction:
```bash
cd infra/terraform
terraform destroy -var-file=terraform.tfvars
```
*Note on RDS Protection*: Verify if `skip_final_snapshot = false` or `deletion_protection = true` is configured in your variables to avoid blocked teardowns or unwanted snapshot storage fees.

#### Step 3: Delete Parameters in AWS Systems Manager
If you populated secrets in SSM Parameter Store:
```bash
aws ssm delete-parameters --names \
    "/cloud-file-manager/prod/db-password" \
    "/cloud-file-manager/prod/session-secret"
```

---

### 4. Workshop Conclusion and Key Takeaways

The **Cloud File Manager** workshop equips participants with foundational full-stack cloud engineering expertise:

1. **Direct-to-S3 Uploads as the Gold Standard**:
   * Offloading binary streaming to Amazon S3 via Presigned POST Policies keeps API instances lightweight, resource-efficient, and primed for horizontal auto-scaling.
2. **Multi-Layered Defense-in-Depth**:
   * Effective application security requires synergy across server-side Opaque Sessions, HttpOnly cookies, dynamic CSRF mitigation, and out-of-band binary magic bytes verification using serverless functions.
3. **Durable Workers Guarantee Data Integrity**:
   * Database-backed lease fencing and jittered exponential backoffs ensure background tasks withstand process crashes without losing jobs or inducing concurrency races.
4. **Automated Reconciliation is Mandatory in Distributed Systems**:
   * Network anomalies inevitably create state drift between databases and object stores. Scheduled reconciliation routines ensure self-healing and accurate user quota accounting.
5. **Engineering Discipline and Rigor**:
   * Distinguishing clearly between "completed source code", "local mock verification", and "live production deployment" builds architectural maturity and prepares teams for enterprise delivery.
