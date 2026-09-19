---
title: "Kiểm định tệp với Lambda và Durable Worker"
date: 2026-09-18T23:00:00+07:00
weight: 3
chapter: false
pre: "<b>5.4.3 </b>"
---





---

### 1. Mục đích
* Hiểu rõ cơ chế kích hoạt sự kiện tự động `s3:ObjectCreated` của Amazon S3 tới hàm AWS Lambda.
* Nắm bắt giải thuật kiểm tra chữ ký nhị phân (magic bytes) của Lambda để ngăn chặn các tệp giả mạo extension hoặc chứa mã thực thi nguy hiểm.
* Quan sát tiến trình Durable Worker quét tác vụ `FINALIZE_UPLOAD`, tranh chấp khóa an toàn và thực thi giao dịch cơ sở dữ liệu để đưa tệp sang trạng thái `READY`.

---

### 2. Sơ đồ Tuần tự Xử lý Bất đồng bộ

<!-- > [CHÈN HÌNH 5-05: Sơ đồ tuần tự (Sequence Diagram) của luồng Direct Upload, Lambda Validation và Worker Finalize — thể hiện tương tác giữa S3, Lambda, SQS, MySQL và Worker.] -->

```
[Amazon S3]                  [AWS Lambda]                 [MySQL DB]             [Durable Worker]
     |                            |                           |                          |
     | (Tệp lưu tại incoming/)    |                           |                          |
     | 1. S3 Event Notification   |                           |                          |
     |--------------------------->|                           |                          |
     |                            | (Đọc 512 bytes đầu tệp)   |                          |
     |                            | (Kiểm tra Magic Bytes)    |                          |
     | 2. Ghi JSON Report         |                           |                          |
     |    reports/{uploadId}.json |                           |                          |
     |<---------------------------|                           |                          |
     |                            | (Đẩy SQS DLQ nếu lỗi)     |                          |
     |                            +-------------------------->| (Tạo Job FINALIZE)       |
     |                                                        |                          |
     |                                                        | 3. Poll Jobs (Due)       |
     |                                                        |<-------------------------|
     |                                                        | 4. Fenced Lease Claim    |
     |                                                        |    (Gia hạn lease_token) |
     |                                                        |------------------------->|
     |                                                                                   |
     | 5. Đọc reports/{uploadId}.json                                                    |
     |<----------------------------------------------------------------------------------|
     |                                                                                   |
     | 6. S3 CopyObject (Phiên bản chính xác từ incoming/ sang objects/)                 |
     |<----------------------------------------------------------------------------------|
     |                                                                                   |
     |                                                        | 7. Fenced DB Transaction:|
     |                                                        |    - Tạo bản ghi File    |
     |                                                        |      (status = READY)    |
     |                                                        |    - Quyết toán Quota    |
     |                                                        |    - Ghi AuditLog        |
     |                                                        |    - Hoàn tất Job        |
     |                                                        |<-------------------------|
```

---

### 3. Chi tiết Giải thuật Kiểm định Nội dung của Lambda

Mã nguồn hàm Lambda tại `lambdas/s3-file-validator/index.js` thực hiện đọc một phần nhỏ dữ liệu nhị phân đầu tệp để phân loại:

```javascript
// Trích xuất chữ ký nhị phân (Magic Bytes Check)
function detectMimeType(buffer) {
  // PDF: %PDF- (0x25 0x50 0x44 0x46 0x2D)
  if (buffer.length >= 5 && buffer.toString('ascii', 0, 5) === '%PDF-') {
    return 'application/pdf';
  }
  // PNG: \x89PNG\r\n\x1a\n (0x89 0x50 0x4E 0x47 0x0D 0x0A 0x1A 0x0A)
  if (buffer.length >= 8 &&
      buffer[0] === 0x89 && buffer[1] === 0x50 && 
      buffer[2] === 0x4e && buffer[3] === 0x47) {
    return 'image/png';
  }
  // JPEG: \xFF\xD8\xFF
  if (buffer.length >= 3 && 
      buffer[0] === 0xff && buffer[1] === 0xd8 && buffer[2] === 0xff) {
    return 'image/jpeg';
  }
  // TXT: Kiểm tra tính hợp lệ của bảng mã UTF-8 / ASCII
  if (isValidUtf8Text(buffer)) {
    return 'text/plain';
  }
  return 'application/octet-stream';
}
```

#### Cấu trúc Báo cáo Thẩm định Chuẩn tắc (`reports/{uploadId}.json`):
```json
{
  "report_id": "rep_9f83a21b",
  "upload_id": "upl_4a12c8e0",
  "user_id": 1,
  "s3_key": "incoming/1/upl_4a12c8e0",
  "s3_version_id": "3/L4kqtJlcpXroDTDmJ+rmSpXd3dIbrHY",
  "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "detected_mime_type": "application/pdf",
  "verdict": "PASSED",
  "validator_version": "1.0.0",
  "validated_at": "2026-09-18T23:05:00.000Z"
}
```

---

### 4. Quy trình Hoàn tất của Durable Worker

Tiến trình nền `worker.js` đảm bảo tính bền vững và nhất quán dữ liệu qua 4 bước:

#### Bước 1: Tranh chấp Khóa tác vụ (Lease Fencing)
Worker sử dụng câu lệnh SQL nguyên tử (Atomic Update) trên bảng `Jobs` để chiếm quyền xử lý mà không bị đụng độ với các tiến trình worker khác:
```sql
UPDATE Jobs 
SET lease_token = 'worker-node-1-uuid', 
    lease_expires_at = DATE_ADD(NOW(), INTERVAL 60 SECOND),
    attempts = attempts + 1
WHERE id = 101 
  AND status = 'PENDING'
  AND (lease_expires_at IS NULL OR lease_expires_at < NOW());
```

#### Bước 2: Thẩm tra Báo cáo và Sao chép Đối tượng S3
Worker nạp file báo cáo JSON từ S3. Nếu `verdict === 'PASSED'`, worker thực hiện lệnh `CopyObject` nội bộ trên S3:
* **Nguồn**: `incoming/1/upl_4a12c8e0` (Kèm theo `VersionId` nguồn chính xác).
* **Đích**: `objects/1/file_77b31a29`.
* Lưu lại `VersionId` của đối tượng mới được tạo ra ở thư mục đích.

#### Bước 3: Mở Giao dịch Cơ sở dữ liệu Cập nhật Nghiệp vụ (Fenced DB Transaction)
Nếu giao dịch S3 thành công, worker mở một Transaction trên MySQL thực thi đồng thời:
1. Thêm bản ghi mới vào bảng `Files` với `status = 'READY'`, đường dẫn lưu trữ và phiên bản S3 chính xác.
2. Cập nhật bảng `UploadSessions` sang `COMPLETED`.
3. Quyết toán hạn mức người dùng: `Users.used_bytes = Users.used_bytes + file_size`.
4. Ghi một bản ghi nhật ký kiểm toán vào bảng `AuditLogs` với hành động `FILE_UPLOAD`.
5. Đánh dấu bản ghi trong bảng `Jobs` là `COMPLETED`.

#### Bước 4: Xử lý Trường hợp Tệp bị Từ chối (Rejection Handling)
Nếu `verdict === 'REJECTED'` (ví dụ tệp giả mạo đuôi file hoặc chứa payload độc hại):
* Tệp nhị phân không được chuyển sang `objects/`.
* Worker giải phóng dung lượng đã đặt trước: `UploadSessions` chuyển thành `FAILED`.
* Hạn mức của người dùng không bị trừ oan.

---

### 5. Cách thức Kiểm tra và Xác minh

#### Kiểm tra 1: Quan sát Log của Tiến trình Worker
Tại cửa sổ Terminal 2 (đang chạy `npm run worker`), bạn sẽ thấy chuỗi log JSON có cấu trúc hiển thị:
```text
[worker] Claimed job ID: 101, type: FINALIZE_UPLOAD, lease_token: worker-node-1-uuid
[worker] Read validation report for upload upl_4a12c8e0: PASSED (application/pdf)
[worker] Copied object to objects/1/file_77b31a29 with destination VersionId: abc123xyz
[worker] Database transaction committed. File ID: 45 is now READY.
[worker] Quota updated: +2097152 bytes for user 1.
[worker] Job 101 marked as COMPLETED.
```

#### Kiểm tra 2: Kiểm tra Giao diện Drive trên Trình duyệt
Quay lại trình duyệt tại `http://localhost:5173/drive`:
* Danh sách tệp tự động cập nhật hiển thị tệp `baocao.pdf` với biểu tượng tài liệu PDF màu đỏ đặc trưng.
* Kích thước tệp, ngày tải lên và người sở hữu hiển thị chính xác.
* Thanh hạn mức lưu trữ (Quota widget) ở thanh bên trái (Sidebar) tăng tương ứng với dung lượng tệp vừa tải lên.
