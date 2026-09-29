# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền trực tiếp câu trả lời vào dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Cao Văn Trường  Mã học viên: 2A202602562

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu đặt giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động bình thường và được deploy lên môi trường production dù lập trình viên sơ suất quên cấu hình biến môi trường `AGENT_API_KEY`. Khi đó, bất kỳ kẻ tấn công hoặc bot quét tự động nào trên Internet chỉ cần gửi header `X-API-Key: changeme` là có thể tự do gọi vào endpoint `/ask` và tiêu tốn toàn bộ ngân sách tài khoản LLM (OpenAI/Anthropic). Ngược lại, cơ chế Fail-Fast bắt buộc không có default: nếu thiếu biến môi trường, Pydantic sẽ ném `ValidationError` và làm ứng dụng crash ngay tại thời điểm khởi động, ngăn chặn hoàn toàn việc một service chưa được bảo mật lọt ra ngoài Internet.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

- Dòng log JSON thu được:
`{"event": "ask_completed", "level": "INFO", "timestamp": "2026-09-29T02:49:44.123456Z", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 25, "cost_usd": 0.00037}`

- Hai việc làm được với Structured JSON log mà `print("đã trả lời xong")` không làm được:
  1. **Lọc và truy vấn tự động trên hệ thống tập trung (ELK / CloudWatch / Datadog)**: Có thể dễ dàng truy vấn chính xác theo trường dữ liệu như `event == "ask_completed" AND user_id == "sv-test"` hoặc tìm các request có `cost_usd > 0.01` mà không cần phải viết regex để bóc tách chuỗi phức tạp.
  2. **Thống kê và cảnh báo theo thời gian thực (Aggregation & Alerting)**: Có thể thực hiện tính toán ngay lập tức như tính tổng chi phí (`SUM(cost_usd)`), tổng số token tiêu thụ của từng user trong ngày, hoặc cấu hình cảnh báo tự động khi chi phí tăng đột biến.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | 305 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~715 MB) bao gồm:
1. **Base image Python đầy đủ**: Image `python:3.11` chứa sẵn toàn bộ bộ biên dịch GCC, build-essentials, các thư viện phát triển C/C++, header files và man pages của hệ thống — những thứ chỉ cần thiết lúc build package nhưng hoàn toàn thừa thãi khi chạy app. Stage runtime đã chuyển sang `python:3.11-slim` tối giản dựa trên Debian.
2. **Pip cache và build artifacts**: Quá trình `pip install` sinh ra các file `.whl`, tarball tạm trong `~/.cache/pip`. Multi-stage build tách biệt hoàn toàn việc cài đặt ở stage `builder`, sang stage `runtime` chỉ copy thư mục `/opt/venv` sạch sẽ, để lại toàn bộ rác build ở stage trước.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại:
  - Tất cả các layer từ đầu đến `RUN pip install` (bao gồm `FROM`, `WORKDIR`, `RUN python -m venv`, `COPY requirements.txt .`, `RUN pip install ...`, và `COPY --from=builder /opt/venv`) đều được **tận dụng lại từ Docker Cache (CACHED)** do nội dung file `requirements.txt` không hề thay đổi.
  - Chỉ có layer `COPY . .` ở stage runtime và các bước sau nó mới phải chạy lại, quá trình rebuild chỉ mất chưa tới 1 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  - Mỗi khi sửa dù chỉ một ký tự trong code, checksum của thư mục thay đổi làm layer `COPY . .` bị vô hiệu hóa cache (cache invalidated).
  - Khi đó, Docker bắt buộc phải thực thi lại toàn bộ layer phía sau, bao gồm cả việc tải và cài đặt lại toàn bộ thư viện từ Internet (`pip install`), khiến thời gian build tăng lên hàng phút và lãng phí băng thông CI/CD.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện khi chạy dưới quyền root:
  1. Kẻ tấn công khai thác thành công một lỗ hổng thực thi mã từ xa (RCE) trong ứng dụng Python (ví dụ qua Prompt Injection, deserialization bug, hoặc shell command injection).
  2. Do process trong container chạy dưới quyền root (UID 0), kẻ tấn công chiếm được quyền root bên trong container.
  3. Từ quyền root này, kẻ tấn công khai thác các lỗ hổng bảo mật của Linux Kernel hoặc tương tác với Docker daemon socket (nếu bị mount nhầm) để thực hiện hành vi thoát khỏi container (container escape).
  4. Do UID 0 trong container mặc định ánh xạ với UID 0 (root) trên máy Host vật lý, kẻ tấn công chiếm quyền kiểm soát toàn bộ máy host.
- Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay tại **Bước 2**: Tiến trình ứng dụng bị giới hạn ở quyền người dùng thông thường (`appuser`), kẻ tấn công dù chèn được mã cũng không thể đọc/ghi các file nhạy cảm của hệ thống, không thể cài công cụ độc hại và không đủ đặc quyền hệ điều hành (capabilities) để tiến hành container escape.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 requests** trong 2 giây liên tiếp.
- Giải thích:
  - Với Fixed Window (cửa sổ cố định reset ở giây 00):
    - Người dùng gửi 10 requests vào giây thứ `00:59` (thuộc phút thứ 0), tiêu thụ hết hạn mức của phút đó.
    - Đúng 1 giây sau, khi đồng hồ chuyển sang `01:00`, bộ đếm được reset về 0. Người dùng lập tức gửi tiếp 10 requests nữa vào giây `01:00` (thuộc phút thứ 1).
    - Kết quả: Từ giây `00:59` đến `01:00` (chỉ trong 2 giây liên tiếp), người dùng đã gửi thành công 20 requests — gấp đôi hạn mức 10 req/phút cho phép, dễ gây sập hệ thống.
  - Thuật toán **Sliding Window** (dùng Redis Sorted Set) khắc phục lỗi này bằng cách luôn tính tổng số request trong đúng 60 giây gần nhất tính từ thời điểm hiện tại (`now - 60s`), đảm bảo không có bất kỳ khoảng 60 giây trượt nào vượt quá 10 requests.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Sự khác nhau:
  - **Rate Limit** kiểm soát **tần suất/tốc độ** (Velocity - ví dụ request/phút) để ngăn ngừa nghẽn cổ chai mạng, DoS hoặc request dồn dập trong khoảng thời gian ngắn.
  - **Cost Guard** kiểm soát **tổng chi phí tài chính tích lũy** (Financial Budget - ví dụ tổng số tiền USD/tháng) dựa trên lượng token LLM thực tế tiêu thụ.
- Tình huống Rate limit cho qua nhưng Cost guard chặn:
  - Một user chỉ gửi đúng 1 request trong ngày (tần suất cực thấp, hoàn toàn hợp lệ với Rate Limit 10 req/phút). Tuy nhiên trong tháng đó, user này đã hỏi nhiều câu dài và tổng chi phí đã chạm trần $10.0 $\rightarrow$ Cost Guard phát hiện vượt ngân sách và trả về `402 Payment Required`.
- Tình huống Cost guard cho qua nhưng Rate limit chặn:
  - Một user mới tạo tài khoản, ngân sách còn nguyên $10.0 chưa tiêu đồng nào. Tuy nhiên user này dùng script gửi dồn dập 15 requests chỉ trong 2 giây $\rightarrow$ Cost Guard kiểm tra thấy vẫn đủ tiền, nhưng Rate Limiter chặn từ request thứ 11 và trả về `429 Too Many Requests` để bảo vệ server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện sụp đổ dây chuyền (Cascading Failure / CrashLoopBackOff):
1. Redis bị nghẽn mạng hoặc khởi động lại tạm thời trong 30 giây.
2. Endpoint gộp kiểm tra kết nối Redis thất bại $\rightarrow$ trả về lỗi 503.
3. Vì đây đóng vai trò là Liveness probe, Orchestrator (Docker/K8s) nhận định tiến trình của container đã bị hỏng/deadlock $\rightarrow$ tiến hành **kill và restart** đồng loạt cả 3 container agent.
4. Khi cả 3 container mới khởi động lại, Redis vẫn chưa online trong khoảng 30s đó $\rightarrow$ probe tiếp tục thất bại $\rightarrow$ Orchestrator tiếp tục restart vòng lặp vô tận (CrashLoopBackOff).
5. Hậu quả: Toàn bộ request đang xử lý dở bị đứt gãy, gây downtime toàn diện. Trong khi nếu tách riêng `/ready`, các container vẫn sống bình thường, chỉ tạm thời ngắt traffic và sẽ tự động phục vụ trở lại ngay khi Redis kết nối lại thành công.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trong Redis (Stateless): `history_length` tăng đều đặn và chính xác qua từng lượt hội thoại (`0 -> 2 -> 4 -> 6...`) bất kể request được điều hướng vào bất kỳ container nào trong 3 container.
- Nếu lưu trong một dict Python trong RAM (Stateful):
  - Do Load Balancer chia đều request sang 3 container theo cơ chế Round-Robin, mỗi container sở hữu một vùng nhớ RAM tách biệt.
  - Con số `history_length` sẽ bị nhảy lộn xộn, ngắt quãng:
    - Request 1 vào container 1: `history_length = 0` (container 1 lưu turn 1).
    - Request 2 bị đẩy sang container 2: `history_length = 0` (container 2 hoàn toàn không biết gì về lượt hỏi trước!).
    - Request 3 bị đẩy sang container 3: `history_length = 0` (container 3 cũng rỗng).
    - Request 4 quay lại container 1: `history_length = 2`.
  - Hậu quả: Agent bị "mất trí nhớ", LLM mất ngữ cảnh của cuộc hội thoại và trả lời sai lệch.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi**: Sau khi deploy lên Railway, endpoint `/health` hoạt động bình thường (200 OK) nhưng cả hai endpoint `/ready` và `/ask` đều trả về `500 Internal Server Error`.
- **Cách tìm ra nguyên nhân**:
  1. Nhận thấy `/health` không có dependency, trong khi `/ready` và `/ask` đều phụ thuộc vào `get_settings()`.
  2. Trong `app/config.py`, biến `agent_api_key: str` được cấu hình fail-fast không có giá trị mặc định.
  3. Kiểm tra tab **Variables** trên Railway Dashboard thì phát hiện chưa cấu hình bất kỳ biến môi trường nào (`No Environment Variables`), dẫn đến Pydantic ném ra ngoại lệ `ValidationError` khi khởi tạo `Settings`.
- **Cách sửa**:
  1. Vào tab **Variables** của service Agent trên Railway, mở **Raw Editor**.
  2. Bổ sung đầy đủ 5 biến môi trường: `AGENT_API_KEY` (khóa bí mật), `REDIS_URL` (`${{Redis.REDIS_URL}}` kết nối Private Network với Redis), `RATE_LIMIT_PER_MINUTE`, `MONTHLY_BUDGET_USD` và `LOG_LEVEL`.
  3. Bấm áp dụng biến để Railway redeploy lại container. Sau khi redeploy, `/ready` lập tức trả về `200 {"status":"ready","redis":true}` và `/ask` trả về `401` bảo mật thành công.
