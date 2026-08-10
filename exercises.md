# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay phần trả lời mẫu dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Quốc Việt  Mã học viên: 2A202601369

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Render, tôi từng để thiếu `AGENT_API_KEY`. App vẫn có thể tạo container, nhưng lần đầu `/ready` cần đọc `Settings` thì log báo `ValidationError: agent_api_key - Field required`. Nhờ fail fast tôi biết ngay cấu hình cloud thiếu biến và thêm nó trong Environment. Nếu để mặc định `changeme`, app sẽ chạy và bất kỳ ai đoán được khóa mặc định đều có thể gọi LLM, đến khi thấy hóa đơn tăng mới phát hiện.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một log của lần gọi thành công có dạng `{"timestamp":"2026-08-10T...+00:00","level":"info","event":"ask_completed","user_id":"sv-test","tokens_in":1,"tokens_out":35,"cost_usd":2.115e-05}`. Tôi có thể lọc tất cả request của `sv-test` để điều tra lỗi/rate limit, và cộng hoặc đặt cảnh báo theo trường `cost_usd` hay `tokens_out`. `print("đã trả lời xong")` không có cấu trúc, không có user, chi phí hay thời gian để máy lọc và tổng hợp.

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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Image one-stage tôi đo được là 1.73 GB, còn multi-stage `agent:multi` là 270 MB, giảm khoảng 1.46 GB. Phần chênh lệch là base `python:3.11` đầy đủ, pip cache và các lớp/build tools không cần khi chạy. Multi-stage chỉ copy package đã cài từ builder sang runtime `python:3.11-slim`, nên bỏ được các phần phục vụ build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi tôi sửa `app/main.py`, các layer `FROM`, `WORKDIR`, `COPY requirements.txt` và `RUN pip install` vẫn dùng cache vì `requirements.txt` không đổi. Các layer `COPY app/`, `COPY utils/` và layer sau đó chạy lại. Nếu `COPY . .` nằm trước `RUN pip install`, chỉ một ký tự thay đổi ở source cũng làm layer COPY đổi hash, Docker phải cài lại toàn bộ dependency; build chậm hơn nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng như command injection có thể cho kẻ tấn công chạy lệnh trong process Python. Nếu process đó là root, họ có thể đọc/sửa toàn bộ filesystem trong container, cài công cụ, lấy credential hoặc khai thác một lỗi/volume mount khác để ảnh hưởng host. `USER appuser` làm lệnh từ process bị chiếm chỉ có quyền user thường, nên bị chặn ở các file và thao tác cần root; nó giảm blast radius dù không thay thế việc vá lỗ hổng.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request trong khoảng 2 giây: gửi 10 request ở 10:00:59, rồi đồng hồ sang 10:01:00 và bộ đếm reset, gửi tiếp 10 request ở 10:01:01. Sliding window nhìn lại đủ 60 giây trước thời điểm request nên vẫn thấy 10 request trước đó và không cho burst thứ hai vượt limit.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit kiểm soát tần suất request trong một cửa sổ thời gian; cost guard kiểm soát tổng tiền theo tháng. Ví dụ user mới chỉ gửi vài request nhưng mỗi request cực dài/tốn nhiều token: chưa chạm 10 request/phút nên rate limit cho qua, nhưng tổng dự kiến vượt 10 USD thì cost guard chặn. Ngược lại, user gửi nhiều câu rất ngắn liên tiếp: tiền tháng còn thấp nên cost guard cho qua, nhưng request thứ 11 trong 60 giây bị rate limit trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu endpoint chung kiểm tra Redis, Redis mất kết nối thì cả 3 container đều trả unhealthy. Load balancer/orchestrator lần lượt coi từng container là chết và restart chúng. Trong 30 giây Redis còn lỗi, các container mới lên lại tiếp tục fail healthcheck và bị restart, làm mất các request đang xử lý dù process ứng dụng vốn vẫn sống. Tách `/health` giúp container vẫn 200 và không bị restart oan; chỉ `/ready` 503 để tạm rút traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis chung, dù request được phân phối qua ba agent khác nhau, `history_length` của cùng `X-User-Id` tăng liên tục theo các lượt (đến giới hạn 20) vì mọi instance đọc cùng key Redis. Nếu dùng dict Python, mỗi instance có một dict riêng nên con số sẽ nhảy, ví dụ 0, 1, rồi lại 0 khi request sang instance khác; agent có cảm giác quên lịch sử.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp trên Render là `/ready` trả 500. Trong Logs có traceback `ValidationError: agent_api_key - Field required` tại `Settings()`. Tôi lần theo stack trace thấy `/ready` tạo Redis store và lúc đó load cấu hình; `AGENT_API_KEY` chưa được set ở Render Environment. Tôi thêm biến này trong dashboard rồi Save and deploy. Sau redeploy, `/ready` trả `{"status":"ready","redis":true}` và `/ask` gọi được với key hợp lệ.
