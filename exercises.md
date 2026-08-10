# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay phần trả lời mẫu dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi quên set `AGENT_API_KEY` trên Render, app dừng ngay thay vì mở `/ask` với khóa mặc định `changeme` mà ai cũng đoán được.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Ví dụ: `{"event":"ask_completed","level":"info","timestamp":"...","user_id":"sv-test","tokens_in":4,"tokens_out":30,"cost_usd":0.00002}`. Tôi có thể lọc log theo `user_id` và cộng `cost_usd` để theo dõi/cảnh báo chi phí; `print` tự do không cho máy lọc các trường này đáng tin cậy.

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
| Multi-stage | 309 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản 1-stage mang Python đầy đủ, pip cache, test dependency và toàn bộ source trong cùng image. Bản multi-stage dùng base slim và chỉ chép virtual environment cùng code cần chạy sang runtime.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer copy `requirements.txt`, tạo venv và `pip install` được cache; layer `COPY app` chạy lại. Nếu `COPY . .` đứng trước `pip install`, mọi sửa code sẽ làm invalid cache và cài lại dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Lỗ hổng có thể cho kẻ tấn công chạy lệnh trong container; nếu process là root, họ có quyền root trong container và dễ khai thác thêm cấu hình/kernel sai để ảnh hưởng host. `USER app` hạ quyền process ngay trước lúc chạy service, nên lệnh khai thác không có quyền root mặc định.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request: gửi 10 request ở giây 59 của một phút, rồi 10 request ở giây 01 của phút kế tiếp. Bộ đếm theo phút đã reset dù chỉ cách nhau 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong 60 giây, còn cost guard giới hạn tổng USD theo tháng. Rate limit có thể cho qua một request ít nhưng user đã hết budget nên cost guard chặn; ngược lại request thứ 11 trong phút bị rate limit chặn dù user vẫn còn budget.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm probe chung trả lỗi; orchestrator đánh dấu cả 3 container unhealthy và lần lượt restart chúng. Trong lúc đó traffic không còn instance khỏe dù process vẫn sống; Redis trở lại thì các container mới mới phục vụ lại. Tách `/health` tránh chuỗi restart này.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, `history_length` tăng 0, 2, 4... dù request rơi vào container nào. Với dict Python, khi request sang instance khác lịch sử lại là 0 hoặc thấp hơn, nên số này không tăng ổn định.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Log Render có `GET /` trả 404. Tôi kiểm tra route và thấy app chỉ có `/health`, `/ready`, `/ask`; sau đó dùng `/health` làm health check và xác minh nó trả 200, service chuyển sang Live.
