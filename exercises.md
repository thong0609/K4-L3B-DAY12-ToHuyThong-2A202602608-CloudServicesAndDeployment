# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.

> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Tô Huy Thông Mã học viên: 2A202602608

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Ví dụ khi tạo service Railway mới, nếu quên khai báo AGENT_API_KEY thì Settings phải báo lỗi thay vì dùng khóa mặc định "changeme" mà người ngoài có thể đoán được. Nhờ vậy cấu hình thiếu được phát hiện trước khi API xử lý yêu cầu có xác thực.

Trong code hiện tại, get_settings() được gọi lười qua dependency, nên thiếu khóa có thể chỉ gây lỗi khi gọi /ask hoặc /ready; /health vẫn trả 200. Do đó “fail fast” ở đây chính xác là lỗi ngay khi khởi tạo Settings, chưa phải bảo đảm process chết ngay lúc khởi động.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log thực tế thu được khi gọi /ask trên Docker:

```json
{
  "event": "ask_completed",
  "level": "info",
  "timestamp": "2026-09-29T05:16:53.525044+00:00",
  "user_id": "scale-report-b4abb0fedfbd4887b3052a2dbc804325",
  "tokens_in": 4,
  "tokens_out": 38,
  "cost_usd": 2.34e-5
}
```

Hai ứng dụng cụ thể: (1) lọc theo user_id và timestamp để truy vết lượt hỏi của một user; (2) tổng hợp cost_usd và tokens_in/tokens_out để theo dõi chi phí, phát hiện lượt hỏi tốn nhiều token. Dòng “đã trả lời xong” không chứa các trường dữ liệu này.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản                       | Dung lượng                  |
| ------------------------- | --------------------------- |
| 1 stage (tái tạo bản đầu) | 446.51 MB (446512131 bytes) |
| Multi-stage               | 63.85 MB (63848401 bytes)   |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Số đo được ghi bằng docker image inspect, dùng cùng phương pháp cho hai image; MB = 1.000.000 bytes. Đây là số Size do Docker Engine báo trên máy này, không phải đo RAM hay dung lượng truyền qua mạng. Bản một stage tái tạo các lệnh Dockerfile ban đầu trên source hiện tại và vẫn dùng .dockerignore an toàn.

Chênh lệch không chỉ do số stage: bản đầu dùng Python đầy đủ, COPY toàn repo và pip có cache; bản mới dùng slim, tắt cache pip và chỉ copy dependency, app, utils vào runtime. Multi-stage giúp loại các thành phần chỉ phục vụ build; trong bài này không cài compiler riêng nên không thể quy toàn bộ chênh lệch cho compiler.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Đã thử thêm một dòng comment vào app/main.py, build lại rồi khôi phục nguyên file. Log cho thấy WORKDIR của builder, COPY requirements.txt, pip install, COPY --from=builder, WORKDIR runtime và lệnh tạo appuser đều CACHED. COPY app và COPY utils phải chạy lại.

Nếu COPY toàn bộ source đứng trước pip install trong cùng stage thì thay đổi source làm mất cache từ bước COPY đó, kéo theo chạy lại pip install dù requirements.txt không đổi. Tách dependency khỏi source giúp các lần sửa code build nhanh hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Một lỗ hổng thực thi mã trong Python có thể cho kẻ tấn công chạy lệnh với quyền của process ứng dụng. Nếu process chạy root, họ có quyền root bên trong container; kết hợp mount nhạy cảm, cấu hình privileged hoặc lỗ hổng thoát container thì có thể gây ảnh hưởng lớn tới host.

USER appuser làm mã bị chiếm quyền chỉ chạy với UID thường (10001), giảm khả năng sửa file hệ thống và lạm dụng đặc quyền. Root trong container không tự động đồng nghĩa root trên host; USER cũng không thay thế việc hạn chế mount, capability và cập nhật bảo mật.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Có thể gửi 20 request trong khoảng 2 giây quanh ranh giới phút: 10 request lúc 10:00:59 và 10 request lúc 10:01:00. Mỗi phút đồng hồ vẫn chỉ có 10 request.

Sliding window đếm 60 giây gần nhất nên 10 request cũ vẫn nằm trong cửa sổ khi phút mới bắt đầu. Trong code, entry hết cửa sổ bị xóa trước khi đếm; timestamp kèm UUID giúp các request cùng thời điểm không ghi đè nhau.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số request trong 60 giây, cost guard giới hạn tổng chi phí theo user/tháng.

Ví dụ user mới gửi một request trong phút nhưng đã tiêu 10.01 USD với ngân sách 10 USD: rate limit cho qua, cost guard trả 402. Ngược lại, user mới tiêu 0.01 USD nhưng đã gửi 10 request trong cửa sổ 60 giây: request thứ 11 bị trả 429 dù ngân sách còn nhiều.

Theo code lab, check chặn khi spent + estimated_cost > budget. /ask gọi check trước LLM rồi record chi phí thật sau đó; vì không truyền ước tính nên một lượt vẫn có thể làm vượt ngân sách trước khi lượt tiếp theo bị chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự có thể xảy ra: Redis mất kết nối → probe dùng chung kiểm tra Redis thất bại ở cả ba container → readiness loại chúng khỏi luồng nhận request → nếu liveness cũng dùng probe đó và vượt ngưỡng lỗi, orchestrator có thể restart cả ba → app khởi động lại nhưng Redis vẫn lỗi nên probe tiếp tục thất bại.

Tách hai endpoint thì /health vẫn 200 khi process hoạt động; /ready trả 503 để ngừng nhận traffic cần Redis. Khi Redis phục hồi, /ready trở lại 200 mà không cần restart ứng dụng. Việc restart còn tùy cấu hình orchestrator; Docker HEALTHCHECK đơn thuần chỉ đánh dấu unhealthy, không tự restart container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Đã kiểm tra ba container agent cùng kết nối một Redis: gọi tuần tự cùng X-User-Id vào A, B, C nhận history_length lần lượt 0, 2, 4. Mỗi lượt thêm hai message nên container sau đọc được dữ liệu container trước ghi.

Do Compose hiện map cố định 8000:8000, phép thử dùng agent đang chạy cộng hai container từ docker compose run --no-deps, không publish cổng của hai container phụ; gọi API bên trong từng container. Đây là kiểm tra ba instance thật, không phải chạy thành công nguyên lệnh --scale trong đề. Hai container phụ đã được dọn sau thử nghiệm.

Nếu dùng dict riêng trong process, lần đầu đến A, B, C đều có thể nhận 0; quay lại A mới thấy 2. Lịch sử phụ thuộc instance nhận request và biến mất khi process restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi quan sát trên Railway: /health trả 200 nhưng /ready và /ask ban đầu trả “500 Internal Server Error”. Phản hồi này chỉ cho biết lỗi phía server; muốn biết nguyên nhân phải xem traceback trong Deployments → Logs và đối chiếu Variables cùng phiên bản code đã deploy.

Đã kiểm tra lại cấu hình API key, kết nối Redis và bản triển khai theo hướng dẫn. Sau đó phép kiểm tra thực tế ngày 29/09/2026 cho thấy /health=200, /ready=200, /ask không key=401, có key=200; request thứ 11 cùng user trả 429.

Không lưu được traceback của lần lỗi ban đầu nên chưa đủ bằng chứng kết luận thiếu AGENT_API_KEY hay sai REDIS_URL là nguyên nhân. Chủ repo cần bổ sung chính xác biến hoặc thao tác đã sửa từ lần deploy thực tế trước khi nộp; không dùng phỏng đoán làm kết luận.
