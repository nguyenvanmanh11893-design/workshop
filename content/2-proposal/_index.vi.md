---
title: "Đề xuất"
date: 2026-09-18T23:00:00+07:00
weight: 2
chapter: false
pre: "<b>2. </b>"
---

# Hệ thống Quản lý Tệp Đám mây Cá nhân 

---

### 1. Tóm tắt điều hành

**Cloud File Manager** là giải pháp lưu trữ và quản lý tệp tin cá nhân trên nền tảng điện toán đám mây (mô hình tương tự Google Drive), được thiết kế nhằm mang lại trải nghiệm lưu trữ trực quan, an toàn và tối ưu tài nguyên cho người dùng. 

Hệ thống kết hợp sức mạnh của kiến trúc web hiện đại (**React 18, TypeScript, Vite, Ant Design**) với backend hướng dịch vụ (**Node.js, Express, Sequelize, MySQL**) và các dịch vụ đám mây công nghiệp của **Amazon Web Services (AWS)** như **Amazon S3, AWS Lambda, Amazon SQS, Amazon CloudWatch** cùng mô hình hạ tầng dưới dạng mã (**Terraform**). 

Trọng tâm thiết kế của giải pháp là giải quyết triệt để các rào cản hiệu năng và bảo mật truyền thống thông qua:
* **Cơ chế Direct-to-S3 Upload**: Tải tệp trực tiếp từ trình duyệt lên S3 bằng Presigned POST Policy, loại bỏ hoàn toàn hiện tượng nghẽn I/O và cạn kiệt bộ nhớ tại máy chủ API.
* **Đặt trước và kiểm soát hạn mức lưu trữ (Quota Reservation)** nghiêm ngặt.
* **Kiểm định nội dung tự động bằng Lambda không nằm trong VPC (Outside-VPC Lambda Validator)**.
* **Hàng đợi xử lý nền bền vững (Durable Background Worker)** có cơ chế khóa phân tán (lease fencing), thử lại lũy tiến (jittered retry) và bảo đảm tính lũy biến (idempotency).
* **Quản lý vòng đời tệp tin chặt chẽ**: Thùng rác (Trash), khôi phục (Restore), xóa vĩnh viễn theo phiên bản (Version-aware S3 Deletion) và tiến trình đối soát (Reconciliation) định kỳ.

---

### 2. Mô tả bài toán và Bối cảnh 

#### 2.1 Vấn đề thực tế
Trong các ứng dụng lưu trữ tệp tin truyền thống, toàn bộ luồng dữ liệu nhị phân (binary streams) của tệp tải lên thường phải đi qua máy chủ API backend trước khi được ghi xuống ổ đĩa hoặc chuyển tiếp lên kho lưu trữ đám mây. Kiến trúc này bộc lộ nhiều điểm yếu nghiêm trọng:
1. **Nghẽn cổ chai I/O và RAM máy chủ**: Máy chủ backend nhanh chóng quá tải khi nhiều người dùng tải lên các tệp dung lượng trung bình đến lớn (ví dụ 50 MiB) cùng lúc.
2. **Chi phí băng thông nhân đôi (Double Egress/Ingress)**: Dữ liệu tải từ máy khách lên API server, rồi từ API server lại đẩy sang kho lưu trữ, gây lãng phí băng thông và tăng độ trễ mạng.
3. **Nguy cơ tệp độc hại và giả mạo định dạng**: Việc chỉ dựa vào phần mở rộng tệp (`.pdf`, `.png`) hoặc header `Content-Type` do người dùng gửi lên dễ bị vượt qua nếu không có bước xác thực nội dung nhị phân (magic bytes).
4. **Mất đồng bộ giữa Database và Storage (State Drift)**: Nếu quá trình tải lên hoặc xóa tệp trên S3 bị gián đoạn (do lỗi mạng hoặc crash ứng dụng), siêu dữ liệu trong cơ sở dữ liệu và tệp thực tế trên S3 sẽ không khớp nhau, dẫn đến tệp rác mồ côi hoặc sai lệch hạn mức lưu trữ.
5. **Rủi ro bảo mật phiên và truy cập dữ liệu chéo (Multi-tenant leakage)**: Trong môi trường nhiều người dùng, việc xác thực lỏng lẻo hoặc kiểm tra quyền sở hữu không triệt để có thể dẫn đến việc người dùng truy cập hoặc can thiệp trái phép vào tệp của người khác.

#### 2.2 Mục tiêu dự án
* Xây dựng giao diện ứng dụng đĩa lưu trữ cá nhân (Personal Drive) hiện đại, mượt mà, phân cấp thư mục rõ ràng và hỗ trợ tiếng Việt.
* Triển khai luồng tải tệp trực tiếp lên Amazon S3 bảo mật, nhanh chóng và không làm tải máy chủ backend.
* Đảm bảo tính toàn vẹn và nhất quán dữ liệu giữa MySQL và Amazon S3 trong mọi kịch bản lỗi mạng, crash hoặc tiến trình bị hủy đột ngột.
* Thiết lập ranh giới bảo mật nhiều lớp: Xác thực chỉ qua mã mời (Invitation-only), phiên bảo mật Opaque Session lưu trong HttpOnly Cookie, phòng chống tấn công CSRF và cô lập tài nguyên theo `user_id`.

#### 2.3 Đối tượng sử dụng và Phạm vi
* **Đối tượng**: Người dùng cá nhân, nhóm nghiên cứu hoặc kỹ sư cần một không gian lưu trữ tài liệu, hình ảnh an toàn, được kiểm soát dung lượng và kiểm định tệp nghiêm ngặt.
* **Phạm vi 6 tuần**: Tập trung hoàn thiện kiến trúc lõi, quy trình xác thực, upload S3 an toàn, xử lý nền, vòng đời tệp, kiểm thử toàn diện, giám sát CloudWatch và kịch bản thực hành tại workshop. Thiết kế hạ tầng Terraform và pipeline CI/CD được đóng gói ở mức kiểm thử cục bộ và sẵn sàng cấu hình, chưa thực hiện triển khai live lên môi trường production thực tế.

---

### 3. Giải pháp và Kiến trúc đề xuất

Hệ thống được thiết kế theo mô hình 3 tầng phân tán kết hợp với các dịch vụ Serverless và hàng đợi trên AWS:

<!-- > [CHÈN HÌNH 2-01: Sơ đồ kiến trúc giải pháp đề xuất cho Cloud File Manager — thể hiện các thành phần Frontend, Backend API, MySQL, Amazon S3, Lambda Validator và Durable Worker.] -->

```
+-----------------------------------------------------------------------------------+
|                                  CLIENT LAYER                                     |
|  React 18 + TypeScript + Vite + Ant Design 5 (SPA)                                |
|  - HttpOnly Cookie (Opaque Session)                                               |
|  - X-CSRF-Token Header                                                            |
+-------------------+---------------------------------------+-----------------------+
                    |                                       |
       1. Request   | 2. Presigned POST                     | 3. Direct Upload
          Session   |    Policy + Quota                     |    (Multipart form)
                    v                                       v
+-------------------+-------------------+   +---------------+-----------------------+
|            BACKEND API                |   |               AMAZON S3               |
|  Node.js / Express.js / Sequelize     |   |  Bucket: Private, Versioned, TLS-only |
|  - Auth & Invitation verification     |   |                                       |
|  - Quota reservation (ACTIVE)         |   |  Prefixes:                            |
|  - Folder & File metadata management  |   |  - incoming/{userId}/{uploadId}       |
|  - Generate S3 Presigned POST         |   |  - objects/{userId}/{fileId}          |
+-------------------+-------------------+   |  - reports/{uploadId}.json            |
                    |                       +---------------+-----------------------+
                    | 4. Metadata sync                      |
                    v                                       | 5. s3:ObjectCreated
+-------------------+-------------------+                   v    (incoming/ only)
|              DATABASE                 |   +---------------+-----------------------+
|  MySQL (InnoDB Engine)                |   |            AWS LAMBDA                 |
|  - Users, Sessions, Invitations       |   |  s3-file-validator (Outside VPC)      |
|  - Folders, Files, Jobs, AuditLogs    |   |  - Check magic bytes (PDF, Image, TXT)|
|  - ReconciliationFindings             |   |  - Write JSON validation report       |
+-------------------+-------------------+   +---------------+-----------------------+
                    ^                                       |
                    | 7. Fenced DB Transaction              | 6. SQS Dead-Letter
                    |    (READY / REJECTED)                 |    Queue on failure
                    |                                       v
+-------------------+-------------------+   +---------------+-----------------------+
|           DURABLE WORKER              |   |            AMAZON SQS                 |
|  Separate Node.js Process             |   |  validator-dlq (Encrypted, 14-day)    |
|  - Claim job via Lease & Fencing      |<--+---------------------------------------+
|  - Exact-version S3 copy/delete       |
|  - Exponential Jittered Retry         |
|  - Quota settlement & Reconciliation  |
+---------------------------------------+
```

#### Các thành phần chính trong kiến trúc:
1. **Frontend Client (React SPA)**: Quản trị giao diện, điều hướng thư mục phân cấp, xử lý form đăng ký/đăng nhập, tải tệp trực tiếp lên S3 qua thanh tiến trình thời gian thực.
2. **Backend API (Express.js)**: Điểm tiếp nhận yêu cầu nghiệp vụ, quản lý phiên máy chủ, cấp phát Presigned POST Policy có ràng buộc điều kiện (dung lượng tối đa 50 MiB, đúng prefix `incoming/`, đúng loại tệp), cập nhật tạm ứng hạn mức.
3. **Amazon S3 Storage**: Lưu trữ đối tượng phân cấp với hai vùng tiền tố rõ ràng:
   * `incoming/`: Nơi chứa tệp thô người dùng tải lên, chỉ tồn tại tạm thời chờ kiểm định.
   * `objects/`: Nơi lưu trữ tệp chính thức đã vượt qua khâu kiểm định nội dung.
   * `reports/`: Nơi chứa kết quả kiểm định JSON của Lambda.
4. **AWS Lambda Validator**: Hàm Serverless được kích hoạt tự động bởi sự kiện `s3:ObjectCreated` tại tiền tố `incoming/`. Lambda đọc các byte đầu tiên để xác thực magic number của tệp (PDF `%PDF-`, PNG `\x89PNG`, JPEG `\xFF\xD8\xFF`, TXT UTF-8 hợp lệ), ghi kết quả vào `reports/`.
5. **Durable Background Worker**: Tiến trình nền chạy độc lập với API, quét các tác vụ trong bảng `Jobs`, thực hiện tranh chấp và gia hạn khóa (lease fencing), đọc báo cáo từ Lambda để chuyển tệp từ `incoming/` sang `objects/`, hoàn tất trạng thái `READY` và ghi nhận lịch sử kiểm toán.
6. **Amazon SQS (Dead-Letter Queue)**: Hứng các sự kiện Lambda thất bại hoặc tệp không thể xử lý để bảo đảm không thất lạc thông tin lỗi.

---

### 4. Vai trò của Công nghệ và Quyết định Thiết kế

| Công nghệ | Vai trò trong hệ thống | Quyết định kỹ thuật cốt lõi |
| :--- | :--- | :--- |
| **React 18 + TypeScript + Vite** | Giao diện người dùng SPA | Kiểm soát chặt chẽ kiểu dữ liệu, thời gian tải trang nhanh, thư viện Ant Design 5 chuẩn hóa giao diện trực quan. |
| **Express.js (Node >=20)** | Máy chủ API RESTful | Kiến trúc module hóa (Routes, Controllers, Services), tích hợp Helmet bảo mật HTTP headers, middleware xác thực tập trung. |
| **MySQL + Sequelize ORM** | Cơ sở dữ liệu quan hệ | Đảm bảo tính toàn vẹn quan hệ (Foreign keys, Indexes), quản lý transaction ACID khi chuyển đổi trạng thái tệp và bảo đảm tính nhất quán của hạn mức (Quota). |
| **Amazon S3** | Kho lưu trữ đối tượng chính | Bật tính năng Versioning bảo vệ dữ liệu, tắt toàn bộ Public Access, chỉ truy cập thông qua Presigned URL có thời hạn (15 phút). |
| **AWS Lambda** | Kiểm định tệp tự động | Đặt ngoài VPC để tối ưu chi phí và tốc độ khởi động; chỉ được cấp quyền đọc `incoming/` và ghi `reports/`. |
| **Amazon SQS** | Hàng đợi sự kiện lỗi | Lưu vết các sự kiện tải lên không xử lý được, mã hóa dữ liệu nghỉ (SSE), thời gian lưu trữ 14 ngày. |
| **Terraform (v1.10.5)** | Quản lý hạ tầng đám mây | Khai báo tài nguyên AWS có cấu trúc tại Singapore (`ap-southeast-1`), kiểm soát an toàn bằng các quy tắc cấm đột biến nguy hiểm (no public DB, IMDSv2, TLS-only). |
| **GitHub Actions** | Tự động hóa CI/CD | Pipeline kiểm thử đa bước: Lint, Unit tests, Mock DB tests, Terraform check, đóng gói artifact kèm mã SHA-256 xác thực. |

---

### 5. Kế hoạch triển khai phù hợp chương trình 6 tuần

Kế hoạch thực hiện dự án được thiết kế đồng bộ với lộ trình 6 tuần:

* **Tuần 1: Nghiên cứu & Thiết lập Kiến trúc nền tảng**
  * Làm quen với AWS Console, AWS CLI, mô hình bảo mật IAM và dịch vụ điện toán EC2.
  * Phân tích yêu cầu bài toán quản lý tệp, xây dựng sơ đồ kiến trúc tổng thể và mô hình dữ liệu (Users, Sessions, Folders, Files).
* **Tuần 2: Phát triển Hệ thống Xác thực & Quản lý Thư mục**
  * Hiện thực hóa cơ chế Opaque Session lưu trữ trong MySQL, gửi qua HttpOnly Cookie.
  * Xây dựng middleware bảo vệ CSRF (`X-CSRF-Token`) và cơ chế mã mời (Invitation-only Registration).
  * Phát triển các API phân cấp thư mục (Tạo, Đổi tên, Di chuyển, Xóa thư mục rỗng) và giao diện Drive trên React.
* **Tuần 3: Thiết kế Luồng Direct Upload S3 & Lambda Validator**
  * Thiết kế quy trình cấp phát Presigned POST Policy kèm cơ chế giữ chỗ dung lượng (Quota Reservation).
  * Viết mã nguồn hàm AWS Lambda `s3-file-validator` kiểm tra chữ ký nhị phân (magic bytes) cho tệp PDF, Ảnh và Văn bản.
  * Tích hợp sự kiện S3 Event Notification kích hoạt Lambda và ghi báo cáo chuẩn tắc `reports/{uploadId}.json`.
* **Tuần 4: Hoàn thiện Durable Worker & Quản lý Vòng đời Tệp**
  * Xây dựng tiến trình nền `worker.js` độc lập với cơ chế khóa hàng đợi (DB-backed lease fencing) và thử lại có độ trễ ngẫu nhiên (jittered exponential backoff).
  * Hiện thực hóa tính năng Thùng rác (Trash), Khôi phục (Restore), Hẹn giờ xóa vĩnh viễn (7-day Delayed Purge) và Xóa S3 theo đúng phiên bản (`(Key, VersionId)` batch delete).
  * Phát triển dịch vụ đối soát dữ liệu (Reconciliation) phát hiện tệp mồ côi và tự động cân bằng dung lượng sử dụng.
* **Tuần 5: Kiểm thử Toàn diện, Hạ tầng Terraform & Tổng kết Workshop**
  * Xây dựng kịch bản kiểm thử tự động: 82 bài test backend, 6 bài test frontend, kiểm thử cấu trúc HCL Terraform.
  * Đóng gói mã nguồn hạ tầng Terraform và quy trình CI/CD có cổng phê duyệt thủ công (Manual Gates).
  * Soạn thảo tài liệu hướng dẫn thực hành Workshop và hoàn thiện báo cáo kỹ thuật.
* **Tuần 6: Giám sát Vận hành CloudWatch, Đóng gói Dự án & Báo cáo Tổng kết**
  * Cấu hình Amazon CloudWatch Logs, Metrics và Billing Alarm (ngưỡng 75 USD) kèm cảnh báo Amazon SNS.
  * Kiểm thử khả năng phục hồi lỗi của hàng đợi SQS DLQ và xác minh cơ chế Lease Fencing chống tranh chấp dữ liệu.
  * Hoàn thiện tài liệu hướng dẫn thực hành Workshop cho Cloud File Manager, đóng gói toàn bộ mã nguồn và hoàn thành Báo cáo tổng kết thực tập.

---

### 6. Ước tính Chi phí Hạ tầng (Budget Estimation)



| Thành phần dịch vụ | Cấu hình đề xuất | Ước tính chi phí / tháng (USD) | Ghi chú & Giả định |
| :--- | :--- | :--- | :--- |
| **Amazon EC2** | 1 x `t4g.small` (hoặc `t3.micro`), 20 GB gp3 EBS Encrypted | ~$15.00 - $18.00 | Máy chủ chạy Nginx reverse proxy, Node.js API và Worker tiến trình nền. |
| **Amazon RDS (MySQL)** | 1 x `db.t4g.micro`, Single-AZ, 20 GB gp3 Storage | ~$17.00 - $20.00 | Cơ sở dữ liệu chính, sao lưu tự động 7 ngày, đặt trong subnet riêng tư. |
| **Amazon S3** | Standard Storage (~10 - 20 GB), PUT/GET requests | ~$0.50 - $1.50 | Chi phí lưu trữ nhị phân và yêu cầu API; bật Versioning. |
| **AWS Lambda** | 128 MB RAM, ~10.000 invocations / tháng | $0.00 (Free Tier) | Nằm trọn trong định mức miễn phí 1 triệu request/tháng của AWS Free Tier. |
| **Amazon SQS & CloudWatch** | 1 DLQ Queue, CloudWatch Logs & Metrics | ~$1.00 - $3.00 | Ghi log JSON có cấu trúc, lưu trữ chỉ số RAM/Disk và cảnh báo. |
| **Data Transfer & Gateway EP** | S3 Gateway Endpoint (Miễn phí), Data Egress thấp | ~$1.00 - $2.00 | Lưu lượng truy cập S3 trong nội bộ AWS qua Gateway Endpoint không tốn phí. |
| **Tổng cộng ước tính** | | **~$35.00 - ~$45.00 / tháng** | *Ngưỡng cảnh báo ngân sách (Budget Alert) thiết lập tại mức 75.00 USD/tháng.* |

*Lưu ý:* Hệ thống hiện tại đã được cấu hình cảnh báo chi phí qua CloudWatch Alarms và SNS Topic. Khi triển khai trên tài khoản thực tế, cần bổ sung bảng tính chi tiết từ công cụ **AWS Pricing Calculator** để có con số chính xác theo lưu lượng thực tế.

---

### 7. Đánh giá Rủi ro và Biện pháp Giảm thiểu (Risk Assessment)

| Rủi ro kỹ thuật | Mức độ | Khả năng | Biện pháp giảm thiểu đã thiết kế |
| :--- | :---: | :---: | :--- |
| **Người dùng tải lên tệp độc hại hoặc giả mạo extension** | Cao | Trung bình | Kiểm tra magic bytes thực tế bằng Lambda độc lập ngoài VPC trước khi chuyển tệp vào thư mục chính thức (`objects/`). Tệp sai quy chuẩn bị từ chối ngay lập tức và hoàn trả hạn mức. |
| **Tải lên dở dang gây chiếm dụng dung lượng ảo (Ghost Quota)** | Trung bình | Cao | Thiết lập hạn thời gian cho phiên tải lên (15 phút). Dịch vụ đối soát định kỳ (`Reconciliation`) tự động quét và giải phóng các hạn mức quá hạn chưa hoàn tất. |
| **Xung đột phiên bản khi xóa tệp trên S3 có bật Versioning** | Cao | Thấp | Sử dụng cơ chế xóa chính xác từng cặp `(Key, VersionId)` bao gồm cả delete markers, sau đó đối soát lại bằng API kiểm kê trước khi đánh dấu xóa hoàn toàn (`PURGED`) trong DB. |
| **Worker bị sự cố (crash) khi đang xử lý tác vụ tải tệp** | Cao | Thấp | Cơ chế khóa phân tán dựa trên cơ sở dữ liệu (`lease_token`, `lease_expires_at`). Nếu worker chết, khóa hết hạn sau thời gian timeout và worker khác sẽ tiếp quản an toàn. |
| **Tấn công giả mạo yêu cầu qua trình duyệt (CSRF)** | Trung bình | Trung bình | Toàn bộ các yêu cầu ghi dữ liệu (POST, PUT, DELETE) bắt buộc phải có header `X-CSRF-Token` khớp với token trong phiên làm việc của máy chủ. |
| **Chi phí điện toán đám mây vượt tầm kiểm soát** | Trung bình | Thấp | Thiết lập ngân sách cảnh báo tự động ở mức 75 USD qua CloudWatch Billing Alarm gửi email qua SNS; vòng đời S3 tự động dọn dẹp các tệp tạm trong `incoming/` sau 30 ngày. |

---

### 8. Kết quả Kỳ vọng (Expected Outcomes)

1. **Về mặt Kỹ thuật & Kiến trúc**:
   * Thiết lập thành công mô hình lưu trữ đám mây phân tầng hiện đại, chứng minh tính hiệu quả vượt trội của luồng tải trực tiếp S3 Presigned POST so với cách truyền thống.
   * Xây dựng được hệ thống xử lý bất đồng bộ bền bỉ với đầy đủ cơ chế an toàn: kiểm định tệp, hàng đợi có khóa chống tranh chấp, và quy trình đối soát dữ liệu tự động.
2. **Về mặt Ứng dụng & Người dùng**:
   * Cung cấp một giao diện web hoàn chỉnh, phản hồi nhanh, dễ sử dụng cho phép quản lý thư mục, tải tệp, xem lịch sử thao tác và khôi phục tệp từ thùng rác một cách an toàn.
3. **Về mặt Đào tạo & Workshop**:
   * Đóng vai trò là bài thực hành mẫu mực cho sinh viên và kỹ sư muốn làm chủ kỹ năng lập trình full-stack kết hợp kiến trúc đám mây AWS nâng cao, quản lý hạ tầng bằng Terraform và tự động hóa với CI/CD.
