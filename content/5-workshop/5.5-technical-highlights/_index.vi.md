---
title: "Điểm kỹ thuật nổi bật"
date: 2026-09-18T23:00:00+07:00
weight: 5
chapter: false
pre: "<b>5.5. </b>"
---





---

### 1. Mô hình Bảo mật Phiên Opaque, Phòng chống CSRF và Đăng ký qua Mã mời

#### 1.1 Opaque Server-Side Session vs. Stateless JWT
Thay vì lưu trữ JSON Web Token (JWT) trong `localStorage` (dễ bị đánh cắp khi xảy ra lỗ hổng XSS), Cloud File Manager sử dụng kiến trúc **Opaque Session**:
* **Lưu trữ phía máy chủ**: Mỗi phiên làm việc là một bản ghi trong bảng `Sessions` của MySQL, chứa mã định danh ngẫu nhiên có độ entropy cao (`session_id`), `user_id`, thời điểm hết hạn và dữ liệu phiên.
* **HttpOnly & SameSite Cookie**: Trình duyệt chỉ nhận một cookie phiên duy nhất có gắn cờ `HttpOnly` (chặn JavaScript đọc cookie) và `SameSite=Lax` (giới hạn gửi cookie trong các truy vấn chéo trang).
* **Thu hồi phiên tức thì (Instant Revocation)**: Máy chủ có thể vô hiệu hóa ngay lập tức một phiên đăng nhập bị nghi ngờ bằng cách xóa bản ghi trong bảng `Sessions`, điều mà JWT phi trạng thái không thể làm được nếu không có danh sách đen phức tạp.

#### 1.2 Phòng chống tấn công CSRF hai lớp
Do sử dụng cookie phiên tự động gửi kèm theo yêu cầu, hệ thống thiết lập cơ chế phòng thủ CSRF nghiêm ngặt:
* Khi người dùng đăng nhập thành công, client gọi endpoint `/api/auth/csrf` để nhận một chuỗi token ngẫu nhiên được lưu trữ trong bộ nhớ tạm (in-memory) của React SPA.
* Toàn bộ các phương thức HTTP làm thay đổi dữ liệu (`POST`, `PUT`, `DELETE`, `PATCH`) bắt buộc phải gửi kèm header `X-CSRF-Token`.
* Middleware `csrf.middleware.js` đối chiếu token này với dữ liệu phiên máy chủ và kiểm tra tính hợp lệ của header `Origin` / `Referer`. Mọi yêu cầu thiếu token hoặc sai lệch nguồn gốc đều bị từ chối ngay lập tức với mã lỗi HTTP `403 Forbidden`.

#### 1.3 Cơ chế Đăng ký Giới hạn qua Mã mời (Invitation-only Model)
* Hệ thống loại trừ rủi ro đăng ký tự động hàng loạt (Spam / Bot registrations) bằng cách bắt buộc nhập mã mời hợp lệ (`token` trong bảng `Invitations`).
* Mỗi mã mời được cấu hình số lượt sử dụng tối đa (`max_uses`), số lượt đã dùng (`uses_count`) và thời hạn hiệu lực (`expires_at`), bảo đảm quyền kiểm soát người dùng tham gia hệ thống.

---

### 2. Tải tệp Trực tiếp lên S3 (Direct-to-S3 Upload) và Đặt trước Hạn mức (Quota Reservation)

#### 2.1 Giải quyết Triệt để Vấn đề Hiệu năng Máy chủ
Trong các hệ thống truyền thống, khi người dùng tải tệp 50 MiB, máy chủ API phải tiếp nhận 50 MiB vào bộ nhớ đệm (RAM) rồi lại gửi 50 MiB đó lên S3:
* **Hậu quả**: Tiêu tốn gấp đôi băng thông mạng, nghẽn luồng xử lý CPU/Event Loop, dễ gây sập máy chủ khi có nhiều kết nối đồng thời.
* **Giải pháp của Cloud File Manager**: Backend chỉ chịu trách nhiệm ký số sinh ra **Presigned POST Policy**. Trình duyệt của người dùng gửi dữ liệu nhị phân trực tiếp đến cụm máy chủ Amazon S3. Máy chủ API hoàn toàn không phải chạm vào dữ liệu nhị phân của tệp.

#### 2.2 Cơ chế Đặt trước Hạn mức Chống Đua (Fail-closed Quota Reservation)
Một thách thức lớn của mô hình tải tệp trực tiếp là: *Làm thế nào để người dùng không tải vượt quá dung lượng khi tệp chưa được ghi nhận vào database?*

Cloud File Manager áp dụng giải thuật **Quota Reservation**:
1. Khi client yêu cầu tạo phiên tải lên (`POST /api/files/upload-session`), hệ thống tính toán:
   $$\text{used\_bytes} + \text{reserved\_bytes} + \text{requested\_file\_size} \le \text{max\_quota\_bytes}$$
2. Nếu điều kiện thỏa mãn, hệ thống tạo bản ghi `UploadSessions` ở trạng thái `ACTIVE`, tạm tính dung lượng này vào `reserved_bytes`.
3. Nếu người dùng mở nhiều tab tải tệp cùng lúc, mọi yêu cầu vượt quá dung lượng đều bị từ chối ngay từ bước khởi tạo phiên.
4. Nếu phiên tải lên bị hủy hoặc bỏ dở quá 15 phút, tiến trình đối soát nền sẽ tự động giải phóng lượng dung lượng đã đặt trước.

---

### 3. Mô hình Xử lý Nền Bền vững (Durable Worker Pattern)

Tiến trình nền `worker.js` được xây dựng để đảm bảo tính sẵn sàng cao, chịu lỗi và không làm mất việc:

#### 3.1 Khóa Phân tán dựa trên Database (Lease Fencing)
Hệ thống không phụ thuộc vào bộ nhớ RAM của một tiến trình duy nhất. Nhiều worker có thể cùng chạy song song mà không sợ xử lý trùng lặp nhờ cơ chế **Lease Token**:
* Mỗi worker chiếm quyền xử lý một công việc bằng cách sinh một `lease_token` (UUID) và cập nhật thời điểm hết hạn `lease_expires_at = NOW() + 60s`.
* Khi worker thực hiện xong công việc và chuẩn bị ghi dữ liệu vào MySQL, câu lệnh SQL bắt buộc phải kiểm tra:
  `WHERE id = :jobId AND lease_token = :myToken AND lease_expires_at >= NOW()`
* Nếu worker xử lý quá chậm khiến lease bị hết hạn và một worker khác đã tiếp quản, worker ban đầu sẽ bị chặn lại (*fenced out*), ngăn chặn việc ghi đè dữ liệu cũ.

#### 3.2 Thử lại Lũy tiến kèm Độ trễ Ngẫu nhiên (Jittered Exponential Backoff)
Khi các thao tác tương tác với Amazon S3 hoặc dịch vụ ngoài gặp sự cố mạng tạm thời:
* Worker tự động tăng biến `attempts` và tính thời điểm thử lại tiếp theo dựa theo công thức:
  $$\text{delay} = \min(\text{MAX\_DELAY}, \text{BASE\_DELAY} \times 2^{\text{attempts}}) \pm \text{jitter}$$
* Việc thêm độ trễ ngẫu nhiên (*jitter*) giúp tránh hiện tượng "bão yêu cầu" (Thundering Herd Problem) khi hàng loạt tác vụ đồng loạt thử lại cùng một thời điểm.
* Nếu vượt quá số lần thử tối đa (ví dụ 5 lần), tác vụ được chuyển sang trạng thái `FAILED` và ghi nhật ký cảnh báo chi tiết.

---

### 4. Quản lý Vòng đời Tệp và Xóa S3 An toàn theo Phiên bản (Version-aware Deletion)

#### 4.1 Thùng rác (Trash) và Trì hoãn Xóa (7-day Delayed Purge)
* Tệp khi bị người dùng đưa vào thùng rác sẽ không bị xóa ngay lập tức nhằm bảo vệ người dùng trước các thao tác nhầm lẫn.
* Hệ thống sinh một tác vụ `PURGE_FILE` có thời điểm kích hoạt sau đúng 7 ngày (`due_at = NOW() + 7 days`).
* Người dùng có thể khôi phục tệp bất cứ lúc nào trong vòng 7 ngày này. Thao tác khôi phục sẽ khóa bản ghi job trước khi mở khóa tệp để đảm bảo tính nhất quán (Lock job before file).

#### 4.2 Xóa S3 Chính xác theo từng Cặp (Key, VersionId)
Trên các bucket Amazon S3 đã bật tính năng Versioning, lệnh gọi API xóa thông thường (`DeleteObject`) sẽ **không xóa dữ liệu nhị phân** mà chỉ gắn thêm một *Delete Marker*. Dung lượng lưu trữ vẫn bị Amazon tính tiền.

Cloud File Manager xử lý triệt để vấn đề này:
1. Worker gọi `ListObjectVersions` để lấy toàn bộ danh sách phiên bản quá khứ, phiên bản hiện tại và các delete markers của tệp.
2. Tạo danh sách các cặp định danh cụ thể `[ { Key: 'objects/...', VersionId: 'v1' }, { Key: 'objects/...', VersionId: 'v2' } ]`.
3. Gửi lệnh `DeleteObjects` để tiêu hủy vĩnh viễn các phiên bản vật lý trên S3.
4. Gọi lại API kiểm kê; chỉ khi xác nhận S3 không còn bất kỳ phiên bản nào, hệ thống mới trừ `used_bytes` trong MySQL và chuyển bản ghi thành `PURGED`.

---

### 5. Dịch vụ Đối soát Tự động (Self-Healing Storage Reconciliation)

Để đối phó với hiện tượng lệch trạng thái (State Drift) vốn khó tránh khỏi trong các hệ thống phân tán, dịch vụ đối soát định kỳ (`reconciliation.service.js`) hoạt động như một cơ chế tự sửa lỗi:
* **Thu hồi hạn mức treo**: Giải phóng dung lượng của các phiên tải lên bị bỏ dở hoặc quá thời hạn 15 phút.
* **Cân bằng số đo hạn mức**: Định kỳ tính toán lại tổng dung lượng thực tế từ bảng `Files` và cập nhật lại vào bảng `Users`, loại bỏ các sai số lũy kế.
* **Phát hiện tệp mồ côi (Orphan Scanner)**: Quét các tệp tồn tại trên S3 nhưng không có siêu dữ liệu trong database. Hệ thống áp dụng quy tắc an toàn: mặc định chạy ở chế độ báo cáo (`report-only`), ghi vết vào bảng `ReconciliationFindings`. Tệp mồ côi chỉ được phép xóa tự động khi người quản trị bật chế độ `delete` và tệp đã tồn tại vượt quá khoảng thời gian ân hạn (Grace Period) tối thiểu 24 giờ.

---

### 6. Hạ tầng dưới dạng Mã (IaC) và Quy trình Phân phối Tự động (CI/CD)



#### 6.1 Kiến trúc Hạ tầng Terraform (v1.10.5)
Mã nguồn hạ tầng tại `infra/terraform/` định nghĩa:
* **Mạng VPC Singapore (`ap-southeast-1`)**: Phân tách rõ ràng giữa 1 Public Subnet (chứa EC2) và 2 Private DB Subnets (chứa RDS MySQL, hoàn toàn không có kết nối Internet).
* **S3 Gateway Endpoint**: Định tuyến lưu lượng S3 qua hạ tầng mạng riêng của AWS, tối ưu bảo mật và không mất phí băng thông.
* **An toàn cấu hình (HCL Security Guards)**: Bộ kiểm tra tự động `infra/tests/check_infra.py` kiểm chứng 13 kịch bản cấm đột biến nguy hiểm: cấm mở public DB, cấm mật khẩu bản rõ, cấm IMDSv1, cấm mở cổng SSH, cấm xóa quy tắc bảo vệ dữ liệu.

#### 6.2 Pipeline CI/CD với Cổng Kiểm soát Thủ công (Manual-gated Delivery)
Quy trình tại `.github/workflows/`:
* **CI Workflow (`ci.yml`)**: Tự động chạy khi có Pull Request: kiểm tra ESLint, chạy 82 bài test backend, 6 bài test frontend, kiểm tra bảo mật dependency (`npm audit`), validate mã nguồn Terraform và đóng gói bản phát hành đính kèm mã băm SHA-256.
* **CD Workflow (`cd.yml`)**: Xác thực qua AWS GitHub OIDC (không dùng access key tĩnh), yêu cầu người quản trị phê duyệt thủ công (Manual Approval Gate), tải artifact đã được xác thực mã băm và kích hoạt script cập nhật an toàn qua AWS Systems Manager.
