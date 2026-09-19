---
title: "Xác thực và Tải tệp trực tiếp lên S3"
date: 2026-09-18T23:00:00+07:00
weight: 2
chapter: false
pre: "<b>5.4.2 </b>"
---





---

### 1. Mục đích
* Đăng ký tài khoản người dùng an toàn bằng mã mời (`WORKSHOP-INVITE-2026`).
* Thiết lập phiên đăng nhập Opaque Session qua HttpOnly Cookie và lấy mã bảo vệ CSRF.
* Tạo cấu trúc thư mục phân cấp trên giao diện Drive.
* Trực tiếp thực hiện quy trình tải tệp từ trình duyệt lên Amazon S3 bằng Presigned POST Policy và kiểm tra trạng thái giữ chỗ hạn mức (Quota Reservation).

---

### 2. Điều kiện Tiên quyết
* Cả hai tiến trình (`npm run dev` và `npm run worker`) đang hoạt động bình thường.
* Cơ sở dữ liệu đã có sẵn mã mời `WORKSHOP-INVITE-2026` được tạo tại **[Mục 5.3](../../5.3-prerequisites/)**.
* Chuẩn bị sẵn một tệp PDF hoặc hình ảnh (PNG/JPEG) có dung lượng nhỏ hơn 50 MiB để thử nghiệm tải lên.

---

### 3. Thao tác Thực hiện

#### Bước 1: Đăng ký Tài khoản Người dùng
1. Mở trình duyệt tại `http://localhost:5173/register`.
2. Điền thông tin đăng ký:
   * **Tên hiển thị**: `Nguyen Van A`
   * **Email**: `user1@example.com`
   * **Mật khẩu**: `Password@123`
   * **Mã mời (Invitation Token)**: `WORKSHOP-INVITE-2026`
3. Nhấn **Đăng ký**.

*Giải thích luồng hệ thống:* Backend kiểm tra token trong bảng `Invitations`. Nếu hợp lệ và chưa hết lượt sử dụng (`uses_count < max_uses`), hệ thống băm mật khẩu bằng `bcryptjs` (salt rounds = 10), tạo bản ghi trong bảng `Users`, tăng `uses_count` lên 1 và tự động gán hạn mức mặc định 1 GiB (`1073741824 bytes`).

<!-- > [CHÈN HÌNH 5-03: Giao diện đăng nhập và quản lý tệp trên Drive — thể hiện danh sách thư mục, bảng tệp và widget hiển thị hạn mức lưu trữ.] -->

#### Bước 2: Đăng nhập và Kiểm tra Phiên
1. Truy cập `http://localhost:5173/login`, nhập email và mật khẩu vừa tạo, nhấn **Đăng nhập**.
2. Sau khi đăng nhập thành công, trình duyệt chuyển hướng vào màn hình chính `/drive`.

*Phân tích bảo mật tại DevTools (F12 -> Application -> Cookies):*
* Bạn sẽ thấy cookie phiên `connect.sid` được gắn cờ `HttpOnly` và `SameSite=Lax`. Mã JavaScript trên trình duyệt hoàn toàn không thể đọc được giá trị này, ngăn chặn nguy cơ tấn công đánh cắp cookie qua XSS.
* Ứng dụng tự động gửi yêu cầu GET tới `/api/auth/csrf` để nhận mã token ngẫu nhiên và lưu trong bộ nhớ client để đính kèm vào header `X-CSRF-Token` cho toàn bộ các yêu cầu HTTP sau.

#### Bước 3: Tạo Thư mục Phân cấp
1. Tại trang Drive, nhấn nút **+ Thư mục mới**.
2. Nhập tên thư mục: `Tai lieu Workshop` và nhấn **Xác nhận**.
3. Nhấp đúp chuột vào thư mục vừa tạo để điều hướng vào bên trong. Thanh breadcrumb trên đầu trang sẽ hiển thị: `Drive > Tai lieu Workshop`.

#### Bước 4: Khởi tạo Phiên Tải tệp và Upload Trực tiếp lên S3
1. Nhấn nút **Tải tệp lên** (Upload) để mở hộp thoại tải tệp.
2. Kéo thả hoặc chọn tệp kiểm thử (ví dụ: `baocao.pdf`, kích thước ~2 MiB).

<!-- > [CHÈN HÌNH 5-04: Hộp thoại tải tệp và thanh tiến trình Direct S3 Upload — minh họa việc gửi tệp trực tiếp lên endpoint S3.] -->

---

### 4. Phân tích Luồng Kỹ thuật Phía sau Thao tác Tải tệp

Quy trình tải tệp diễn ra theo 3 giai đoạn nghiêm ngặt:

```
[Trình duyệt Client]                   [Backend API]                    [Amazon S3]
        |                                     |                               |
        | 1. POST /api/files/upload-session   |                               |
        |    { filename, size, mime, folder } |                               |
        |------------------------------------>|                               |
        |                                     | (Kiểm tra Quota: used + size) |
        |                                     | (Tạo UploadSession: ACTIVE)   |
        |                                     | (Tạo S3 Presigned POST Data)  |
        | 2. Phản hồi Presigned POST Policy   |                               |
        |<------------------------------------|                               |
        |                                                                     |
        | 3. POST multipart/form-data (trực tiếp lên S3)                      |
        |    [Fields: key, policy, X-Amz-Signature, file payload]            |
        |-------------------------------------------------------------------->|
        |                                                                     |
        | 4. HTTP 204 No Content (Tải lên S3 thành công)                      |
        |<--------------------------------------------------------------------|
```

1. **Giai đoạn 1 (Request Upload Session)**:
   * Trình duyệt gửi thông tin metadata của tệp tới API: `POST /api/files/upload-session`.
   * Backend kiểm tra hạn mức người dùng: Nếu `used_bytes + reserved_bytes + size > max_quota`, API lập tức trả về lỗi `400 Quota Exceeded`.
   * Nếu hợp lệ, backend tạo bản ghi `UploadSessions` ở trạng thái `ACTIVE` (được giữ chỗ tạm thời), đồng thời dùng AWS SDK tạo Presigned POST Policy với thời hạn 15 phút, ép buộc tiền tố lưu trữ là `incoming/${userId}/${uploadId}`.
2. **Giai đoạn 2 (Direct Upload to S3)**:
   * Trình duyệt nhận được địa chỉ endpoint S3 cùng các trường chữ ký bảo mật.
   * Client khởi tạo một đối tượng `FormData` và sử dụng `XMLHttpRequest` để POST trực tiếp lên Amazon S3, cập nhật thanh tiến trình phần trăm (Progress Bar) mượt mà cho người dùng.
3. **Giai đoạn 3 (S3 Confirmation)**:
   * Amazon S3 tiếp nhận tệp, xác thực chữ ký số hợp lệ và lưu tệp tại `incoming/${userId}/${uploadId}`.
   * S3 trả về mã trạng thái HTTP `204 No Content` cho client.

---

### 5. Cách thức Kiểm tra và Xác minh

#### Kiểm tra 1: Kiểm tra bản ghi Giữ chỗ Hạn mức trong Cơ sở dữ liệu
Mở công cụ dòng lệnh MySQL và kiểm tra trạng thái phiên tải tệp:

```sql
SELECT id, user_id, filename, file_size, status, created_at 
FROM UploadSessions 
ORDER BY id DESC LIMIT 1;
```
*Kết quả mong đợi:*
Bản ghi hiển thị `status` là `ACTIVE`, cho thấy dung lượng tệp đã được đặt trước thành công trong hệ thống.

#### Kiểm tra 2: Kiểm tra Tệp Tạm trên Amazon S3
Sử dụng AWS CLI hoặc Console kiểm tra tiền tố `incoming/`:

```bash
aws s3 ls s3://example-cloud-file-manager-bucket-local/incoming/
```
*Kết quả mong đợi:*
Tệp nhị phân đã xuất hiện an toàn tại đúng tiền tố `incoming/{userId}/{uploadId}`, sẵn sàng cho khâu thẩm định tiếp theo của AWS Lambda.
