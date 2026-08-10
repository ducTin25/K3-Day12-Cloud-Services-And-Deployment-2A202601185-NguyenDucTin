# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: .......................... Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định `changeme`, app vẫn khởi động khi tôi quên cấu hình secret trên
> môi trường production. Khi đó người khác có thể dùng khóa mặc định để gọi API
> và làm phát sinh chi phí. Với cách fail fast, app báo lỗi ngay lúc deploy nên
> phát hiện và cấu hình `AGENT_API_KEY` trước khi nhận request thật.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Ví dụ log thu được:
>
> ```json
> {
>   "event": "ask_completed",
>   "level": "info",
>   "timestamp": "2026-08-10T03:27:24.658103+00:00",
>   "user_id": "sv01",
>   "tokens_in": 12,
>   "tokens_out": 24,
>   "cost_usd": 0.0001
> }
> ```
>
> Từ dòng log này, tôi có thể lọc các request theo `user_id` để biết user nào
> gọi nhiều hoặc phát sinh nhiều chi phí, và tính tổng lỗi/chi phí trong một
> khoảng thời gian bằng các trường có cấu trúc. `print("đã trả lời xong")` chỉ
> cho biết có một việc hoàn tất, không có dữ liệu để lọc hay thống kê tự động.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

IMAGE ID DISK USAGE CONTENT SIZE EXTRA
day12-agent:prod e2795f2a56ff 183MB 0B  
| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Kết quả đo thực tế bằng `docker image inspect`:
>
> | Bản | Dung lượng |
> |-----|-----------:|
> | 1 stage gốc (`python:3.11`) | 1132.70 MB |
> | Multi-stage | 174.65 MB |
>
> Bản 1-stage lớn hơn khoảng 958.05 MB vì nó giữ lại base image
> `python:3.11` đầy đủ cùng toàn bộ lớp cài đặt. Multi-stage dùng base
> `slim` cho runtime và chỉ copy dependency cần thiết từ `builder`, nên image
> runtime nhỏ hơn đáng kể, đồng thời không chứa các công cụ build dư thừa.
>

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, layer cài đặt dependency vẫn được dùng lại vì
> `requirements.txt` được copy và cài trước layer copy source. Các layer từ
> base image, copy `requirements.txt` và `pip install` được cache; layer copy
> source và các layer sau nó phải chạy lại.
>
> Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi trong source sẽ làm
> layer `COPY` thay đổi và khiến Docker phải chạy lại `pip install`, làm build
> chậm hơn dù dependency không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng có thể cho phép kẻ tấn công thực thi mã hoặc mở shell trong
> container. Nếu process chạy bằng root, mã đó có quyền đọc/sửa nhiều file và
> thực hiện các thao tác đặc quyền trong container; nếu có thêm lỗ hổng Docker
> hoặc cấu hình mount sai, kẻ tấn công có thể dùng quyền đó để tiếp cận tài
> nguyên của máy host. Lệnh `USER appuser` chuyển process sang user thường,
> nên cắt quyền root ngay tại bước container thực thi ứng dụng và giới hạn
> mức độ thiệt hại.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với hạn mức 10 request/phút nhưng reset theo phút đồng hồ, user có thể gửi
> 10 request ngay trước mốc chuyển phút, chẳng hạn ở giây 59, rồi gửi thêm 10
> request ngay sau mốc đó, ở giây 00 hoặc 01. Vì hai nhóm thuộc hai phút khác
> nhau, hệ thống cho qua tổng cộng **20 request trong khoảng 2 giây**. Sliding
> window 60 giây tránh được khe hở này vì vẫn nhìn thấy các request của 60 giây
> gần nhất.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **số lượng request trong một khoảng thời gian**, còn cost
> guard giới hạn **tổng tiền đã tiêu trong tháng**.
>
> Ví dụ rate limit vẫn cho qua nhưng cost guard phải chặn: user mới chỉ gửi
> vài request nên chưa vượt 10 request/phút, nhưng mỗi request tạo ra rất
> nhiều token và tổng chi phí đã vượt ngân sách tháng.
>
> Trường hợp ngược lại: user gửi 11 request liên tiếp trong một phút nhưng
> mỗi request rất rẻ. Tổng chi phí vẫn dưới ngân sách nên cost guard cho qua,
> nhưng rate limiter chặn request thứ 11 bằng lỗi 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` cũng kiểm tra Redis, khi Redis mất kết nối thì cả 3 container
> đều trả về trạng thái unhealthy hoặc HTTP 503. Orchestrator hiểu rằng process
> của cả 3 container đều hỏng nên khởi động lại cả cụm. Trong 30 giây Redis bị
> lỗi, các container bị dừng và khởi động lại; khi Redis hoạt động trở lại có
> thể không còn instance nào đang phục vụ request, làm sự cố Redis ngắn trở
> thành downtime toàn bộ service. Vì vậy `/health` chỉ kiểm tra process, còn
> `/ready` mới kiểm tra Redis và trả 503 để load balancer tạm ngừng gửi request.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lưu lịch sử trong một dict Python, mỗi container sẽ có một bản sao state
> riêng. Khi load balancer chuyển request sang container khác, container đó
> không thấy lịch sử của container trước nên `history_length` có thể quay lại
> 0 hoặc chỉ tăng trong phạm vi các request tình cờ vào cùng một container.
> Kết quả sẽ thay đổi không ổn định, ví dụ có thể thấy 0, 2, 0, 2 thay vì tăng
> đều. Với Redis dùng chung, mọi container đọc cùng một danh sách nên lịch sử
> được giữ nhất quán dù request đi vào instance nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> _Câu trả lời của bạn_
