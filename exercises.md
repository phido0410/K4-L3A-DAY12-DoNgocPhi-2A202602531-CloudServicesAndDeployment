# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder dưới mỗi câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Ngọc Phi  Mã học viên: 2A202602531

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: tôi deploy lên Railway nhưng quên set `AGENT_API_KEY` trong
dashboard.

- Nếu có mặc định `"changeme"`: app vẫn khởi động bình thường, `/health` trả
  200, Railway báo deploy thành công. Nhưng lúc này `/ask` được "bảo vệ" bằng
  một khóa nằm công khai trong source code trên GitHub, nên ai đọc repo cũng
  gọi được. Tôi không nhận ra gì cho tới khi thấy chi phí LLM tăng.
- Không có mặc định: tôi đã thử chạy `Settings()` khi thiếu biến, nó báo ngay
  `ValidationError: agent_api_key Field required`. Trên Railway container sẽ
  crash lúc khởi động, health check fail, deploy bị đánh dấu thất bại, và log
  ghi rõ thiếu biến nào. Lỗi hiện ra đúng lúc tôi đang nhìn màn hình deploy,
  sửa bằng cách set biến rồi deploy lại, không có khoảng thời gian nào service
  chạy với khóa yếu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thu được từ `docker compose logs agent`:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:40:38.301654+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
```

Hai việc làm được mà `print("đã trả lời xong")` không làm được:

1. **Tổng hợp chi phí theo user**: lọc `event == "ask_completed"`, nhóm theo
   `user_id`, cộng `cost_usd` để biết user nào tiêu nhiều tiền nhất hôm nay.
   Dòng `print` không có user, không có số tiền nên không tính được gì.
2. **Đặt cảnh báo tự động**: vì mỗi dòng có `level` và `timestamp` chuẩn
   ISO-8601, công cụ log của cloud có thể đếm số dòng `level == "error"` trong
   5 phút gần nhất và báo động khi vượt ngưỡng. Với text tự do thì phải viết
   regex đoán nội dung, dễ sai và vỡ khi ai đó đổi câu chữ.

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
| 1 stage (bản đầu) | 1820 MB (1.82GB) |
| Multi-stage | 297 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Chênh lệch khoảng 1.5GB. Tôi xem `docker history` của hai image và thấy phần
chênh đến từ ba nguồn:

1. **Base image** (phần lớn nhất): `python:3.11` bản đầy đủ nặng khoảng 1.6GB
   vì có sẵn gcc, make, header file, git và nhiều thư viện hệ thống để biên
   dịch. App chạy thì không cần những thứ đó. `python:3.11-slim` chỉ có
   Python và vài thư viện tối thiểu.
2. **`COPY . .` copy cả thứ không cần** (lớp 66.6MB): bản single build khi
   `.dockerignore` chỉ có `.git`, nên `.venv` (63MB, lại là thư viện build cho
   macOS, vô dụng trong container Linux), `tests/`, tài liệu `.md` và cả
   `.env` chứa khóa thật đều nằm trong image. Chạy `ls -a /app` trong
   container single thấy rõ các file này.
3. **Cache của pip**: lớp `pip install` của bản single là 95.3MB, còn thư viện
   copy sang bản multi chỉ 65.9MB. Phần chênh phần lớn là cache pip giữ lại do
   không dùng `--no-cache-dir`.

Bản multi-stage chỉ gồm: slim base + thư mục `/install` từ stage builder +
`app/` và `utils/` (khoảng 130KB).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Tôi thêm một dòng comment vào `app/main.py` rồi build lại bản multi-stage:

- **Dùng lại từ cache (`CACHED`)**: toàn bộ stage builder (`WORKDIR /build`,
  `COPY requirements.txt`, `RUN pip install`), và ở stage runtime là
  `WORKDIR /app`, `COPY --from=builder /install`, `RUN useradd`.
- **Chạy lại**: chỉ `COPY app ./app` (file vừa đổi) và `COPY utils ./utils`
  (vì Docker huỷ cache từ layer đầu tiên thay đổi trở xuống). Bước cuối xong
  trong 0.1 giây.

Với `Dockerfile.single` (`COPY . .` đứng trước `pip install`), cũng sửa một
dòng comment như vậy: `COPY . .` thay đổi nên mọi layer phía sau mất cache,
`RUN pip install` phải cài lại toàn bộ thư viện, mất **139 giây**, tổng thời
gian build 142.8 giây. Tức là mỗi lần sửa một dấu phẩy trong code là chờ hơn
2 phút, trong khi nếu đặt `requirements.txt` riêng lên trước thì chỉ mất vài
giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện khi container chạy root:

1. Code hoặc một thư viện có lỗ hổng cho phép chạy lệnh tuỳ ý (ví dụ lỗi
   deserialize dữ liệu, command injection, hoặc CVE của một dependency).
2. Kẻ tấn công chạy được lệnh bên trong container **với quyền của process
   app**, tức là root (uid 0).
3. Với root trong container: đọc/sửa mọi file, cài thêm công cụ, đọc biến môi
   trường chứa secret, sửa code app để cài backdoor.
4. Container dùng chung kernel với host, và uid 0 trong container chính là uid
   0 trên host (khi không bật user namespace). Nếu container có cấu hình lỏng
   (mount thư mục host, mount `docker.sock`, chạy `--privileged`) hoặc kernel
   có lỗ hổng escape, kẻ tấn công thoát ra ngoài và **là root trên host**.

`USER appuser` cắt chuỗi ngay ở bước 2: process app chạy với uid 10001, nên dù
khai thác được lỗ hổng, kẻ tấn công cũng chỉ có quyền của một user thường:
không cài được gói hệ thống, không ghi được ra ngoài những file appuser sở hữu,
và nếu có thoát khỏi container thì cũng chỉ là một user không có quyền trên
host. Tôi đã kiểm tra: `whoami` trong `agent:single` ra `root`, trong container
agent của compose ra `appuser`.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa **20 request trong 2 giây**, gấp đôi hạn mức.

Cách làm: gửi 10 request lúc 10:00:59, lúc này bộ đếm của phút 10:00 là 10,
vẫn hợp lệ. Sang 10:01:00 bộ đếm reset về 0, gửi tiếp 10 request lúc
10:01:00–10:01:01, bộ đếm phút 10:01 cũng chỉ là 10. Tổng 20 request trong
khoảng 2 giây mà không lần nào bị chặn.

Với sliding window, lúc 10:01:01 tôi đếm các request trong 60 giây gần nhất
(từ 10:00:01), vẫn thấy 10 request lúc 10:00:59, nên request thứ 11 bị 429 ngay.
Không có "ranh giới phút" để lợi dụng.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn **số lượng request trong một khoảng thời gian ngắn**
(10 request / 60 giây), bảo vệ service khỏi bị gọi dồn dập. Cost guard giới hạn
**tổng số tiền trong cả tháng** (10 USD / user), bảo vệ ngân sách. Một cái đếm
số lần, một cái đếm tiền; mã lỗi cũng khác (429 và 402).

- **Rate limit cho qua, cost guard chặn**: một user gọi đều đặn 5 request/phút
  (dưới hạn mức) nhưng mỗi request gửi câu hỏi dài gần 2000 ký tự, kèm lịch sử
  20 message, nên mỗi lần tốn nhiều token. Gọi liên tục nhiều ngày, tổng chi
  phí chạm 10 USD và `/ask` trả 402 dù chưa lần nào vượt tốc độ.
- **Rate limit chặn, cost guard cho qua**: một script gửi 15 câu hỏi rất ngắn
  trong vài giây. Tổng chi phí chỉ khoảng 0.0003 USD, còn rất xa ngân sách,
  nhưng từ request thứ 11 bị 429. Khi chạy thử tôi thấy đúng như vậy: sau 10
  request thành công thì các lần tiếp theo đều trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Giây 0: Redis mất kết nối. Cả 3 container vẫn chạy bình thường, nhưng
   endpoint gộp của cả 3 bắt đầu trả 503 cùng lúc.
2. Orchestrator gọi liveness probe, thấy fail liên tiếp đủ số lần `retries`, và
   kết luận cả 3 container đều "chết".
3. Orchestrator **restart cả 3 container gần như đồng thời**. Mọi request đang
   xử lý dở bị cắt, và trong lúc khởi động lại không còn container nào nhận
   request.
4. Container khởi động xong, gọi probe, Redis vẫn chưa về nên lại fail, lại bị
   restart. Nhiều platform tăng dần thời gian chờ giữa các lần restart
   (backoff).
5. Giây 30: Redis quay lại, nhưng các container đang nằm trong vòng restart
   hoặc chờ backoff, nên service còn sập thêm một lúc nữa. Một sự cố Redis 30
   giây thành sự cố toàn hệ thống lâu hơn thế.

Khi tách riêng như bài lab: `/health` không đụng Redis nên vẫn 200, không
container nào bị restart; `/ready` trả 503 nên load balancer tạm ngừng gửi
traffic. Khi Redis về, `/ready` tự trả 200 lại. Tôi đã thử dừng Redis khi app
đang chạy: `/health` vẫn 200, `/ready` trả 503 `{"status": "not ready",
"redis": false}`; bật Redis lại thì `/ready` về 200 mà không cần restart app.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Tôi scale lên 3 container agent. Vì `ports: "8000:8000"` chỉ cho một container
chiếm cổng 8000, tôi dùng một file override tạm để mỗi container mở một cổng
(8000, 8001, 8002), rồi gọi luân phiên từng cổng với cùng `X-User-Id:
sv-scale` để mô phỏng load balancer round-robin. (Tôi định dùng nginx có sẵn
nhưng lúc đó máy không tải được image từ Docker Hub.)

| Lượt | Container (cổng) | `history_length` |
|---|---|---|
| 1 | 8000 | 0 |
| 2 | 8001 | 2 |
| 3 | 8002 | 4 |
| 4 | 8000 | 6 |
| 5 | 8001 | 8 |
| 6 | 8002 | 10 |

Con số tăng đều 2 mỗi lượt dù mỗi lượt vào một container khác, vì cả 3 đọc và
ghi chung một Redis.

Nếu lưu bằng dict Python, mỗi container có một dict riêng trong RAM của nó. Với
cùng cách gọi luân phiên, kết quả sẽ là 0, 0, 0, 2, 2, 2: mỗi container chỉ
nhớ những lượt rơi vào chính nó, agent "mất trí nhớ" một cách ngẫu nhiên tùy
request vào container nào. Ngoài ra khi một container restart hoặc deploy bản
mới, dict của nó mất sạch, `history_length` quay về 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

**Lỗi:** deploy lên Render bằng Blueprint xong, service báo Live và `/health`
trả 200, nhưng `/ready` và cả `/ask` (không gửi key) đều trả
`500 Internal Server Error`.

**Tìm nguyên nhân:**

1. Suy luận từ mã lỗi: nếu chỉ `REDIS_URL` sai thì `/ask` không có key vẫn
   phải trả 401, vì kiểm tra key chạy trước. `/ask` cũng 500 nghĩa là lỗi nằm
   ngay trong `verify_api_key`, mà dòng đầu của nó là `get_settings()`.
2. Mở tab Logs của service trên Render, thấy traceback:
   `pydantic_core.ValidationError: 1 validation error for Settings`
   `agent_api_key  Field required [type=missing, input_value={'port': '10000', ...}]`.
   Tức là container không nhận được biến `AGENT_API_KEY`. Lúc tạo Blueprint,
   bước tạo Key Value bị treo khá lâu, có lẽ giá trị tôi dán vào ô
   `AGENT_API_KEY` không được lưu.
3. Điều đáng chú ý là ngay sau traceback, log vẫn đầy các dòng
   `GET /health 200 OK`, nên Render tưởng service khoẻ.

**Sửa:**

- Thêm `AGENT_API_KEY` trong tab Environment. Container đang chạy không tự đọc
  biến mới, nên phải deploy lại thì mới có hiệu lực. Sau đó `/ready` trả 200,
  `/ask` không key trả 401, có key trả 200.
- Sửa lỗ hổng trong code: `get_settings()` có `lru_cache` và chỉ được gọi khi
  có request đầu tiên cần config, nên "fail fast" của CP1 thực ra không fast:
  thiếu secret mà app vẫn khởi động, `/health` vẫn 200. Tôi thêm lời gọi
  `get_settings()` vào `lifespan` để đọc config ngay lúc khởi động. Thử lại ở
  máy khi thiếu key: uvicorn báo `Application startup failed` và thoát. Trên
  Render, lần sau thiếu biến thì deploy sẽ bị đánh dấu thất bại và bản cũ vẫn
  chạy, thay vì lên Live với một service hỏng.

Bài học: health check trả 200 chưa chắc service dùng được; fail fast phải xảy
ra lúc khởi động thì mới chặn được deploy hỏng.
