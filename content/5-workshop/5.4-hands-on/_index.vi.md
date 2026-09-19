---
title: "Các bước thực hành chính"
date: 2026-09-18T23:00:00+07:00
weight: 4
chapter: false
pre: "<b>5.4. </b>"
---





---

### Lộ trình Thực hành Từng bước

Mỗi bài thực hành được thiết kế chuẩn mực với đầy đủ: **Mục đích**, **Điều kiện tiên quyết**, **Thao tác & Lệnh thực thi**, **Kết quả mong đợi (Expected Outcome)** và **Cách thức kiểm tra, xác minh (Verification)**.

* **[5.4.1. Khởi chạy và kiểm tra dịch vụ](5.4.1-bootstrap-services/)**:
  * Khởi động máy chủ phát triển Frontend và Backend API (`npm run dev`).
  * Khởi động tiến trình xử lý nền Worker độc lập (`npm run worker`).
  * Kiểm tra các cổng giám sát sức khỏe (Health Probes): `/health/live` và `/health/ready`.
* **[5.4.2. Xác thực và Tải tệp trực tiếp lên S3](5.4.2-auth-and-direct-upload/)**:
  * Đăng ký tài khoản người dùng bằng mã mời hợp lệ.
  * Đăng nhập nhận Opaque Session Cookie và truy xuất mã token CSRF.
  * Tạo thư mục cá nhân và điều hướng cây thư mục.
  * Khởi tạo phiên tải tệp, kiểm tra hạn mức lưu trữ (Quota Reservation) và tải tệp trực tiếp từ trình duyệt lên tiền tố S3 `incoming/` bằng Presigned POST Policy.
* **[5.4.3. Kiểm định tệp với Lambda và Durable Worker](5.4.3-validation-and-finalization/)**:
  * Cơ chế sự kiện S3 `s3:ObjectCreated` kích hoạt hàm AWS Lambda `s3-file-validator`.
  * Phân tích magic bytes nhị phân của tệp và ghi báo cáo chuẩn tắc `reports/{uploadId}.json`.
  * Tiến trình Durable Worker tranh chấp khóa (Lease Fencing) trong bảng `Jobs`, sao chép tệp an toàn sang `objects/`, mở giao dịch cơ sở dữ liệu cập nhật trạng thái `READY` và quyết toán hạn mức.
* **[5.4.4. Vòng đời tệp, Thùng rác và Đối soát dữ liệu](5.4.4-lifecycle-and-reconciliation/)**:
  * Lấy Presigned URL có thời hạn (15 phút) để tải tệp xuống máy khách an toàn.
  * Chuyển tệp vào Thùng rác (Trash) và đặt lịch xóa vĩnh viễn sau 7 ngày (Delayed Purge).
  * Khôi phục tệp về cây thư mục trước khi tiến trình xóa kích hoạt.
  * Thực thi yêu cầu xóa vĩnh viễn chủ động (`PURGE_PENDING` -> HTTP 202) và xóa S3 theo đúng từng cặp phiên bản `(Key, VersionId)`.
  * Chạy tiến trình đối soát (Reconciliation) phát hiện tệp rác mồ côi và tự động cân bằng số đo hạn mức.
