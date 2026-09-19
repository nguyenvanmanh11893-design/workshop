---
title: "Chuẩn bị môi trường"
date: 2026-09-18T23:00:00+07:00
weight: 3
chapter: false
pre: "<b>5.3. </b>"
---





---

### 1. Yêu cầu Công cụ và Phiên bản Phần mềm

Trước khi bắt đầu, máy tính thực hành cần được cài đặt sẵn các công cụ với phiên bản tối thiểu như sau:

| Công cụ / Phần mềm | Phiên bản yêu cầu | Mục đích sử dụng | Lệnh kiểm tra phiên bản |
| :--- | :--- | :--- | :--- |
| **Node.js** | `>= 20.x` *(Khuyến nghị v20.20.2 hoặc v22.x LTS)* | Môi trường thực thi cho Backend API và Worker | `node -v` |
| **npm** | `>= 10.x` | Quản lý gói thư viện qua cơ chế npm workspaces | `npm -v` |
| **MySQL Server** | `>= 8.0` *(hoặc XAMPP MySQL trên Windows)* | Hệ quản trị cơ sở dữ liệu quan hệ chính | `mysql --version` |
| **Git** | `>= 2.40` | Quản lý phiên bản mã nguồn | `git --version` |
| **AWS CLI** | `>= 2.x` *(Tùy chọn khi kết nối S3 thật)* | Cấu hình thông tin xác thực AWS cục bộ | `aws --version` |
| **Terraform** | `1.10.5` *(Tùy chọn khi kiểm tra IaC)* | Kiểm tra và validate mã nguồn hạ tầng đám mây | `terraform version` |

---

### 2. Tải Mã nguồn và Cài đặt Thư viện Phụ thuộc

Hệ thống sử dụng cơ chế **npm workspaces** để đồng thời quản trị hai ứng dụng con `backend/` và `frontend/` dưới một cây thư mục thống nhất và một file khóa duy nhất `package-lock.json` tại thư mục gốc.

#### Bước 1: Mở terminal tại thư mục dự án
```bash
cd C:\Users\admin\Desktop\Projectmanh
```

#### Bước 2: Cài đặt toàn bộ thư viện phụ thuộc
Thực hiện cài đặt từ thư mục gốc. Không chạy `npm install` riêng lẻ bên trong các thư mục con nhằm tránh làm sai lệch file lockfile chung:
```bash
npm install
```

*Kết quả mong đợi:* Thư mục `node_modules/` tại thư mục gốc được khởi tạo đầy đủ; các liên kết tượng trưng (symlinks) cho `backend` và `frontend` được thiết lập tự động.

---

### 3. Thiết lập Cấu hình Biến Môi trường An toàn

Sao chép file cấu hình mẫu `.env.example` thành `.env` tại thư mục gốc:

```bash
cp .env.example .env
```

#### Bảng cấu hình biến môi trường mẫu :

```ini
# ==============================================================================
# CẤU HÌNH MÁY CHỦ API BACKEND
# ==============================================================================
PORT=3000
NODE_ENV=development
APP_ORIGIN=http://localhost:5173

# ==============================================================================
# CẤU HÌNH CƠ SỞ DỮ LIỆU MYSQL CỤC BỘ
# ==============================================================================
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=cloud_file_manager
DB_USER=root
DB_PASSWORD=your_secure_local_password

# ==============================================================================
# CẤU HÌNH BẢO MẬT PHIÊN VÀ HẠN MỨC DUNG LƯỢNG
# ==============================================================================
SESSION_SECRET=local_dev_session_secret_change_in_production_min_32_chars
DEFAULT_QUOTA_BYTES=1073741824       # 1 GiB mặc định cho mỗi người dùng mới
MAX_FILE_SIZE_BYTES=52428800         # Giới hạn 50 MiB cho mỗi tệp tải lên
UPLOAD_SESSION_EXPIRATION_SECONDS=900 # Phiên tải lên hết hạn sau 15 phút

# ==============================================================================
# CẤU HÌNH AMAZON S3 STORAGE (Sử dụng giá trị mẫu an toàn)
# ==============================================================================
AWS_REGION=ap-southeast-1
AWS_S3_BUCKET=example-cloud-file-manager-bucket-local
# Lưu ý: Ứng dụng ưu tiên nạp thông tin qua AWS Default Credential Provider Chain.
# Tuyệt đối không lưu Access Key / Secret Key dạng thô trong kho mã nguồn!

# ==============================================================================
# CẤU HÌNH TIẾN TRÌNH NỀN VÀ ĐỐI SOÁT (WORKER & RECONCILIATION)
# ==============================================================================
WORKER_POLL_INTERVAL_MS=3000
PURGE_RETENTION_DAYS=7
ORPHAN_CLEANUP_MODE=report-only      # Mặc định chỉ báo cáo, không tự xóa tệp mồ côi
```

---

### 4. Khởi tạo Cơ sở dữ liệu và Chạy Database Migrations

#### Bước 1: Khởi động MySQL Server
Đảm bảo dịch vụ MySQL đang chạy trên cổng `3306` (thông qua XAMPP Control Panel hoặc MySQL Windows Service).

#### Bước 2: Tạo cơ sở dữ liệu trống
Mở client dòng lệnh MySQL hoặc phpMyAdmin và tạo cơ sở dữ liệu UTF-8:
```sql
CREATE DATABASE IF NOT EXISTS cloud_file_manager 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;
```

#### Bước 3: Chạy bộ kịch bản di trú dữ liệu 
Dự án sử dụng kịch bản migration tuần tự từ 001 đến 007. Chạy lệnh sau từ thư mục gốc:

```bash
npm run db:migrate
```

*Kết quả mong đợi hiển thị trên terminal:*
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

#### Bước 4: Kiểm tra trạng thái di trú
```bash
npm run db:status
```
Lệnh này xác nhận toàn bộ các bảng nghiệp vụ (`Users`, `Sessions`, `Invitations`, `Folders`, `Files`, `UploadSessions`, `Jobs`, `AuditLogs`, `ReconciliationFindings`) đã được tạo thành công với đầy đủ các khóa ngoại và chỉ mục tối ưu.

---

### 5. Khởi tạo Mã mời Thành viên 

Do hệ thống hoạt động theo cơ chế **Chỉ đăng ký qua mã mời (Invitation-only Registration)** nhằm ngăn chặn việc lạm dụng tài nguyên, người thực hành cần tạo sẵn ít nhất một mã mời hợp lệ trong bảng `Invitations`.

Thực thi câu lệnh SQL sau trong database `cloud_file_manager`:

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

*Mã mời `WORKSHOP-INVITE-2026` hiện đã sẵn sàng để sử dụng tại bước thực hành đăng ký tài khoản.*
