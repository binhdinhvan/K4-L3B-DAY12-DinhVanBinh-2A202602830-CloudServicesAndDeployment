# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Việc app "chết sớm" giúp phát hiện ngay lỗi cấu hình từ lúc khởi động, ngăn chặn việc deploy một ứng dụng không bảo mật. Nếu để mặc định là "changeme", khi deploy lên production mà quên set biến môi trường, app vẫn chạy bình thường. Khi đó bất kỳ ai biết hoặc đoán được key đều có thể gọi API, làm lộ dữ liệu hoặc tiêu tốn tiền API của hệ thống (vì không có xác thực an toàn thực sự).

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> `{"event": "Request processed", "endpoint": "/ask", "user_id": "test_user", "processing_time_ms": 150, "timestamp": "2024-03-20T10:00:00Z", "level": "info"}`
> 
> Hai việc làm được với log JSON:
> 1. Dễ dàng parse và query tự động bằng các hệ thống quản lý log (như Elasticsearch, Datadog) để lọc log theo một trường cụ thể (ví dụ: `user_id="test_user"`), giúp debug chính xác một user nào đó.
> 2. Dễ dàng tạo dashboard thống kê hoặc thiết lập alert (ví dụ tính thời gian xử lý trung bình `processing_time_ms` hoặc cảnh báo khi có quá nhiều log `level="error"`).

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1.0 GB |
| Multi-stage | ~150 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch là các công cụ build (như compiler, gcc, make), source code không cần thiết và các bộ đệm (cache) sinh ra trong quá trình cài đặt dependencies (ví dụ apt cache, pip cache). Multi-stage build loại bỏ các công cụ này ở stage cuối cùng và chỉ copy lại môi trường đã cài sẵn và thư mục code để chạy, giúp giảm thiểu tối đa kích thước.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Nếu sửa 1 ký tự trong `app/main.py`, các layer trước lệnh `COPY app/ ./app/` (như cài đặt OS package, copy requirements và pip install) sẽ được dùng lại từ cache. Chỉ các layer từ `COPY app/ ./app/` trở đi bị mất cache và phải chạy lại. 
> Nếu đặt `COPY . .` lên trước `RUN pip install`, vì file code (app) thay đổi liên tục trong quá trình code, layer `COPY . .` sẽ thường xuyên bị mất cache mỗi khi bạn sửa code. Kéo theo đó, layer `RUN pip install` đứng sau nó cũng phải chạy lại toàn bộ, làm cho quá trình build rất chậm chạp thay vì dùng được cache của pip install.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: Ứng dụng chạy bằng quyền root -> Kẻ tấn công lợi dụng lỗ hổng bảo mật trong code (ví dụ remote code execution) -> Kẻ tấn công có thể thực thi lệnh bên trong container với quyền root -> Vì root trong container có đặc quyền cao, kẻ tấn công có thể leo thang đặc quyền để thoát ra ngoài (container breakout), can thiệp vào máy host.
> Lệnh `USER` cắt đứt chuỗi tấn công ở bước thực thi. Bằng cách chuyển quyền thực thi process sang một user không có đặc quyền, nếu bị lợi dụng lỗ hổng, kẻ tấn công cũng chỉ có quyền thấp của user đó, không thể thay đổi hệ thống, cài đặt phần mềm độc hại hay tương tác sâu với máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp.
> Giải thích: Giả sử hạn mức là 10 request/phút. Người dùng gửi 10 request ở thời điểm 00:59, lúc này vẫn hợp lệ. Đến 01:00 (chỉ cách 1 giây), bộ đếm tự động reset về 0 theo đồng hồ phút. Người dùng có thể ngay lập tức gửi thêm 10 request nữa. Kết quả là có 20 request lọt qua chỉ trong vòng 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác nhau: Rate limit bảo vệ hệ thống khỏi tình trạng quá tải bằng cách giới hạn số lượng request trong một khoảng thời gian ngắn (ví dụ tính bằng giây, phút). Cost guard bảo vệ ngân sách bằng cách giới hạn tổng giá trị tài nguyên/chi phí tích lũy trong một khoảng thời gian dài (ví dụ tính bằng tháng).
> - Tình huống rate limit cho qua nhưng cost guard chặn: Người dùng thỉnh thoảng mới gọi API (không bị chặn rate limit), nhưng mỗi request xử lý một file rất lớn tốn nhiều token LLM, dần dần đến cuối tháng tổng chi phí vượt quá ngân sách nên cost guard sẽ chặn.
> - Tình huống ngược lại: Người dùng sử dụng script gọi API liên tục gửi hàng chục request nhỏ gọn trong một giây. Dù tổng chi phí của các request này còn rất nhỏ và chưa bị cost guard chặn, nhưng rate limit sẽ chặn ngay lập tức để tránh làm nghẽn server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Trình tự: Redis mất kết nối -> Endpoint chung `/health` kiểm tra Redis và fail -> Hệ thống orchestration (ví dụ Kubernetes) gọi `/health` thấy fail liên tục sẽ đánh giá container bị "chết" (dead) -> Orchestrator tự động kill container đó và khởi động lại container mới -> Container mới khởi động lên, nhưng Redis vẫn chưa khôi phục nên check `/health` lại fail -> Container lại bị kill và lặp lại vòng lặp crash loop liên tục, vô nghĩa. (Nếu tách `/ready` riêng, app chỉ báo `/ready` fail để ngừng nhận request, container vẫn sống chờ Redis khôi phục chứ không bị kill oan).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu dùng dict Python lưu tại bộ nhớ RAM, biến `history_length` sẽ tăng không đồng đều, bị đứt quãng hoặc hiển thị sai. Vì 3 container có 3 dict độc lập, các request của bạn sẽ được bộ cân bằng tải (load balancer) phân phối ngẫu nhiên tới 1 trong 3 container. Lịch sử bạn trò chuyện với container 1 sẽ không có ở container 2, nên độ dài trả về ở từng request sẽ lộn xộn. Khi dùng Redis, do đây là nơi lưu trữ tập trung, cả 3 container đều đọc ghi chung nên `history_length` luôn tăng dần một cách đồng bộ.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp: Ứng dụng không thể kết nối tới cơ sở dữ liệu trên cloud khi chạy thực tế, báo lỗi `ConnectionRefusedError: [Errno 111] Connect call failed` liên quan đến port 6379 trong mục log của Render.
> Tìm ra nguyên nhân: Đọc cấu hình code thấy hệ thống vẫn đang trỏ mặc định vào `localhost:6379` để tìm Redis như lúc chạy test ở máy cá nhân, nhưng trên môi trường Render, Redis nằm ở một địa chỉ mạng khác (Internal URL hoặc External URL của dịch vụ Redis).
> Cách sửa: Vào mục Environment Variables của Web Service trên Render, thêm biến môi trường `REDIS_URL` chứa chuỗi kết nối Internal URL của Redis cung cấp. Code sẽ lấy `os.getenv("REDIS_URL")` và kết nối thành công.
