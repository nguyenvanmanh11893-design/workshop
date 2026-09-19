---
title: "Tuần 6 - Giám sát CloudWatch, Đóng gói Dự án và Báo cáo Thực tập Cloud File Manager"
weight: 6
chapter: false
pre: "<b>1.6. </b>"
---

# WORKLOG TUẦN 6

**Thời gian:** 07/09/2026 - 13/09/2026

## Mục tiêu tuần 6:

- Tìm hiểu Amazon CloudWatch và AWS CloudTrail phục vụ giám sát ứng dụng Cloud-native.
- Cấu hình Log, Metric và Alarm giám sát máy chủ API, hàng đợi SQS và tiến trình Durable Worker.
- Thiết lập cảnh báo ngân sách (Billing Alarm) 75 USD qua CloudWatch và Amazon SNS.
- Tổng hợp toàn bộ kiến thức AWS và hoàn tất việc kiểm thử, đóng gói dự án **Cloud File Manager**.
- Soạn thảo tài liệu hướng dẫn thực hành Workshop và hoàn thiện Báo cáo tổng kết thực tập.

## Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | --- | --- | --- | --- |
| 2 - 3 | Nghiên cứu Amazon CloudWatch, CloudTrail và tích hợp logging cho API, Worker của Cloud File Manager | 07/09/2026 | 08/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| 4 | Cấu hình CloudWatch Metrics, Alarms và thiết lập Billing Alarm 75 USD với SNS Email Alert | 09/09/2026 | 09/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| 5 | Kiểm thử xử lý lỗi ngoại lệ Dead-Letter Queue (SQS) và xác minh cơ chế Lease Fencing dưới tải song song | 10/09/2026 | 10/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| 6 - 7 | Hoàn thiện tài liệu thực hành chi tiết cho Workshop Cloud File Manager (các mục 5.1 đến 5.7) | 11/09/2026 | 12/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| Chủ nhật | Đóng gói toàn diện mã nguồn, cấu hình Terraform và hoàn thiện Báo cáo tổng kết thực tập | 13/09/2026 | 13/09/2026 | — |

### Nội dung triển khai hệ thống

- Ứng dụng quản lý tệp trên đám mây **Cloud File Manager**.
- Backend sử dụng Node.js, Express.js và Sequelize ORM (MySQL).
- Frontend xây dựng với React 18, TypeScript, Vite và Ant Design 5.
- Tải tệp trực tiếp lên Amazon S3 qua Presigned POST Policy kèm cơ chế giữ chỗ hạn mức (Quota Reservation).
- Hàm AWS Lambda kiểm định magic bytes nhị phân tự động kích hoạt bởi sự kiện S3.
- Tiến trình nền Durable Worker xử lý bất đồng bộ, khóa phân tán Lease Fencing và thử lại Jittered Backoff.
- Quản lý vòng đời tệp, thùng rác, xóa vĩnh viễn theo phiên bản S3 và đối soát hạn mức tự động (Reconciliation).

### Kiến trúc AWS triển khai

```text
Client (React 18 + Vite)
→ Backend API (Express + Sequelize) & Amazon S3 (Direct Upload)
→ AWS Lambda (Magic Bytes Validator) & Amazon SQS (Dead-Letter Queue)
→ MySQL Database & Durable Background Worker
→ Amazon CloudWatch (Logs, Metrics, Billing Alarms)
```

Đồng thời xác định rõ vai trò của Amazon S3, Amazon VPC, Security Groups, AWS IAM, AWS Systems Manager Parameter Store, Terraform IaC và GitHub Actions CI/CD.

## Kết quả đạt được tuần 6:

- Hiểu và ứng dụng thành công Amazon CloudWatch và AWS CloudTrail vào giám sát vận hành thực tế.
- Thiết lập hoàn chỉnh cơ chế giám sát Logs, Metrics và Alarms cho toàn bộ hệ thống Cloud File Manager.
- Cấu hình thành công Billing Alarm 75 USD bảo đảm an toàn chi phí điện toán đám mây.
- Đóng gói hoàn chỉnh dự án Cloud File Manager, kiểm thử toàn diện 82 test backend, 6 test frontend và kiểm tra Terraform HCL.
- Hoàn thiện tài liệu hướng dẫn thực hành Workshop và hoàn tất Báo cáo tổng kết thực tập đúng tiến độ.
