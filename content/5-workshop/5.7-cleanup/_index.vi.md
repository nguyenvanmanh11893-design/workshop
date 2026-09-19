---
title: "Dọn dẹp tài nguyên và Tổng kết"
date: 2026-09-18T23:00:00+07:00
weight: 7
chapter: false
pre: "<b>5.7. </b>"
---





---

### 1. Phạm vi Tài nguyên Cần Dọn dẹp

Sau khi hoàn thành các bài thực hành, các tài nguyên được khởi tạo bao gồm:

| Phạm vi | Tài nguyên đã tạo | Mức độ ảnh hưởng | Hành động đề xuất |
| :--- | :--- | :---: | :--- |
| **Cục bộ (Local)** | Tiến trình Node.js (Dev server, API, Worker) | Tiêu thụ RAM / CPU cục bộ | Dừng tiến trình bằng phím tắt `Ctrl + C`. |
| **Cục bộ (Local)** | Thư mục build `frontend/dist/` | Chiếm dung lượng ổ đĩa cục bộ | Xóa thư mục nếu không cần đóng gói. |
| **Cơ sở dữ liệu** | Database `cloud_file_manager` và dữ liệu mẫu | Chiếm dung lượng MySQL | Giữ lại để đối chiếu hoặc xóa bảng kiểm thử. |
| **Amazon S3** | Các đối tượng trong `incoming/`, `objects/`, `reports/` | Tính phí lưu trữ S3 Storage | Xóa toàn bộ các tệp mẫu đã tải lên trong buổi lab. |
| **Hạ tầng AWS** | *(Nếu đã từng chạy thử Terraform trên AWS thật)* | Tính phí máy chủ EC2, RDS, CloudWatch | Thực hiện quy trình hủy tài nguyên qua Terraform. |

---

### 2. Hướng dẫn Dọn dẹp Môi trường Cục bộ (Local Environment)

#### Bước 1: Dừng các Tiến trình đang chạy
* Tại **Terminal 1** (chạy `npm run dev`): Nhấn tổ hợp phím `Ctrl + C`, chọn `Y` để kết thúc cả hai máy chủ Express API và Vite Frontend.
* Tại **Terminal 2** (chạy `npm run worker`): Nhấn tổ hợp phím `Ctrl + C` để dừng tiến trình xử lý nền. Tiến trình Worker sẽ thực hiện quy trình ngắt an toàn (Graceful Shutdown), hoàn tất các công việc đang dở dang và giải phóng các khóa lease.

#### Bước 2: Dọn dẹp Thư mục Biên dịch Tĩnh
Nếu đã thực thi lệnh `npm run build`, bạn có thể xóa thư mục đầu ra để giải phóng không gian đĩa:
```bash
rm -rf frontend/dist
```

#### Bước 3: Đặt lại Cơ sở dữ liệu Cục bộ (Tùy chọn)
Nếu muốn dọn dẹp sạch toàn bộ dữ liệu kiểm thử trong MySQL để chuẩn bị cho buổi thực hành mới:
```sql
DROP DATABASE cloud_file_manager;
CREATE DATABASE cloud_file_manager CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```
Sau đó chạy lại kịch bản di trú: `npm run db:migrate`.

---

### 3. Hướng dẫn Hủy Tài nguyên Đám mây AWS (AWS Cloud Teardown)

#### Bước 1: Xóa toàn bộ các Đối tượng và Phiên bản trong Bucket S3
Do bucket S3 được bật tính năng Versioning, lệnh xóa bucket thông thường sẽ bị lỗi nếu còn đối tượng hoặc delete markers. Bạn cần xóa sạch toàn bộ các phiên bản trước:
```bash
# Xóa toàn bộ các phiên bản tệp và delete markers trên S3
aws s3api delete-objects \
    --bucket example-cloud-file-manager-bucket \
    --delete "$(aws s3api list-object-versions \
        --bucket example-cloud-file-manager-bucket \
        --query='{Objects: Versions[].{Key:Key,VersionId:VersionId}}' \
        --output json)"
```

#### Bước 2: Hủy Hạ tầng qua Terraform
Di chuyển vào thư mục hạ tầng và thực hiện lệnh hủy:
```bash
cd infra/terraform
terraform destroy -var-file=terraform.tfvars
```
*Lưu ý về RDS Guard:* Cấu hình Terraform của dự án có thiết lập tham số bảo vệ dữ liệu `skip_final_snapshot = false` hoặc `deletion_protection = true`. Khi thực hiện destroy môi trường thử nghiệm, cần kiểm tra kỹ các biến này để tránh việc lệnh hủy bị chặn hoặc tạo ra snapshot lưu trữ tính phí ngoài ý muốn.

#### Bước 3: Dọn dẹp Tham số Bí mật trong AWS Systems Manager
Nếu đã bootstrap các biến mật mã trong SSM Parameter Store:
```bash
aws ssm delete-parameters --names \
    "/cloud-file-manager/prod/db-password" \
    "/cloud-file-manager/prod/session-secret"
```

---

### 4. Tổng kết Workshop và Bài học Kinh nghiệm 

Workshop **Cloud File Manager** đã hoàn thành trọn vẹn mục tiêu trang bị kiến thức và kỹ năng thực hành cho người tham gia:

1. **Mô hình Direct S3 Upload là chuẩn mực cho ứng dụng lưu trữ hiện đại**:
   * Việc chuyển toàn bộ tải I/O tệp nhị phân sang Amazon S3 thông qua Presigned POST Policy giúp máy chủ backend giữ được kích thước nhỏ gọn, tiêu thụ ít tài nguyên và dễ dàng mở rộng quy mô (Horizontal Scaling).
2. **Bảo mật nhiều lớp không thể chỉ dựa vào một cơ chế duy nhất**:
   * Cần sự kết hợp chặt chẽ giữa Opaque Session phía máy chủ, HttpOnly Cookie, token động phòng chống CSRF, và kiểm định nội dung nhị phân (magic bytes) độc lập bằng Serverless Lambda trước khi lưu trữ chính thức.
3. **Xử lý nền bền vững (Durable Worker) là xương sống của tính toàn vẹn dữ liệu**:
   * Cơ chế khóa phân tán (Lease Fencing) và thử lại có độ trễ ngẫu nhiên (Jittered Exponential Backoff) giúp hệ thống chống chọi tốt với sự cố crash tiến trình, không bị mất việc và không tạo ra xung đột dữ liệu.
4. **Đối soát tự động (Reconciliation) là giải pháp bắt buộc trong hệ thống phân tán**:
   * Dù kiến trúc được thiết kế cẩn thận đến đâu, sự cố mạng bất ngờ vẫn có thể gây lệch trạng thái giữa DB và Storage. Dịch vụ đối soát định kỳ giúp hệ thống tự phục hồi, tự sửa lỗi và bảo vệ hạn mức người dùng chính xác.
5. **Tính trung thực và kỷ luật trong kỹ thuật phần mềm**:
   * Luôn phân biệt rõ ràng giữa "mã nguồn đã hoàn thiện", "kết quả kiểm thử cục bộ" với "hệ thống đã triển khai production". Việc hiểu rõ các ranh giới kiểm chứng giúp kỹ sư tự tin làm chủ hệ thống và sẵn sàng cho các giai đoạn triển khai thực tế tiếp theo.
