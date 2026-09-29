# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu deploy lên cloud mà quên đặt `AGENT_API_KEY`, ứng dụng phải dừng ngay khi khởi động. Nhờ vậy lỗi cấu hình xuất hiện trong build/runtime log trước khi service nhận traffic. Nếu có mặc định `changeme`, service vẫn chạy và người ngoài có thể đoán khóa rồi dùng API miễn phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Ví dụ: `{"event":"ask_completed","level":"info","timestamp":"2026-09-29T10:00:00+00:00","user_id":"sv01","cost_usd":0.12}`. Log JSON có thể được hệ thống log lọc theo `event` hoặc `user_id`, đồng thời tính tổng chi phí theo trường số `cost_usd`; một câu `print` tự do không có schema ổn định để làm hai việc đó.

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

Docker daemon trên máy chưa chạy nên chưa có số đo thật để ghi. Khi daemon hoạt động, cần build cả hai image rồi ghi kích thước từ `docker images`. Phần chênh lệch chủ yếu là compiler, cache pip và dependency build ở stage builder; image runtime multi-stage chỉ giữ Python slim, virtualenv và source cần chạy.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại, sửa `app/main.py` chỉ làm mất cache từ layer `COPY app ./app` của stage builder trở đi; layer cài dependency vẫn được dùng lại vì `COPY requirements.txt` và `pip install` đứng trước. Nếu `COPY . .` đặt trước `RUN pip install`, mọi thay đổi source sẽ làm layer source đổi và buộc cài lại dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu code có lỗ hổng cho phép chạy lệnh, tiến trình root có thể đọc hoặc sửa nhiều tài nguyên trong container và có mức đặc quyền cao hơn khi khai thác thêm lỗi cấu hình runtime. `USER appuser` làm tiến trình ứng dụng chạy với UID thường, cắt chuỗi leo thang đặc quyền ngay sau bước thoát khỏi ứng dụng.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với hạn mức 10 request/phút, cách đếm theo phút đồng hồ cho phép 20 request trong 2 giây: gửi 10 request ở giây 59 của phút trước và 10 request ở giây 00 hoặc 01 của phút sau. Sliding window nhìn lại đúng 60 giây gần nhất nên không có khe hở này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số lần gọi trong một khoảng thời gian ngắn, còn cost guard giới hạn tổng tiền theo user trong tháng. Rate limit có thể cho qua request nếu user mới gọi ít lần nhưng cost guard chặn vì ngân sách tháng gần hết. Ngược lại, rate limit có thể chặn một chuỗi gọi nhanh dù mỗi request rất rẻ, trong khi ngân sách tháng vẫn còn nhiều.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Khi Redis mất kết nối, mỗi container vẫn trả `/health` là 200 vì liveness chỉ kiểm tra process. `/ready` của cả ba container lần lượt kiểm tra Redis và trả 503, load balancer ngừng gửi traffic mới. Khi Redis hoạt động lại, `/ready` trả 200 và các container được đưa trở lại pool; nếu gộp hai probe, cả cụm sẽ bị coi là không sống và có thể bị restart dù process vẫn chạy.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với dict trong RAM, request đầu tiên vào container A tăng lịch sử ở A, nhưng request tiếp theo vào B không thấy dữ liệu đó nên `history_length` có thể quay về 0. Khi dùng Redis, mọi instance đọc cùng key nên lịch sử tăng ổn định theo các message đã lưu, bất kể request đi vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Chưa triển khai cloud trong môi trường hiện tại nên chưa có lỗi deploy và chưa thể ghi một thông báo lỗi thật. Docker daemon cũng chưa chạy, vì vậy phần kiểm chứng CP5 cần tiếp tục bằng Railway hoặc Render: tạo Redis, đặt `AGENT_API_KEY` và `REDIS_URL` trong dashboard, đọc runtime log nếu health check lỗi, rồi ghi lại lỗi và cách sửa sau lần deploy thực tế.
