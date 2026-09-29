# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: viết câu trả lời chi tiết bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lâm Hải Dương  Mã học viên: 2A202602676

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy gấp lên Railway/Render vào môi trường production nhưng quên cấu hình biến `AGENT_API_KEY` trên dashboard: nếu để mặc định `"changeme"`, container vẫn khởi động xanh (`200 OK`), và vì mã nguồn trên GitHub là public nên các bot quét repo sẽ biết ngay khóa mặc định `"changeme"` để gọi `/ask` miễn phí liên tục, làm cạn sạch ngân sách LLM trước khi mình kịp nhận ra. Ngược lại, khi không để giá trị mặc định, `pydantic-settings` ném `ValidationError` làm container dừng ngay lúc khởi động (Fail Fast), báo đỏ trên dashboard deploy ngay lập tức để mình phát hiện và bổ sung biến môi trường trước khi dịch vụ mở cửa nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thực tế thu được khi gọi `/ask`:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:12:07.588409+00:00", "user_id": "2A202602676", "tokens_in": 7, "tokens_out": 46, "cost_usd": 2.865e-05}`
>
> Hai việc làm được với dòng log JSON này mà `print("đã trả lời xong")` không làm được:
> 1. **Thống kê và truy vết chi phí tự động theo từng người dùng:** Hệ thống quản lý log (Railway Logs, Datadog, CloudWatch) có thể parse các trường `user_id`, `cost_usd`, `tokens_in`, `tokens_out` để tính tổng tiền tiêu thụ của từng user theo giờ/ngày hoặc tìm ra ngay user nào đang gửi prompt bất thường gây tốn token.
> 2. **Lọc theo mức độ sự cố và cài đặt cảnh báo tự động (Alerting):** Có thể viết query lọc chính xác theo `level == "error"` hoặc `event == "ask_completed"` kèm điều kiện `cost_usd > 0.05` trong khoảng thời gian `timestamp` 5 phút gần nhất để tự động bắn cảnh báo về Slack/Email.

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
| 1 stage (bản đầu) | 1120 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch khoảng **~849 MB** bao gồm:
> 1. Các gói công cụ biên dịch và thư viện phát triển hệ thống Debian đi kèm trong base image `python:3.11` đầy đủ (`gcc`, `g++`, `make`, `libc-dev`, header files, `git`, `curl`, tài liệu manpages...) vốn không có trong bản `python:3.11-slim`.
> 2. Bộ nhớ đệm tải gói của pip (`~/.cache/pip`) và các file tạm sinh ra trong quá trình build wheel ở stage `builder` (đã bị loại bỏ nhờ `--no-cache-dir` và chỉ `COPY --from=builder /install /usr/local` sang stage `runtime`).
> 3. Các thư mục thừa ở máy host như `.venv`, `.git`, `.pytest_cache`, `tests` vốn bị lệnh `COPY . .` ở bản đầu chép hết vào image, nay đã được chặn bởi `.dockerignore` và chỉ copy đúng `app/` cùng `utils/`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - **Với Dockerfile hiện tại:** Khi sửa một ký tự trong `app/main.py`, toàn bộ các layer ở stage `builder` (`WORKDIR /build`, `COPY requirements.txt .`, `RUN pip install ...`) và các layer đầu của stage `runtime` (`WORKDIR /app`, `COPY --from=builder /install /usr/local`) đều được lấy thẳng từ cache (`CACHED` — tốn 0.0s). Chỉ có các layer từ `COPY app ./app`, `COPY utils ./utils` và `RUN useradd ...` trở xuống là phải chạy lại, giúp tổng thời gian build lại chỉ mất khoảng **2 giây**.
> - **Nếu đặt `COPY . .` lên trước `RUN pip install`:** Ngay khi `app/main.py` thay đổi 1 ký tự, checksum của layer `COPY . .` thay đổi làm mất hiệu lực (invalidate) cache của chính nó và toàn bộ các layer phía sau. Hệ quả là Docker buộc phải chạy lại `RUN pip install -r requirements.txt`, tải và cài đặt lại toàn bộ thư viện từ đầu dù `requirements.txt` không hề thay đổi (mất vài phút mỗi lần sửa code).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> - **Chuỗi sự kiện tấn công khi chạy bằng `root`:** (1) Kẻ tấn công khai thác một lỗ hổng trong ứng dụng Python hoặc thư viện bên thứ ba (như Remote Code Execution / Command Injection) để chạy lệnh shell bên trong container $\rightarrow$ (2) Vì container mặc định chạy tiến trình với `root` (UID 0), kẻ tấn công có ngay quyền `root` bên trong container $\rightarrow$ (3) Do container dùng chung nhân Linux (kernel) với máy host, nếu kết hợp với một lỗ hổng kernel/container runtime (container escape) hoặc cấu hình mount nhạy cảm (như mount `/var/run/docker.sock` hay thư mục hệ thống của host), quyền UID 0 trong container sẽ ánh xạ trực tiếp thành quyền `root` toàn quyền trên máy chủ vật lý (host).
> - **Nơi lệnh `USER appuser` cắt đứt chuỗi:** Lệnh `USER appuser` (UID 10001) chặn đứng chuỗi tấn công ngay ở bước (2). Khi tiến trình Uvicorn chạy dưới quyền user thường không có đặc quyền (unprivileged user), dù kẻ tấn công có chiếm được shell trong container thì cũng không thể cài đặt công cụ, không sửa được file hệ thống trong image, và không có đặc quyền UID 0 để thực hiện leo thang đặc quyền sang máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Nếu đếm theo phút đồng hồ (Fixed Window, reset vào giây `:00` mỗi phút) với hạn mức 10 request/phút, một người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
>
> **Cách đạt được:** Người dùng đợi đến giây cuối cùng của phút hiện tại (ví dụ lúc `10:00:59`) và bắn liền **10 request** — cả 10 đều hợp lệ vì phút `10:00` mới ghi nhận 10 request. Ngay giây tiếp theo (`10:01:00`), đồng hồ bước sang phút mới nên bộ đếm bị xóa về `0`, người dùng lập tức bắn thêm **10 request** nữa (vẫn hợp lệ trong khung phút `10:01`). Như vậy chỉ trong 2 giây từ `10:00:59` đến `10:01:00`, hệ thống phải gánh **20 request** (gấp đôi hạn mức thiết kế). Dùng Sliding Window 60 giây với Redis ZSET sẽ chặn ngay loạt thứ hai vì tại `10:01:00`, cửa sổ trượt `[10:00:00, 10:01:00]` vẫn còn lưu đủ 10 timestamp của giây `10:00:59`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - **Điểm khác nhau:** **Rate limit** kiểm soát **tốc độ/tần suất gọi** trong thời gian ngắn (số request trên mỗi 60 giây, trả mã `429`) nhằm chống spam và quá tải hệ thống; còn **Cost guard** kiểm soát **tổng chi phí tiền tệ tích lũy** trong dài hạn (tổng USD tiêu thụ trong cả tháng, trả mã `402`) nhằm bảo vệ ngân sách tài chính.
> - **Tình huống Rate limit cho qua nhưng Cost guard phải chặn:** Một user gọi API rất thưa (ví dụ 1 request mỗi 5 phút — thấp hơn nhiều so với giới hạn 10 request/phút nên Rate limit luôn cho qua), nhưng mỗi request lại gửi kèm câu hỏi rất dài và lịch sử hội thoại lớn tốn nhiều token, làm tổng chi phí cộng dồn trong tháng vượt ngưỡng `10.0 USD`. Lúc này Cost guard sẽ chặn lại bằng lỗi `402 Payment Required`.
> - **Tình huống Cost guard cho qua nhưng Rate limit phải chặn:** Một user mới tinh trong tháng (`spent = 0.0 USD`, còn nguyên ngân sách `10.0 USD` nên Cost guard cho qua) nhưng dùng script bắn dồn dập 15 câu hỏi cực ngắn (`"hi"`, mỗi câu chỉ tốn vài phần trăm nghìn đô) trong vòng 3 giây. Khi đó từ request thứ 11 trở đi sẽ bị Rate limit chặn ngay bằng lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Khi gộp `/health` (liveness) và `/ready` (readiness) làm một và bắt nó kiểm tra kết nối Redis, nếu Redis mất kết nối trong 30 giây thì chuỗi sự kiện diễn ra theo thứ tự sau:
> 1. **Giây 0:** Redis gặp sự cố mạng tạm thời hoặc đang failover nên không phản hồi `ping()`.
> 2. **Giây 5–15:** Orchestrator (Docker/Kubernetes/Railway) gọi Liveness probe vào `/health` của cả 3 container `agent`; do `store.ping()` trả `False`, cả 3 container đồng loạt trả mã `503 Service Unavailable`.
> 3. **Ngay khi vượt số lần `retries`:** Orchestrator hiểu nhầm rằng bản thân tiến trình Python/FastAPI trong cả 3 container bị treo (deadlock), nên ra lệnh **kill và restart đồng loạt cả 3 container**.
> 4. **Trong lúc restart:** Các request đang xử lý dở bị ngắt đột ngột, và các container mới bật lên vẫn tiếp tục không nối được Redis nên lại bị kill tiếp (rơi vào vòng lặp CrashLoopBackOff).
> 5. **Giây 30:** Dù Redis đã hoạt động bình thường trở lại, toàn bộ 3 container web vẫn đang tắt hoặc đang chờ khởi động lại, biến một sự cố chập chờn 30 giây của Redis thành sập toàn bộ hệ thống (downtime kéo dài).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> - **Khi lưu lịch sử trong Redis (Stateless):** Qua mỗi lần gọi `/ask` với cùng một `X-User-Id`, trường `history_length` trong response tăng đều đặn thêm 2 sau mỗi lượt hỏi (`0 -> 2 -> 4 -> 6 -> 8...` cho tới khi chạm trần `20`), bất kể request đó được điều phối vào container nào trong 3 container `agent`.
> - **Nếu lưu lịch sử trong một `dict` Python trên RAM của từng process (Stateful):** Vì mỗi container có vùng nhớ RAM tách biệt, khi Load Balancer phân phối request theo vòng tròn (Round-Robin) qua 3 container A, B, C: lượt hỏi 1 vào container A sẽ thấy `history_length = 0` (và chỉ lưu vào RAM của A); lượt hỏi 2 vào container B lại thấy `history_length = 0`; lượt hỏi 3 vào container C cũng thấy `history_length = 0`; đến lượt hỏi 4 quay lại container A mới thấy `history_length = 2`. Con số `history_length` sẽ bị nhảy thất thường (`0, 0, 0, 2, 2, 2, 4...`) và agent liên tục "quên" nội dung vừa trao đổi ở lượt trước.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - **Lỗi gặp phải:** Trong quá trình build Docker image và chuẩn bị deploy lên Railway, bước cài đặt thư viện `RUN pip install --no-cache-dir --prefix=/install -r requirements.txt` bị dừng giữa chừng với thông báo lỗi: `pip._vendor.urllib3.exceptions.ReadTimeoutError: HTTPSConnectionPool(host='files.pythonhosted.org', port=443): Read timed out` (và trên Railway ban đầu `/ready` trả `503 Service Unavailable` nếu chưa liên kết biến `REDIS_URL` từ service Redis sang service `ai-agent`).
> - **Cách tìm ra nguyên nhân:** Đọc trực tiếp build log của Docker và `railway logs` trên terminal: thấy `pip` mặc định có timeout ngắn (15 giây) nên khi tải các gói nặng như `uvloop`/`pydantic_core` lúc mạng chập chờn sẽ bị ngắt kết nối; đồng thời trên Railway khi thêm database Redis (`railway add --database redis`), biến `REDIS_URL` nằm ở service `Redis` chứ không tự động gắn sang service `ai-agent`.
> - **Cách sửa:** Bổ sung cờ `--default-timeout=120 --retries 10` vào lệnh `pip install` trong `Dockerfile` để tăng sức chịu lỗi mạng khi build, và dùng lệnh `railway variable set "REDIS_URL=\${{Redis.REDIS_URL}}" --service ai-agent` để tham chiếu biến `REDIS_URL` từ service `Redis` sang `ai-agent`, giúp cả `/health` và `/ready` trên URL HTTPS công khai đều trả về `200 OK`.
