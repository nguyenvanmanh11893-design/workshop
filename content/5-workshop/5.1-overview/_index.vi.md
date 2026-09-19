---
title: "Tổng quan Workshop"
date: 2026-09-18T23:00:00+07:00
weight: 1
chapter: false
pre: "<b>5.1. </b>"
---




---

### 1. Bối cảnh và Bài toán kỹ thuật

Trong kỷ nguyên số hóa, nhu cầu lưu trữ và chia sẻ tệp tin cá nhân trên nền tảng đám mây ngày càng trở nên thiết yếu. Tuy nhiên, việc tự xây dựng một hệ thống lưu trữ tương tự Google Drive đặt ra nhiều thách thức hóc búa về mặt kiến trúc và vận hành:
* **Tải máy chủ khi tiếp nhận tệp lớn**: Nếu toàn bộ luồng dữ liệu nhị phân (binary payload) truyền tải qua API server, máy chủ sẽ nhanh chóng cạn kiệt RAM và tắc nghẽn luồng xử lý I/O mạng khi có nhiều người dùng đồng thời.
* **Nguy cơ tệp độc hại và giả mạo định dạng**: Ứng dụng không thể chỉ tin cậy vào phần mở rộng tệp (`.jpg`, `.pdf`) hay header `Content-Type` do client gửi lên. Cần một cơ chế độc lập, tự động để kiểm tra nội dung thực tế (magic bytes) trước khi cho phép lưu trữ chính thức.
* **Tính nhất quán giữa cơ sở dữ liệu và đám mây (Storage Consistency)**: Trong môi trường phân tán, các thao tác tải lên, đổi tên, xóa tệp hay di chuyển có thể gặp sự cố mạng bất cứ lúc nào. Nếu không có cơ chế bù trừ và xử lý bất đồng bộ tin cậy, dữ liệu giữa cơ sở dữ liệu quan hệ (metadata) và kho lưu trữ S3 sẽ bị lệch, dẫn đến tệp rác hoặc hiển thị sai hạn mức sử dụng.
* **Bảo mật phiên và cô lập dữ liệu nhiều người dùng**: Ngăn chặn tuyệt đối việc người dùng này truy cập hoặc xóa tệp của người dùng khác, bảo vệ ứng dụng khỏi các lỗ hổng phổ biến như Cross-Site Request Forgery (CSRF) và đánh cắp mã phiên (Session Hijacking).

Dự án **Cloud File Manager** được thiết kế nhằm cung cấp lời giải toàn diện cho các vấn đề trên, kết hợp giữa ứng dụng web React hiện đại, máy chủ API Node.js/Express, cơ sở dữ liệu quan hệ MySQL và kho lưu trữ đối tượng Amazon S3 có kiểm định Serverless bằng AWS Lambda.

---

### 2. Mục tiêu học tập 

Sau khi hoàn thành workshop này, người tham gia sẽ làm chủ được các kiến thức và kỹ năng sau:
1. **Nắm vững mô hình tải tệp trực tiếp (Direct-to-S3 Upload)**: Hiểu rõ cơ chế hoạt động của Presigned POST Policy, cách tạo chữ ký bảo mật từ backend và cách phía client gửi tệp trực tiếp lên Amazon S3 mà không cần đi qua máy chủ API.
2. **Thiết kế cơ chế kiểm soát hạn mức lưu trữ (Quota Reservation)**: Hiểu cách ứng dụng đặt trước dung lượng (reservation) an toàn, tránh race condition khi nhiều tệp được tải lên cùng lúc và tự động hoàn trả hạn mức khi tệp bị từ chối hoặc quá hạn.
3. **Lập trình Serverless Event-Driven trên AWS**: Xây dựng hàm AWS Lambda được kích hoạt tự động bởi sự kiện S3 (`s3:ObjectCreated`), phân tích chữ ký nhị phân (magic bytes) và tạo báo cáo kiểm định JSON.
4. **Xây dựng tiến trình nền bền vững (Durable Worker Pattern)**: Nắm bắt kỹ thuật quản lý hàng đợi tác vụ trên cơ sở dữ liệu (Database-backed Job Queue), cơ chế gia hạn khóa (Lease Fencing), thử lại với độ trễ ngẫu nhiên (Jittered Exponential Backoff) và đảm bảo tính lũy biến (Idempotency).
5. **Làm chủ vòng đời tệp tin và đối soát dữ liệu (Lifecycle & Reconciliation)**: Triển khai tính năng Thùng rác (Trash), hẹn giờ xóa vĩnh viễn (Delayed Purge), xóa S3 an toàn theo từng phiên bản (`(Key, VersionId)`) và viết script quét đối soát tệp mồ côi (Orphan Scanner).
6. **Thực hành bảo mật ứng dụng web nâng cao**: Thiết lập phiên Opaque Session lưu trên database kết hợp HttpOnly Cookie, bảo vệ chống CSRF bằng token động và xác thực đăng ký chỉ qua mã mời (Invitation-only).
7. **Tiếp cận Hạ tầng dưới dạng mã (IaC) và CI/CD**: Đọc hiểu cấu hình mạng VPC, subnet cô lập, S3 Gateway Endpoint, CloudWatch monitoring trong Terraform và quy trình tự động hóa kiểm thử trong GitHub Actions.

---

### 3. Chuẩn đầu ra của Workshop 

Người tham gia workshop sẽ thu được các sản phẩm và kết quả cụ thể:
* **Môi trường cục bộ vận hành trơn tru**: Chạy đồng thời hệ thống gồm Frontend React, Backend Express API, cơ sở dữ liệu MySQL và tiến trình nền Worker.
* **Bộ dữ liệu mẫu được khởi tạo**: Tài khoản quản trị, mã mời thành viên, các thư mục phân cấp và các tệp tải lên kiểm định thành công.
* **Bằng chứng kiểm thử hoàn chỉnh**: Tự chạy và kiểm chứng 82 bài unit test backend và 6 bài test frontend đạt 100% tỷ lệ pass.
* **Khả năng giải thích và phản biện kiến trúc**: Nắm vững lý do đưa ra các quyết định thiết kế (trade-offs) về mặt bảo mật, hiệu năng và chi phí.

---

### 4. Phạm vi Workshop và Sự Kết nối Lộ trình 6 tuần

Workshop được xây dựng theo tiến độ học tập và tích lũy kiến thức trong chương trình đào tạo 6 tuần cùng các kỹ thuật chuyên sâu:

| Tuần | Trọng tâm học tập nguồn | Ánh xạ vào Workshop Cloud File Manager | Module trong Workshop |
| :---: | :--- | :--- | :---: |
| **Tuần 1** | Nền tảng AWS, Console, CLI, IAM, EC2 | Thiết kế kiến trúc tổng thể, mô hình bảo mật và vai trò của các dịch vụ đám mây. | **Phần 5.1 & 5.2** |
| **Tuần 2** | Lưu trữ đám mây, Cơ sở dữ liệu và Mạng | Thiết lập MySQL, Sequelize migrations, Opaque Session, HttpOnly Cookie và phân cấp thư mục. | **Phần 5.3 & 5.4.1** |
| **Tuần 3** | Serverless, Tải tệp và Tích hợp S3 | Cơ chế Direct S3 Upload (Presigned POST), Quota Reservation và hàm AWS Lambda kiểm định magic bytes. | **Phần 5.4.2 & 5.4.3** |
| **Tuần 4** | Xử lý bất đồng bộ, Hàng đợi & Vòng đời tệp | Xây dựng Durable Worker, Lease Fencing, Retry, Thùng rác, Purge theo phiên bản S3 và Reconciliation. | **Phần 5.4.4 & 5.5** |
| **Tuần 5** | Kiểm thử, Tự động hóa IaC & CI/CD | Chạy bộ test suite toàn diện, phân tích cấu hình Terraform, quy trình CI/CD và xác minh an toàn. | **Phần 5.5 & 5.6** |
| **Tuần 6** | Giám sát Vận hành, Quản trị Rủi ro & Tổng kết | Cấu hình CloudWatch Logs/Metrics, cảnh báo Billing Alarm 75 USD với SNS, đóng gói mã nguồn và dọn dẹp tài nguyên. | **Phần 5.6 & 5.7** |

