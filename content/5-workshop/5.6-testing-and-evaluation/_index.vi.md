---
title: "Kiểm thử và Đánh giá kết quả"
date: 2026-09-18T23:00:00+07:00
weight: 6
chapter: false
pre: "<b>5.6. </b>"
---




---

### 1. Chiến lược và Ma trận Kiểm thử

Dự án áp dụng chiến lược kiểm thử đa tầng bao gồm: Kiểm tra tĩnh mã nguồn (Linting), Kiểm thử đơn vị (Unit Tests), Kiểm thử tích hợp giả lập (Mocked Integration Tests), Kiểm tra cú pháp hạ tầng (IaC Validation) và Kiểm thử các kịch bản an ninh tiêu cực (Negative Security Mutation Tests).

| Bộ kiểm thử / Công cụ | Đối tượng kiểm tra | Kết quả thực tế | Tỷ lệ đạt | Ghi chú kỹ thuật |
| :--- | :--- | :---: | :---: | :--- |
| **ESLint (`npm run lint`)** | Toàn bộ mã nguồn JS/TS | **PASS** | 100% | Đạt chuẩn lint, sửa các biến thừa, không làm thay đổi hành vi nghiệp vụ. |
| **Backend Tests (`npm test`)** | API, Auth, Quota, Worker, Lifecycle, S3 | **PASS** | **82 Pass / 0 Fail** | 82 bài test vượt qua; 2 bài test MySQL thực tế được bỏ qua (opt-in skip) do không có DB kết nối lúc chạy tự động. |
| **Frontend Tests (`npm test`)** | Component UI, Formatters, Api Client | **PASS** | **6 Pass / 0 Fail** | Kiểm tra hiển thị icon, định dạng dung lượng và cơ chế bắt token CSRF. |
| **Frontend Build (`npm run build`)** | React TypeScript compile & bundle | **PASS** | 100% | Biên dịch hoàn tất ra thư mục `frontend/dist/`. |
| **Kiểm tra Phụ thuộc (`npm audit`)** | Rà soát lỗ hổng thư viện | **PASS** | 0 High / 0 Crit | Cập nhật AWS SDK; không còn lỗ hổng mức High/Critical (còn 7 cảnh báo Moderate gián tiếp). |
| **Terraform Validate (`terraform`)** | Cấu hình HCL tại `infra/terraform/` | **PASS** | 100% | Cú pháp HCL 1.10.5 hợp lệ hoàn toàn; đã cài đặt AWS Provider 5.100.0. |
| **Kiểm thử Bảo mật HCL (`check_infra.py`)** | Các quy tắc cấm đột biến nguy hiểm | **PASS** | **13/13 Cases** | Chặn thành công 13 kịch bản: DB công khai, mật khẩu bản rõ, IMDSv1, SSH, NAT, xóa quy tắc dữ liệu. |
| **Kiểm thử Launcher (`test_launch.py`)** | Khởi chạy host và nạp bí mật | **PASS** | **3/3 Tests** | Kiểm tra cơ chế nạp SecureString từ SSM, từ chối nạp khi thiếu biến hoặc timeout. |
| **Kiểm thử Đóng gói & Rollback Phase 10A** | Quy trình release và tự động phục hồi | **PASS** | **6/6 Tests** | Kiểm tra xác thực mã băm SHA-256, từ chối file `.env`, rollback tự động khi smoke test lỗi. |

---

### 2. Chi tiết Kết quả Kiểm thử Backend (82 Tests Passed)

Bộ kiểm thử backend sử dụng trình chạy kiểm thử tiêu chuẩn của Node.js (`node --test`), bao quát toàn bộ các module then chốt qua các giai đoạn phát triển:

* **Phase 1 (Cấu trúc cơ bản & S3 Mock)**: Kiểm thử các API tệp tin cơ bản, giả lập lỗi S3 và xử lý ngoại lệ mạng.
* **Phase 2 (Quản lý Thư mục)**: Kiểm tra phân cấp thư mục cha - con, đổi tên, di chuyển và ngăn chặn xóa thư mục có chứa tệp.
* **Phase 3 (Bảo mật Phiên & CSRF)**: Kiểm thử tạo phiên Opaque trong DB, gắn cờ HttpOnly, kiểm tra từ chối khi thiếu header `X-CSRF-Token`.
* **Phase 4 (Hạn mức Quota)**: Kiểm thử tính toán dung lượng, chặn tải lên khi vượt hạn mức, kiểm tra tính toán tổng dung lượng.
* **Phase 5 (Direct Upload & Presigned POST)**: Kiểm tra sinh chính sách Presigned POST, ràng buộc kích thước tệp và tiền tố `incoming/`.
* **Phase 6 (Durable Worker & Lease Fencing)**: Kiểm tra xử lý tranh chấp giữa 2 worker chạy song song, hết hạn lease, gia hạn lease, thử lại lũy tiến (retry exponential backoff).
* **Phase 7 (Vòng đời Tệp & Reconciliation)**: Kiểm thử Thùng rác (Trash), khôi phục (Restore), xóa S3 chính xác theo từng phiên bản (`DeleteObjects`), quét tệp mồ côi và đối soát tự cân bằng hạn mức.
* **Phase 10A (Observability & Health Probes)**: Kiểm thử khử dữ liệu nhạy cảm trong log (redaction), gắn mã truy vết `requestId`, kiểm tra cổng `/health/live` và `/health/ready`.

---

### 3. Ranh giới và Giới hạn Kiểm chứng (Testing Boundaries & Limitations)

Để đảm bảo tính khách quan của báo cáo kỹ thuật, các giới hạn sau đây được ghi nhận rõ ràng:

1. **Chưa triển khai lên AWS Production (Not Deployed)**:
   * Toàn bộ các tài nguyên đám mây (VPC, Subnet, EC2, RDS, Lambda, SQS, CloudWatch) mới chỉ được kiểm tra tĩnh qua mã nguồn Terraform (`validate` và `check_infra.py`).
   * Chưa có bất kỳ lệnh `terraform apply` nào được thực thi trên tài khoản AWS thực tế.
2. **Kiểm thử Môi trường Giả lập (Mocked Tests)**:
   * Các bài kiểm thử đơn vị sử dụng dữ liệu giả lập (mocks/stubs) cho Amazon S3, AWS Lambda và Amazon SQS.
   * Kết quả pass tại máy trạm cục bộ **không chứng minh** rằng mạng AWS IAM, phân quyền S3 Bucket Policy hay độ trễ mạng thực tế đã hoạt động hoàn hảo.
3. **Chưa chạy kiểm thử Tải và Tranh chấp MySQL thực tế (Real Concurrency)**:
   * 2 bài kiểm thử tích hợp MySQL thực tế (`phase6b real lock race` và `phase7 mysql integration`) đã bị bỏ qua (skipped) trong lần chạy tự động do yêu cầu cơ sở dữ liệu chuyên biệt.
   * Cần thực hiện kiểm thử trên cụm MySQL InnoDB thực tế để kiểm chứng khả năng chịu tải hàng nghìn kết nối đồng thời.
4. **Chưa thực hiện Kiểm thử Trình duyệt Toàn diện (Browser E2E)**:
   * Chưa chạy bộ kiểm thử tự động trên trình duyệt thật (như Playwright hoặc Cypress) trên môi trường triển khai thực tế.
   * Trạng thái hiện tại của dự án là **Phase 10A đã hoàn thành mã nguồn; Phase 10B (Live Acceptance trên AWS) vẫn chưa thực hiện**.

---

### 4. Bảng Xử lý Sự cố Thường gặp (Troubleshooting Guide)

Trong quá trình thực hành workshop, người tham gia có thể gặp một số lỗi phổ biến sau:

| Mã lỗi / Hiện tượng | Nguyên nhân gốc rễ | Cách xử lý và khắc phục |
| :--- | :--- | :--- |
| **`400 Quota Exceeded`** | Dung lượng tệp tải lên vượt quá hạn mức còn lại của người dùng. | Kiểm tra dung lượng tại `/api/users/quota`; dọn dẹp Thùng rác hoặc nâng hạn mức trong bảng `Users` (`max_quota_bytes`). |
| **`403 Invalid CSRF Token`** | Gửi yêu cầu POST/PUT/DELETE nhưng thiếu hoặc sai header `X-CSRF-Token`. | Đảm bảo client đã gọi `GET /api/auth/csrf` để nhận token hợp lệ và đính kèm vào header yêu cầu. |
| **`400 Invalid Invitation Code`** | Mã mời không tồn tại, đã hết lượt sử dụng (`uses_count >= max_uses`) hoặc hết hạn. | Kiểm tra bảng `Invitations` trong cơ sở dữ liệu và tạo mã mới theo hướng dẫn tại mục 5.3. |
| **Tệp mãi ở trạng thái `UPLOADED`, không chuyển sang `READY`** | Tiến trình Worker chưa được bật, hoặc Lambda chưa tạo file `reports/{uploadId}.json`. | Kiểm tra Terminal 2 xem `npm run worker` có đang chạy không; kiểm tra thư mục S3 `reports/` xem báo cáo kiểm định đã được sinh ra chưa. |
| **`503 Service Unavailable` tại `/health/ready`** | Cơ sở dữ liệu MySQL bị dừng hoặc sai thông tin kết nối trong `.env`. | Khởi động lại dịch vụ MySQL Server; kiểm tra các tham số `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`. |
| **`SignatureDoesNotMatch` khi Direct Upload S3** | Sai lệch thông tin cấu hình AWS Credentials hoặc lệch thời gian hệ thống (Clock Skew). | Đồng bộ lại đồng hồ hệ điều hành; kiểm tra biến `AWS_REGION` và thông tin cấu hình AWS CLI. |
