---
title: "Vòng đời tệp, Thùng rác và Đối soát dữ liệu"
date: 2026-09-18T23:00:00+07:00
weight: 4
chapter: false
pre: "<b>5.4.4 </b>"
---





---

### 1. Mục đích
* Trải nghiệm cơ chế tải tệp xuống thông qua Presigned Download URL có thời hạn ngắn (15 phút) thay vì mở truy cập S3 công khai.
* Thao tác đưa tệp vào Thùng rác (Trash) và hiểu cơ chế hẹn giờ xóa vĩnh viễn sau 7 ngày (7-day Delayed Purge).
* Thực hiện khôi phục tệp về thư mục làm việc và kích hoạt lệnh xóa vĩnh viễn chủ động.
* Nắm vững giải thuật xóa S3 an toàn theo từng phiên bản (`(Key, VersionId)`) và quy trình đối soát dữ liệu (Reconciliation) ngăn chặn lệch hạn mức lưu trữ.

---

### 2. Thao tác Thực hành Từng bước

#### Bước 1: Tải tệp xuống an toàn (Download File)
1. Tại danh sách tệp trên trang `/drive`, nhấn vào biểu tượng **Tùy chọn** (3 dấu chấm) bên cạnh tệp `baocao.pdf`.
2. Chọn **Tải xuống** (Download).

*Phân tích luồng kỹ thuật:*
* Client gửi yêu cầu `GET /api/files/45/download`.
* Backend kiểm tra quyền sở hữu (`req.user.id === file.user_id`) và đảm bảo trạng thái tệp đang là `READY`.
* Backend sử dụng AWS SDK tạo ra một **Presigned GET URL** có thời hạn hiệu lực chính xác 15 phút, đính kèm header `response-content-disposition: attachment; filename="baocao.pdf"`.
* Trình duyệt tự động tải tệp trực tiếp từ Amazon S3 mà không làm tiêu hao băng thông của máy chủ backend.

#### Bước 2: Chuyển tệp vào Thùng rác (Move to Trash)
1. Nhấn vào tùy chọn của tệp `baocao.pdf` và chọn **Chuyển vào thùng rác**.
2. Một thông báo Ant Design xuất hiện xác nhận: *"Tệp đã được chuyển vào thùng rác"*.
3. Tệp biến mất khỏi cây thư mục Drive chính và xuất hiện tại mục **Thùng rác (Trash)** trên thanh menu bên trái (`http://localhost:5173/trash`).

<!-- > [CHÈN HÌNH 5-06: Giao diện Thùng rác (Trash) và các thao tác Khôi phục / Xóa vĩnh viễn — thể hiện danh sách tệp đã xóa, thời gian tự động xóa và nút thao tác.] -->

*Phân tích quy tắc nghiệp vụ trong cơ sở dữ liệu:*
* Bản ghi của tệp được cập nhật: `status = 'TRASHED'` và `trashed_at = NOW()`.
* **Quan trọng về hạn mức:** Tệp trong thùng rác **vẫn giữ nguyên dung lượng tính vào quota** của người dùng (`used_bytes` không đổi) để tránh việc lạm dụng thùng rác làm kho lưu trữ miễn phí.
* Hệ thống tự động tạo một tác vụ trong bảng `Jobs`:
  * `job_type`: `PURGE_FILE`
  * `due_at`: `DATE_ADD(NOW(), INTERVAL 7 DAY)` (Hẹn giờ xóa vĩnh viễn sau 7 ngày).

#### Bước 3: Khôi phục tệp (Restore File)
1. Truy cập trang `/trash`.
2. Nhấn nút **Khôi phục** (Restore) bên cạnh tệp `baocao.pdf`.
3. Tệp lập tức được đưa trở lại thư mục gốc ban đầu trên Drive với trạng thái `READY`.
4. Tác vụ hẹn giờ xóa vĩnh viễn trong bảng `Jobs` tự động bị hủy hoặc bỏ qua an toàn nhờ cơ chế khóa trước (Lock job before file).

#### Bước 4: Yêu cầu Xóa vĩnh viễn Chủ động (Permanent Purge)
1. Chuyển lại tệp `baocao.pdf` vào thùng rác.
2. Tại trang `/trash`, nhấn nút **Xóa vĩnh viễn** (Delete Permanently) và xác nhận hộp thoại cảnh báo.

*Phân tích xử lý bất đồng bộ (Asynchronous Purge Flow):*
1. **API phản hồi ngay lập tức**: Backend chuyển trạng thái tệp sang `PURGE_PENDING` và trả về mã HTTP `202 Accepted`. Khi đã ở trạng thái này, mọi yêu cầu khôi phục tệp (`restore`) đều bị từ chối dứt khoát.
2. **Tiến trình Durable Worker tiếp nhận**:
   * Worker lấy tác vụ `PURGE_FILE` và chiếm quyền điều khiển.
   * **Giải thuật xóa S3 theo phiên bản (Version-aware S3 Deletion)**: Do bucket S3 có bật tính năng Versioning, lệnh xóa thông thường chỉ tạo một *Delete Marker* mà không giải phóng dung lượng thực tế. Worker thực hiện:
     * Gọi `ListObjectVersions` để liệt kê toàn bộ các phiên bản và delete markers của key `objects/1/file_77b31a29` (kể cả các phiên bản có ID là `null`).
     * Gom nhóm và gửi lệnh `DeleteObjects` xóa chính xác từng cặp `(Key, VersionId)`.
     * Gọi lại API kiểm kê để xác nhận danh sách phiên bản của key đó trên S3 đã thực sự trống rỗng hoàn toàn.
   * **Quyết toán dung lượng & Tombstone**: Sau khi S3 xác nhận đã xóa sạch đối tượng, Worker thực hiện giao dịch SQL:
     * Trừ dung lượng thực tế của tệp khỏi `Users.used_bytes`.
     * Giữ lại một bản ghi dấu vết (Tombstone) với trạng thái `PURGED` và lưu thời điểm `purged_at` nhằm phục vụ việc kiểm toán lịch sử.

---

### 3. Quy trình Đối soát Dữ liệu Định kỳ (Reconciliation Service)

Nhằm bảo đảm tính toàn vẹn tuyệt đối giữa cơ sở dữ liệu và kho lưu trữ S3, hệ thống cung cấp dịch vụ đối soát định kỳ (`reconciliation.service.js`) thực hiện 4 nhiệm vụ cốt lõi:

```
+------------------------------------------------------------------------------------+
|                         TIẾN TRÌNH ĐỐI SOÁT (RECONCILIATION)                       |
+------------------------------------------------------------------------------------+
| 1. Quét phiên treo: Hủy các UploadSessions quá 15 phút chưa xong -> Trả Quota      |
| 2. Phục hồi tác vụ: Tự động khôi phục các Job FINALIZE bị mất hoặc sót do crash    |
| 3. Cân bằng Quota: Tính lại used_bytes từ tổng kích thước các tệp READY + TRASHED  |
| 4. Quét tệp mồ côi (Orphan Scanner): Tìm các đối tượng trên S3 không có trong DB   |
+------------------------------------------------------------------------------------+
```

#### Chi tiết 4 bước đối soát:
1. **Dọn dẹp phiên tải dở dang (Stalled Sessions Expiry)**: Quét các bản ghi trong `UploadSessions` có trạng thái `ACTIVE` đã tạo quá 15 phút mà không có tệp hoặc không gọi hoàn tất. Giải phóng dung lượng đặt trước và chuyển trạng thái sang `EXPIRED`.
2. **Phục hồi tác vụ hoàn tất (Recover Missing Finalize Work)**: Phát hiện các phiên tải lên đã có báo cáo `PASSED` nhưng vì sự cố worker mà chưa có bản ghi `Files`, tự động tạo lại tác vụ `FINALIZE_UPLOAD` theo mã định danh duy nhất (deduplication key).
3. **Hiệu chỉnh hạn mức lưu trữ (Quota Self-healing)**: Tính toán lại tổng dung lượng thực tế từ bảng `Files` (`SELECT SUM(size) WHERE user_id = ? AND status IN ('READY', 'TRASHED')`). Nếu phát hiện lệch so với `Users.used_bytes`, hệ thống tự động ghi đè giá trị chính xác và ghi cảnh báo kiểm toán.
4. **Phát hiện tệp mồ côi (Orphan Object Detection)**: Quét tiền tố `objects/` trên S3:
   * Nếu phát hiện một đối tượng tồn tại trên S3 nhưng không có bản ghi tương ứng trong database, hệ thống ghi nhận vào bảng `ReconciliationFindings`.
   * **Quy tắc an toàn 24 giờ**: Tệp mồ côi chỉ được phép xóa tự động khi thỏa mãn hai điều kiện:
     * Chế độ dọn dẹp được bật rõ ràng: `ORPHAN_CLEANUP_MODE=delete` (mặc định là `report-only`).
     * Tệp đã được quan sát liên tục và tồn tại vượt quá khoảng thời gian ân hạn (Grace Period) tối thiểu 24 giờ.

---

### 4. Cách thức Kiểm tra và Xác minh

#### Kiểm tra 1: Kiểm tra Hạn mức sau khi Xóa Vĩnh viễn
Truy cập trang `/drive` hoặc gọi API `/api/users/quota`:
```bash
curl -i -b "connect.sid=..." http://localhost:3000/api/users/quota
```
*Kết quả:* Dung lượng sử dụng `used_bytes` đã giảm đúng bằng dung lượng tệp vừa bị xóa vĩnh viễn, thanh dung lượng trên giao diện người dùng trở về trạng thái ban đầu.

#### Kiểm tra 2: Kiểm tra Nhật ký Kiểm toán (Audit Log)
Truy cập mục **Lịch sử hoạt động (Activity)** trên thanh điều hướng (`http://localhost:5173/activity`):
* Bảng hiển thị đầy đủ chuỗi lịch sử thao tác của người dùng:
  * `FILE_UPLOAD`: Tải tệp lên thành công.
  * `FILE_TRASH`: Chuyển tệp vào thùng rác.
  * `FILE_RESTORE`: Khôi phục tệp.
  * `FILE_PURGE`: Xóa tệp vĩnh viễn khỏi hệ thống.
