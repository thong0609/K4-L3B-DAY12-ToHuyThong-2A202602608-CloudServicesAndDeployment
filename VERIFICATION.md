# Kết quả kiểm tra thực tế — 2026-09-29

Cloud: https://k4-l3b-day12-tohuythong-2a202602608-cloudservice-production.up.railway.app

- GET /health: 200 {"status":"ok","service":"day12-agent","version":"1.0.0"}
- GET /ready: 200 {"status":"ready","redis":true}
- POST /ask không key: 401 {"detail":"invalid or missing API key"}
- POST /ask với khóa cục bộ: 200
```json
{
  "answer": "Theo mình hiểu, Docker la gi liên quan tới cách hệ thống được đóng gói và vận hành. Điểm mấu chốt là tách cấu hình ra khỏi code và giữ service ở trạng thái stateless.",
  "user_id": "report-34e4bff757af41bf98a29ecfa8e57e3f",
  "history_length": 0,
  "cost_usd": 2.505e-05,
  "tokens": {
    "in": 3,
    "out": 41
  }
}
```
- 11 request liên tiếp cùng user: [200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 429]

## Bộ test
Lần kiểm tra đầu CP1–CP5: 78 passed, 5 skipped. Thiếu DEPLOY_API_KEY cho một test; bốn test fallback không áp dụng. Kiểm tra HTTP riêng có key đã thành công.

Image multi-stage: 63848401 bytes (60.89 MiB), đo bằng docker image inspect.

Điểm tự chấm ban đầu: 85/100; exercises.md chưa trả lời (0/15). Không làm bonus CI/CD.

## Ba container dùng chung Redis
- Container A: history_length=0
- Container B: history_length=2
- Container C: history_length=4

## Log JSON thật (câu 2)
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:16:53.525044+00:00", "user_id": "scale-report-b4abb0fedfbd4887b3052a2dbc804325", "tokens_in": 4, "tokens_out": 38, "cost_usd": 2.34e-05}
```

CP5 kiểm tra lại sau khi cấu hình DEPLOY_API_KEY: 9 passed, 4 skipped (local fallback không áp dụng).

## Bổ sung câu 3–4

Single-stage: 446512131 bytes (446.51 MB). Multi-stage: 63848401 bytes (63.85 MB).

Build sau thay đổi comment main.py: dependency và useradd CACHED; COPY app và COPY utils chạy lại. Source đã khôi phục sau thử nghiệm.
