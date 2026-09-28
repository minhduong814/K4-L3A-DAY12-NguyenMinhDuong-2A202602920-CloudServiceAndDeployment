# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Minh Dương  Mã học viên: 2A202602920

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu quên cấu hình `AGENT_API_KEY` trên cloud, app fail ngay khi khởi động và
dashboard báo deploy không thành công. Nhờ vậy mình sửa biến môi trường trước
khi service nhận request; nếu dùng `changeme`, service vẫn chạy và người khác
có thể gọi API bằng khóa mặc định, gây phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log thực tế khi chạy local với mock LLM:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T11:14:21.799550+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}`.
Từ đó có thể lọc theo `user_id` để tìm user tiêu nhiều tiền và đếm các event
theo thời gian để tính tỷ lệ lỗi/hoàn thành. Một câu `print()` không có schema
ổn định nên khó truy vấn tự động như vậy.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Mình chưa đo được hai image thật trong workspace vì Docker daemon không chạy,
nên không tự điền số MB. Bản multi-stage được thiết kế nhỏ hơn vì image runtime
chỉ chứa Python slim, dependency đã cài và source; compiler cùng các file build
chỉ tồn tại trong stage builder và không được đưa sang runtime.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi chỉ sửa `app/main.py`, các layer từ base image, `COPY requirements.txt` và
`pip install` vẫn dùng lại cache; các layer copy source trở đi phải tạo lại.
Nếu `COPY . .` đứng trước `pip install`, mọi thay đổi nhỏ trong source sẽ làm
Docker mất cache ở layer đó và phải cài lại toàn bộ dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu app có lỗ hổng cho phép thực thi lệnh, tiến trình trong container có thể
đọc/sửa mọi thứ mà user của nó được phép truy cập. Khi chạy bằng root, quyền đó
có thể bao gồm các thao tác đặc quyền trong container và làm tăng khả năng ảnh
hưởng tới host qua lỗ hổng runtime hoặc mount. `USER appuser` giới hạn tiến
trình ở user thường, nên lỗ hổng không tự động biến thành quyền root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với hạn mức 10 request/phút và bộ đếm reset ở giây 00, có thể gửi 10 request
ở giây 59 của phút trước và 10 request ở giây 00–01 của phút sau. Như vậy có
thể đạt 20 request trong khoảng 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số lần gọi trong một cửa sổ thời gian, còn cost guard giới
hạn tổng chi phí theo user trong tháng. Một user có thể còn dưới 10 request/phút
nhưng mỗi request rất dài nên vượt ngân sách và bị 402. Ngược lại, user còn
nhiều ngân sách nhưng gửi quá nhanh thì bị rate limiter chặn 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Redis mất kết nối, endpoint gộp kiểm tra Redis trả lỗi hoặc 503 cho cả ba
container. Orchestrator hiểu đó là liveness failure và restart cả ba container
cùng lúc. Trong 30 giây đó, toàn bộ cụm có thể ngừng phục vụ; khi Redis hoạt
động lại thì các container còn phải khởi động lại đồng loạt. Tách `/health` và
`/ready` giúp `/health` vẫn 200, còn load balancer chỉ tạm ngừng gửi traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với state trong dict riêng của từng process, request đi vào container nào sẽ
chỉ thấy lịch sử của container đó. Vì load balancer phân phối request, giá trị
`history_length` sẽ nhảy hoặc quay lại 0/2 tùy instance, thay vì tăng đều theo
từng lượt. Lưu trong Redis khiến mọi instance dùng chung lịch sử.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Khi kiểm tra public URL từ workspace, `curl` báo `Could not resolve host` cho
hostname Railway. Điều này cho thấy tên miền chưa phân giải được từ mạng kiểm
tra hiện tại (hoặc URL cần xác nhận lại trong Railway Dashboard), nên chưa thể
kết luận lỗi nằm ở FastAPI. Cách xử lý là kiểm tra lại domain public trong
Railway, thử `curl` từ mạng khác, rồi chỉ ghi output `/health` và `/ready` sau
khi DNS hoạt động.
