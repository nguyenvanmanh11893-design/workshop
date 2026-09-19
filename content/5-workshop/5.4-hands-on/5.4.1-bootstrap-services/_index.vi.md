---
title: "Khởi chạy và kiểm tra dịch vụ"
date: 2026-09-18T23:00:00+07:00
weight: 1
chapter: false
pre: "<b>5.4.1 </b>"
---





---

### 1. Mục đích
* Khởi động đồng thời máy chủ API Backend và máy chủ phát triển Frontend React qua một lệnh duy nhất.
* Khởi động tiến trình xử lý hàng đợi nền độc lập (Durable Worker).
* Xác minh tính sẵn sàng của cơ sở dữ liệu và hệ thống thông qua các API giám sát `/health/live` và `/health/ready`.
* Truy cập giao diện người dùng và tài liệu API tương tác Swagger.

---

### 2. Điều kiện Tiên quyết
* Đã hoàn thành các bước tại **[Mục 5.3: Chuẩn bị môi trường](../../5.3-prerequisites/)**.
* MySQL Server đang chạy tại cổng `3306` và đã thực thi xong toàn bộ các file di trú (`npm run db:migrate`).
* Đã cấu hình file `.env` hợp lệ tại thư mục gốc dự án.

---

### 3. Thao tác Thực hiện

#### Bước 1: Khởi động Ứng dụng Web 
Mở một cửa sổ Terminal (Terminal 1) tại thư mục gốc của dự án và thực thi lệnh:

```bash
npm run dev
```

*Lệnh này sử dụng cấu hình npm workspaces để khởi chạy đồng thời:*
* Backend Express API lắng nghe tại cổng `http://localhost:3000`.
* Frontend Vite Dev Server lắng nghe tại cổng `http://localhost:5173` (đã cấu hình proxy `/api` chuyển tiếp trực tiếp sang cổng `3000`).

#### Bước 2: Khởi động Tiến trình Xử lý Nền (Durable Worker)
Mở một cửa sổ Terminal thứ hai (Terminal 2) tại thư mục gốc của dự án và chạy lệnh:

```bash
npm run worker
```

*Tiến trình nền `worker.js` sẽ khởi tạo cơ chế polling theo chu kỳ (mặc định 3000ms), sẵn sàng quét và tranh chấp các tác vụ trong bảng `Jobs`.*

---

### 4. Kết quả Mong đợi 

* **Terminal 1 (App Dev)**:
```text
[backend]  Server running in development mode on port 3000
[backend]  Database connected successfully.
[frontend] VITE v5.x.x  ready in 320 ms
[frontend] ➜  Local:   http://localhost:5173/
[frontend] ➜  Network: use --host to expose
```

* **Terminal 2 (Worker)**:
```text
[worker] Worker started. Polling interval: 3000ms
[worker] Heartbeat registered in database.
[worker] Polling for jobs... (0 active leases)
```

---

### 5. Cách thức Kiểm tra và Xác minh

#### Kiểm tra 1: Kiểm tra Liveness Probe của API
Mở một cửa sổ terminal mới và gửi yêu cầu HTTP GET tới endpoint kiểm tra trạng thái sống:

```bash
curl -i http://localhost:3000/health/live
```
*Kết quả phản hồi:* HTTP `200 OK`
```json
{
  "status": "ok",
  "timestamp": "2026-09-18T23:00:00.000Z"
}
```

#### Kiểm tra 2: Kiểm tra Readiness Probe và Kết nối Cơ sở dữ liệu
Gửi yêu cầu HTTP GET tới endpoint kiểm tra mức độ sẵn sàng:

```bash
curl -i http://localhost:3000/health/ready
```
*Kết quả phản hồi:* HTTP `200 OK`
```json
{
  "status": "ready",
  "database": "connected",
  "uptime_seconds": 45
}
```


#### Kiểm tra 3: Truy cập Giao diện Trình duyệt
Mở trình duyệt web (Google Chrome hoặc Microsoft Edge) và truy cập địa chỉ:
* **Ứng dụng chính**: `http://localhost:5173` — Màn hình đăng nhập Cloud File Manager hiển thị với ngôn ngữ tiếng Việt và giao diện chuẩn Ant Design.
* **Tài liệu API Swagger**: `http://localhost:3000/api-docs` — Trang Swagger UI hiển thị danh mục toàn bộ các REST API của hệ thống kèm theo mô tả schema request/response.
