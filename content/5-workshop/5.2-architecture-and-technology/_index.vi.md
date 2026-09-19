---
title: "Kiến trúc và Công nghệ"
date: 2026-09-18T23:00:00+07:00
weight: 2
chapter: false
pre: "<b>5.2. </b>"
---




---

### 1. Kiến trúc Tổng thể Hệ thống

Hệ thống **Cloud File Manager** được thiết kế theo kiến trúc phân tách rõ ràng giữa lớp quản lý siêu dữ liệu (Metadata Management) và lớp lưu trữ nhị phân (Binary Object Storage). 

<!-- > [CHÈN HÌNH 5-01: Sơ đồ kiến trúc tổng thể Cloud File Manager — thể hiện các thành phần và luồng dữ liệu đã được xác minh.] -->

```
+----------------------------------------------------------------------------------------------------+
|                                         CLIENT LAYER                                               |
|  React 18 + TypeScript + Vite + Ant Design 5 (Single Page Application)                             |
|  - HttpOnly Cookie (Opaque Session ID) | Custom Header: X-CSRF-Token                               |
+-----------------------------------+----------------------------------------+-----------------------+
                                    |                                        |
                 1. Khởi tạo phiên  | 2. Nhận S3 Presigned POST              | 3. Tải tệp trực tiếp
                    và giữ chỗ Quota|    Policy & Credentials                |    (Direct Upload)
                                    v                                        v
+-----------------------------------+--------------------+   +---------------+-----------------------+
|                 BACKEND API SERVER                     |   |               AMAZON S3               |
|  Node.js (>=20) / Express.js (ES Modules)              |   |  Bucket: Private, Versioned, TLS-only |
|  - Middleware: Opaque Session, CSRF, Error, RequestId  |   |                                       |
|  - Controllers: Auth, Folder, File, Quota, Activity    |   |  Cấu trúc tiền tố (Prefixes):         |
|  - Services: Presigned URL, Quota, Lifecycle           |   |  - incoming/{userId}/{uploadId}       |
|  - Sequelize ORM (Migrations, Models)                  |   |  - objects/{userId}/{fileId}          |
+-----------------------------------+--------------------+   |  - reports/{uploadId}.json            |
                                    |                        +---------------+-----------------------+
                                    | 4. Lưu metadata tạm                    |
                                    v    vào bảng UploadSessions             | 5. s3:ObjectCreated
+-----------------------------------+--------------------+                   v    (chỉ tiền tố incoming/)
|                 DATABASE LAYER                         |   +---------------+-----------------------+
|  MySQL (InnoDB Engine)                                 |   |            AWS LAMBDA                 |
|  - Bảng: Users, Sessions, Invitations                  |   |  s3-file-validator (Outside VPC)      |
|  - Bảng: Folders, Files, UploadSessions                |   |  - Đọc magic bytes phân tích tệp      |
|  - Bảng: Jobs, AuditLogs, ReconciliationFindings       |   |  - Xuất báo cáo reports/{uploadId}.json|
+-----------------------------------+--------------------+   +---------------+-----------------------+
                                    ^                                        |
                                    | 7. Giao dịch khóa an toàn              | 6. SQS Failure
                                    |    (Fenced DB Transaction)             |    Destination (DLQ)
+-----------------------------------+--------------------+                   v
|             DURABLE BACKGROUND WORKER                  |   +---------------+-----------------------+
|  Tiến trình nền riêng biệt: npm run worker             |   |            AMAZON SQS                 |
|  - Quét bảng Jobs: FINALIZE_UPLOAD, PURGE_FILE, ...    |<--+  validator-dlq (Mã hóa SSE, 14 ngày)  |
|  - Tranh chấp khóa: Lease token & lease expiration     |   +---------------------------------------+
|  - Sao chép tệp nội bộ S3 (incoming -> objects)       |
|  - Cập nhật File READY, quyết toán Quota & AuditLog    |
+--------------------------------------------------------+
```

---

### 2. Vai trò và Chức năng của Từng Thành phần

#### 2.1 Frontend 
* **Công nghệ**: React 18, TypeScript, Vite, Ant Design 5, React Router v6.
* **Vai trò**:
  * Cung cấp giao diện đĩa lưu trữ đám mây trực quan với cây thư mục phân cấp, thanh điều hướng breadcrumbs, hộp thoại tạo/đổi tên/di chuyển thư mục.
  * Hộp thoại tải tệp (`UploadModal.tsx`) hỗ trợ kéo thả tệp, kiểm tra sơ bộ định dạng (PDF, JPEG, PNG, TXT) và dung lượng (< 50 MiB) phía trình duyệt.
  * Quản lý phiên đăng nhập trong suốt bằng cookie, tự động đính kèm header `X-CSRF-Token` trong toàn bộ các yêu cầu HTTP ghi dữ liệu (POST, PUT, DELETE).
  * Thực hiện gửi yêu cầu Multipart Form trực tiếp lên endpoint của Amazon S3 và cập nhật thanh tiến trình (progress bar) theo thời gian thực.

#### 2.2 Backend API Server
* **Công nghệ**: Node.js (>=20), Express.js (ES Modules), Sequelize ORM.
* **Vai trò**:
  * **Xác thực & Phân quyền**: Kiểm tra tính hợp lệ của mã mời khi đăng ký; quản lý phiên làm việc bằng Opaque Session ID lưu trong bảng `Sessions`; xác thực quyền sở hữu tệp tuyệt đối theo `req.user.id`.
  * **Tạo chính sách tải tệp S3**: Sinh ra Presigned POST Policy với đầy đủ điều kiện ràng buộc khắt khe (kích thước tối đa, đường dẫn chính xác tại tiền tố `incoming/{userId}/{uploadId}`).
  * **Quản lý hạn mức (Quota Management)**: Đặt trước dung lượng (`used_bytes + size <= max_bytes`) ngay khi phiên tải lên được tạo, ngăn chặn việc người dùng vượt quá dung lượng cho phép.
  * **API quản lý vòng đời**: Cung cấp các endpoint chuyển tệp vào thùng rác (`/trash`), khôi phục (`/restore`), xóa vĩnh viễn (`/purge`) và xem nhật ký thao tác (`/activity`).

#### 2.3 Amazon S3
* **Cấu hình**: Bucket riêng tư (Private), bật tính năng kiểm soát phiên bản (Bucket Versioning), bắt buộc giao thức TLS (HTTPS only) và chặn toàn bộ quyền truy cập công khai (Block Public Access).
* **Quy hoạch tiền tố (Prefix Architecture)**:
  * `incoming/{userId}/{uploadId}`: Lưu trữ tạm thời tệp người dùng vừa tải lên.
  * `reports/{uploadId}.json`: Chứa kết quả thẩm định nội dung tệp do hàm Lambda sinh ra.
  * `objects/{userId}/{fileId}`: Lưu trữ chính thức các tệp đã vượt qua khâu kiểm định nội dung.

#### 2.4 AWS Lambda Validator
* **Hàm**: `lambdas/s3-file-validator` (Node.js 20/22 runtime).
* **Vai trò**:
  * Đặt ngoài VPC để tối ưu thời gian khởi động (cold start) và tiết kiệm chi phí ENI/NAT Gateway.
  * Nhận sự kiện `s3:ObjectCreated` tại tiền tố `incoming/`.
  * Phân tích byte đầu tiên (magic bytes) của tệp tin:
    * PDF: Bắt đầu bằng chuỗi nhị phân tương ứng `%PDF-`.
    * PNG: Bắt đầu bằng 8 bytes chữ ký `\x89PNG\r\n\x1a\n`.
    * JPEG: Bắt đầu bằng 3 bytes `\xFF\xD8\xFF`.
    * TXT: Xác thực mã hóa ký tự hợp lệ UTF-8 / ASCII.
  * Ghi báo cáo thẩm định dạng JSON chuẩn tắc vào `reports/{uploadId}.json`.
  * Nếu phát sinh lỗi hoặc ngoại lệ, sự kiện tự động được đẩy về hàng đợi **Amazon SQS Dead-Letter Queue (DLQ)**.

#### 2.5 Lớp Xử lý Nền Bền vững 
* **Tiến trình**: Chạy riêng biệt thông qua lệnh `npm run worker`.
* **Vai trò**:
  * Quét các công việc đến hạn trong bảng `Jobs` của MySQL theo cơ chế tranh chấp khóa phân tán (Lease Fencing).
  * Đọc báo cáo kiểm định từ `reports/{uploadId}.json`:
    * Nếu **HỢP LỆ (PASSED)**: Thực hiện sao chép nguyên trạng phiên bản chính xác (Exact-version Copy) từ `incoming/` sang `objects/`, mở giao dịch cơ sở dữ liệu cập nhật trạng thái tệp sang `READY`, quyết toán hạn mức và ghi lịch sử kiểm toán.
    * Nếu **TỪ CHỐI (REJECTED)**: Cập nhật trạng thái `REJECTED`, hoàn trả hạn mức lưu trữ đã đặt trước và ghi nhật ký cảnh báo.
  * Thực hiện các tác vụ nền định kỳ: Xóa phiên tải lên bỏ dở quá hạn (`EXPIRE_UPLOAD_SESSION`), xóa vĩnh viễn tệp trong thùng rác sau 7 ngày (`PURGE_FILE`), và chạy thuật toán đối soát tệp mồ côi (`RECONCILE`).

---

### 3. Thiết kế Ranh giới Mạng và Bảo mật 

Theo cấu hình hạ tầng Terraform chuẩn hóa tại Singapore (`ap-southeast-1`):

<!-- > [CHÈN HÌNH 5-02: Sơ đồ mạng và phân vùng bảo mật AWS theo thiết kế Terraform tại Singapore (ap-southeast-1).] -->

```
+----------------------------------------------------------------------------------------------------+
|                                    AWS REGION: ap-southeast-1 (Singapore)                          |
|                                                                                                    |
|  +----------------------------------------------------------------------------------------------+  |
|  | VPC (10.0.0.0/16)                                                                            |  |
|  |                                                                                              |  |
|  |  +-------------------------------------+  +-----------------------------------------------+  |  |
|  |  | Public Subnet (10.0.1.0/24)          |  | Private DB Subnets (10.0.11.0/24, 10.0.12.0/24) |  |
|  |  |                                     |  | (Không có Internet Gateway hoặc NAT Gateway)  |  |  |
|  |  |  [EC2 Instance]                    |  |                                               |  |  |
|  |  |  - Nginx Reverse Proxy (443 TLS)    |  |  [Amazon RDS MySQL]                           |  |  |
|  |  |  - Express API (127.0.0.1:3000)     |  |  - Port 3306 chỉ mở cho EC2 Security Group    |  |  |
|  |  |  - Durable Worker (Tiến trình nền)  |  |  - Mã hóa KMS lưu trữ dữ liệu                 |  |  |
|  |  |  - IMDSv2 bắt buộc                  |  |  - Tự động sao lưu 7 ngày                     |  |  |
|  |  |  - Không mở cổng SSH (Port 22)      |  |                                               |  |  |
|  |  +------------------+------------------+  +-----------------------+-----------------------+  |  |
|  |                     |                                             |                          |  |
|  |                     +----------------------+----------------------+                          |  |
|  |                                            |                                                 |  |
|  |                     +----------------------v----------------------+                          |  |
|  |                     | S3 Gateway Endpoint (Miễn phí, nội bộ VPC)  |                          |  |
|  |                     +----------------------+----------------------+                          |  |
|  +--------------------------------------------|-------------------------------------------------+  |
|                                               |                                                    |
|  +--------------------------------------------v-------------------------------------------------+  |
|  | Dịch vụ AWS ngoài VPC (Outside VPC Services)                                                 |  |
|  |                                                                                              |  |
|  |  [Amazon S3 Bucket]               [AWS Lambda]                   [Amazon SQS]                |  |
|  |  - Private, Versioned, CORS       - s3-file-validator            - validator-dlq (Encrypted) |  |
|  |  - incoming/, objects/, reports/  - Least privilege IAM          - Lưu vết lỗi 14 ngày       |  |
|  +----------------------------------------------------------------------------------------------+  |
+----------------------------------------------------------------------------------------------------+
```

#### Các chốt bảo mật cốt lõi:
1. **Không mở cổng SSH**: Toàn bộ thao tác quản trị máy chủ EC2 được thực hiện qua AWS Systems Manager (SSM Session Manager), loại bỏ hoàn toàn nguy cơ tấn công brute-force cổng 22.
2. **Cơ sở dữ liệu hoàn toàn cách ly**: RDS MySQL nằm trong Subnet riêng biệt, không có bảng định tuyến ra Internet Gateway, chỉ chấp nhận kết nối trên cổng 3306 từ duy nhất Security Group của máy chủ EC2.
3. **S3 Gateway Endpoint**: Toàn bộ lưu lượng truyền tải dữ liệu giữa EC2/Worker và Amazon S3 đi qua Gateway Endpoint nội bộ của AWS, không đi qua mạng Internet công cộng và không phát sinh phí truyền dữ liệu.
4. **Bảo vệ danh tính đám mây bằng IMDSv2**: Bắt buộc sử dụng Instance Metadata Service phiên bản 2 (`http_tokens = "required"`), vô hiệu hóa các cuộc tấn công SSRF nhằm đánh cắp IAM Role của máy chủ.

---

### 4. Trạng thái Triển khai: Cục bộ vs. Đám mây AWS

Để đảm bảo tính trung thực kỹ thuật tuyệt đối trong báo cáo, sự khác biệt giữa môi trường cục bộ và đám mây AWS được xác định rõ:

| Tiêu chí | Môi trường Cục bộ  | Môi trường AWS Production |
| :--- | :--- | :--- |
| **Trạng thái thực tế** | **ĐÃ HOÀN THIỆN & KIỂM THỬ** | **ĐÃ CẤU HÌNH & KIỂM TRA MÃ NGUỒN** |
| **Giao diện Web** | Chạy bằng Vite dev server (`http://localhost:5173`) | Được build thành tài nguyên tĩnh và phân phối qua Nginx HTTPS |
| **API & Worker** | Chạy bằng Node.js trên máy trạm cục bộ | Chạy dưới dạng systemd service do root sở hữu trên EC2 Linux |
| **Cơ sở dữ liệu** | MySQL Server cục bộ (hoặc XAMPP) cổng 3306 | Amazon RDS MySQL Multi-Subnet có mã hóa lưu trữ KMS và sao lưu 7 ngày |
| **Kho lưu trữ tệp** | Amazon S3 bucket thật hoặc giả lập S3 qua bộ test adapter | Amazon S3 Singapore với Versioning, TLS-only, CORS và Gateway Endpoint |
| **Kiểm định tệp** | Xử lý trực tiếp qua adapter dịch vụ hoặc Lambda đóng gói | AWS Lambda trigger tự động từ S3, có SQS Dead-Letter Queue |
| **Bí mật & Cấu hình**| File `.env` cục bộ (gitignored, chỉ nạp khi dev) | AWS SSM Parameter Store dạng `SecureString`, nạp trực tiếp vào RAM |
