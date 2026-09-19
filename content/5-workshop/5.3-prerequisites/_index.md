---
title: "Prerequisites and Environment Setup"
date: 2026-09-18T23:00:00+07:00
weight: 3
chapter: false
pre: "<b>5.3. </b>"
---


---

### 1. Required Toolchains and Runtime Versions

Ensure your development workstation meets the minimum version requirements below:

| Tool / Software | Minimum Version | Purpose | Version Check Command |
| :--- | :--- | :--- | :--- |
| **Node.js** | `>= 20.x` *(v20.20.2 or v22.x LTS recommended)* | Runtime engine for Backend API and Worker | `node -v` |
| **npm** | `>= 10.x` | Monorepo dependency management via npm workspaces | `npm -v` |
| **MySQL Server** | `>= 8.0` *(or XAMPP MySQL on Windows)* | Relational database management system | `mysql --version` |
| **Git** | `>= 2.40` | Source control management | `git --version` |
| **AWS CLI** | `>= 2.x` *(Optional for live S3 verification)* | AWS local credential and CLI profile management | `aws --version` |
| **Terraform** | `1.10.5` *(Optional for IaC verification)* | Infrastructure validation and HCL security checks | `terraform version` |

---

### 2. Dependency Installation

The repository uses **npm workspaces** to manage both `backend/` and `frontend/` under a single root tree with a centralized `package-lock.json`.

#### Step 1: Open Terminal in Project Directory
```bash
cd C:\Users\admin\Desktop\Projectmanh
```

#### Step 2: Install All Dependencies
Execute installation from the workspace root. Do not run `npm install` inside subdirectories to prevent lockfile divergence:
```bash
npm install
```

*Expected Result:* The root `node_modules/` directory is fully populated with symlinks linking `backend` and `frontend` packages.

---

### 3. Secure Environment Configuration

Copy the template configuration `.env.example` to `.env` in the root directory:

```bash
cp .env.example .env
```

#### Sample Configuration Parameters:

```ini
# ==============================================================================
# BACKEND API SERVER CONFIGURATION
# ==============================================================================
PORT=3000
NODE_ENV=development
APP_ORIGIN=http://localhost:5173

# ==============================================================================
# LOCAL MYSQL DATABASE CONFIGURATION
# ==============================================================================
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=cloud_file_manager
DB_USER=root
DB_PASSWORD=your_secure_local_password

# ==============================================================================
# SESSION SECURITY AND STORAGE QUOTA
# ==============================================================================
SESSION_SECRET=local_dev_session_secret_change_in_production_min_32_chars
DEFAULT_QUOTA_BYTES=1073741824       # 1 GiB default quota per user
MAX_FILE_SIZE_BYTES=52428800         # 50 MiB max file upload limit
UPLOAD_SESSION_EXPIRATION_SECONDS=900 # Upload sessions expire in 15 minutes

# ==============================================================================
# AMAZON S3 STORAGE CONFIGURATION
# ==============================================================================
AWS_REGION=ap-southeast-1
AWS_S3_BUCKET=example-cloud-file-manager-bucket-local
# Note: The application leverages the AWS Default Credential Provider Chain.
# Never commit raw AWS Access Keys or Secret Keys into version control!

# ==============================================================================
# WORKER AND RECONCILIATION CONFIGURATION
# ==============================================================================
WORKER_POLL_INTERVAL_MS=3000
PURGE_RETENTION_DAYS=7
ORPHAN_CLEANUP_MODE=report-only      # Default to audit report only
```

---

### 4. Database Provisioning & Sequelize Migrations

#### Step 1: Start MySQL Server
Ensure the MySQL service is listening on port `3306` (via XAMPP Control Panel or MySQL Windows Service).

#### Step 2: Create Empty Database
Open your MySQL CLI or phpMyAdmin and execute:
```sql
CREATE DATABASE IF NOT EXISTS cloud_file_manager 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;
```

#### Step 3: Run Database Migrations
The project applies ordered sequential migrations from `001` to `007`. Run the following from the root directory:

```bash
npm run db:migrate
```

*Expected Terminal Output:*
```text
Applying migration: 001-create-users.js ... OK
Applying migration: 002-create-folders.js ... OK
Applying migration: 003-create-files.js ... OK
Applying migration: 004-phase5b-direct-upload.js ... OK
Applying migration: 005-phase6b-durable-worker.js ... OK
Applying migration: 006-phase7-lifecycle-reconciliation.js ... OK
Applying migration: 007-phase10a-correlation.js ... OK
All migrations executed successfully.
```

#### Step 4: Verify Migration Status
```bash
npm run db:status
```
This confirms that all business tables (`Users`, `Sessions`, `Invitations`, `Folders`, `Files`, `UploadSessions`, `Jobs`, `AuditLogs`, `ReconciliationFindings`) exist with their respective foreign keys and indexes.

---

### 5. Seed Initial Registration Invitation Token

Since registration is **Invitation-Only** to prevent resource exhaustion, create a valid invitation token in the `Invitations` table:

```sql
INSERT INTO Invitations (token, max_uses, uses_count, expires_at, created_at, updated_at)
VALUES (
    'WORKSHOP-INVITE-2026', 
    10, 
    0, 
    DATE_ADD(NOW(), INTERVAL 30 DAY), 
    NOW(), 
    NOW()
);
```

*The invitation code `WORKSHOP-INVITE-2026` is now ready for account registration.*
