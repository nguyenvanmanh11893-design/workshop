---
title: "4.1. FCAJ Community Day"
weight: 1
chapter: false
---



## Thông tin chung về sự kiện

- **Tên sự kiện:** FCAJ Community Day – AWS First Cloud AI Journey
- **Ngày:** Thứ Bảy, 25/07/2026
- **Thời gian:** 09:00 – 12:00 (GMT+7)
- **Địa điểm:** Công ty TNHH Amazon Web Services Việt Nam tại thành phố Hà Nội
- **Vai trò:** Người tham dự

<!-- TODO_SCREENSHOT: Ảnh tập thể sự kiện FCAJ Community Day; bổ sung ảnh gốc khi cần. -->

## Mục tiêu của sự kiện

FCAJ Community Day được tổ chức nhằm giới thiệu những công nghệ mới nhất về điện toán đám mây (Cloud Computing) và Trí tuệ nhân tạo tạo sinh (Generative AI) thông qua các buổi chia sẻ thực tế từ các chuyên gia AWS và các kỹ sư giàu kinh nghiệm.

Sự kiện cũng tạo cơ hội cho sinh viên và lập trình viên tìm hiểu các giải pháp Cloud trong thực tế, khám phá các công cụ phát triển ứng dụng sử dụng AI và kết nối với cộng đồng AWS.

## Diễn giả

1. Tuấn Vũ
2. Thu Nguyễn
3. Nam Lã
4. Hồ Việt Anh
5. Đức Bùi

## Nội dung nổi bật

### 1. OpenClaw - The Rise and Practice of Open-Source AI Agents

**Kiến trúc của OpenClaw:**

- **L1:** Lớp trừu tượng LLM Pi-AI.
- **L2:** Động cơ thực thi Pi-Agent-core với Agent Loop.
- **L3:** Pi-Coding-Agent với quản lý phiên, xác thực và kỹ năng.
- **L4:** Gateway với bộ nhớ, định danh, cron và kênh giao tiếp.

**Sự bùng nổ và hệ sinh thái Skills:** OpenClaw ghi nhận sự tăng trưởng thần tốc trên GitHub, vượt qua Linux và React, cùng hệ sinh thái các kỹ năng phục vụ lập trình, quản trị và tích hợp.

**Ứng dụng thực tế:**

- Tự động hóa gửi email.
- Đồng bộ hóa với Notion.
- Tạo slide tự động bằng nhiều công cụ.

**Rủi ro hệ thống và giới hạn:**

- Lỗ hổng bảo mật trong Skills: Skill Injection.
- Rủi ro phân quyền lỏng lẻo: Privilege Escalation.
- Hạn chế của bộ nhớ hiện tại: Context Saturation.

**Ý nghĩa đối với lập trình viên:** Công việc đang dịch chuyển từ viết code thủ công sang lắp ráp kỹ năng, đồng thời đòi hỏi định hình lại tư duy bảo mật trong kỷ nguyên AI Agent.

### 2. AI Trends Through the Lens of an Infrastructure Engineer

- Giới thiệu xu hướng AI dưới góc nhìn của kỹ sư hạ tầng AWS.
- Công việc vận hành chuyển từ thủ công sang tự động hóa.
- AI được ứng dụng vào Infrastructure as Code (IaC), giám sát, chi phí và bảo mật.
- Hạ tầng đang tiến tới hướng tự hành với sự hỗ trợ của các dịch vụ AWS.
- Quá trình này đặt ra những thách thức và yêu cầu về các kỹ năng cần chuẩn bị cho tương lai.

### 3. AI trong phát triển phần mềm

*Báo cáo nguồn ghi tên phiên là "Ship Fast with AI, Not by"; tiêu đề được giữ nguyên như trong nguồn và không được suy đoán phần còn thiếu.*

**Nút thắt cổ chai:** AI giúp viết code nhanh hơn khoảng 10 lần, nhưng các công đoạn kiểm thử, đánh giá, triển khai và vận hành thuộc vòng lặp ngoài vẫn giữ nguyên nhịp độ 1 lần, từ đó tạo ra điểm nghẽn mới.

**Vòng lặp trong và vòng lặp ngoài:**

- Vòng lặp trong gồm viết, chạy và gỡ lỗi; các hoạt động này được AI hỗ trợ tăng tốc miễn phí.
- Vòng lặp ngoài gồm đánh giá, tích hợp và triển khai; đây là phần việc cốt lõi do con người xây dựng.

**Vai trò của con người:** Kỹ sư chuyển từ viết từng dòng code sang:

- Đọc thiết kế.
- Quản lý các điểm tiếp giáp (seams).
- Tập trung đánh giá rủi ro ở các phần quan trọng.

**Giải pháp cốt lõi:** Kiến thức được mã hóa thành hạ tầng thông qua các quy tắc lint và tài liệu wiki dùng chung trong kho lưu trữ. Mục tiêu là tự động hóa và tích lũy tri thức cho cả con người lẫn AI.

**Kết luận:** Đầu tư vào vòng lặp ngoài giúp công việc tích lũy theo thời gian, giúp kỹ sư trở nên có năng lực vượt trội thay vì quá phụ thuộc vào AI.

### 4. AI Trends to Business

- Trình bày xu hướng AI trong năm 2026 và cách AI tạo ra giá trị kinh doanh cho đội ngũ Sales.
- Nhấn mạnh sự chuyển dịch của AI từ một xu hướng đơn thuần thành "đồng nghiệp số" xuất hiện ở mọi nơi.
- Các công cụ phổ biến được nhắc đến gồm ChatGPT, Claude, AWS Quick, NotebookLM và Gemini.

## Kiến thức tiếp thu

### 1. AI Agent và OpenClaw

- Hiểu khái niệm và kiến trúc cơ bản của AI Agent.
- Biết cách AI Agent sử dụng Skills và các công cụ để tự động hóa công việc.
- Nhận thức về các rủi ro như Skill Injection, Privilege Escalation và quản lý quyền truy cập.

### 2. AI trong hạ tầng AWS

- Hiểu cách AI hỗ trợ IaC, giám sát, bảo mật và tối ưu chi phí.
- Nắm được xu hướng phát triển từ vận hành thủ công sang hạ tầng tự động và tự hành.

### 3. AI trong phát triển phần mềm

- AI có thể tăng tốc viết code, kiểm thử và xử lý lỗi.
- Người phát triển cần tập trung hơn vào thiết kế, đánh giá, kiểm soát rủi ro và vận hành hệ thống.
- Cần chuẩn hóa tài liệu, quy trình và kiến thức để AI có thể hỗ trợ hiệu quả.

### 4. AI trong kinh doanh

- AI đang trở thành "đồng nghiệp số", hỗ trợ nhiều hoạt động trong doanh nghiệp.
- Biết thêm các công cụ như ChatGPT, Claude, AWS Quick, NotebookLM và Gemini.
- Việc ứng dụng AI cần gắn với bài toán và mục tiêu kinh doanh cụ thể.

## Bài học rút ra

1. AI đang chuyển từ công cụ hỗ trợ sang "đồng nghiệp số", có khả năng thực hiện nhiều nhiệm vụ và phối hợp với các công cụ khác.
2. Không chỉ cần biết sử dụng AI mà còn cần hiểu cách xây dựng và quản lý hệ thống AI Agent, đặc biệt là về kỹ năng, bộ nhớ, phân quyền và bảo mật.
3. Tốc độ phát triển sản phẩm không chỉ phụ thuộc vào tốc độ viết code. Kiểm thử, đánh giá, tích hợp và vận hành vẫn là những yếu tố quyết định chất lượng sản phẩm.
4. Cần xây dựng quy trình để AI có thể làm việc hiệu quả, thay vì chỉ sử dụng AI riêng lẻ cho từng nhiệm vụ.
5. Bảo mật phải được đặt ngay từ đầu khi ứng dụng AI, đặc biệt với quyền truy cập, Skill Injection và Privilege Escalation.
6. Kỹ năng của người làm công nghệ cần chuyển từ việc chỉ tập trung viết code sang tư duy hệ thống, thiết kế quy trình, đánh giá kết quả và quản trị rủi ro.
7. AI cần gắn với mục tiêu kinh doanh cụ thể. Việc lựa chọn và triển khai AI nên hướng đến các giá trị thực tế như tiết kiệm thời gian, tăng năng suất, cải thiện chất lượng công việc và hỗ trợ ra quyết định.
8. Quan trọng nhất, AI không thay thế tư duy của con người. Người có khả năng đặt vấn đề đúng, kiểm soát AI và biến kết quả của AI thành giá trị thực tế sẽ có lợi thế lớn trong môi trường làm việc mới.

## Cảm nhận sau sự kiện

Sự kiện giúp tôi có cái nhìn rõ hơn về xu hướng phát triển và ứng dụng AI trong công nghệ và doanh nghiệp. Đặc biệt, tôi nhận thấy AI không chỉ hỗ trợ viết code hay xử lý công việc mà đang dần trở thành một "đồng nghiệp số", có khả năng tự động hóa nhiều quy trình.

Qua đó, tôi cũng hiểu rằng việc sử dụng AI hiệu quả cần đi kèm với kiến thức chuyên môn, tư duy hệ thống và khả năng kiểm soát rủi ro.

## Vai trò và đóng góp cá nhân

- **Vai trò:** Người tham dự.
- Tham dự và theo dõi các nội dung chia sẻ tại sự kiện.
- Tổng hợp các kiến thức và nội dung chính trong báo cáo tổng kết.
