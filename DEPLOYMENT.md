# Thông Tin Deploy — Checkpoint 5

## Thông tin học viên

| Mục         | Nội dung                                                                                    |
| ----------- | ------------------------------------------------------------------------------------------- |
| Họ và tên   | Tô Huy Thông                                                                                |
| Mã học viên | 2A202602608                                                                                 |
| Repo        | https://github.com/thong0609/K4-L3B-DAY12-ToHuyThong-2A202602608-CloudServicesAndDeployment |

## Trạng thái

| Mục                  | Nội dung                                                                            |
| -------------------- | ----------------------------------------------------------------------------------- |
| Public URL           | https://k4-l3b-day12-tohuythong-2a202602608-cloudservice-production.up.railway.app/ |
| Platform             | Railway                                                                             |
| Ngày xác minh deploy | 29/09/2026                                                                          |

## Các bước triển khai Railway

1. Commit và push code đã kiểm tra lên repository trên.
2. Trong Railway, tạo project từ GitHub repo này.
3. Thêm Redis vào cùng project, đặt tên service là `Redis`.
4. Trong Variables của service agent, cấu hình các biến trong bảng dưới.
5. Deploy agent bằng Dockerfile. Không cần Start Command riêng: Dockerfile đã đọc PORT.
6. Chờ kiểm tra `/ready` thành công, tạo public domain trong Networking của service agent.
7. Ghi domain HTTPS thật vào dòng Public URL và ngày deploy vào bảng trên.
8. Chạy các lệnh kiểm tra và lưu ảnh dashboard, kết quả `/health`.

## Biến môi trường cần set trên cloud

Đã xác minh API key hoạt động và ứng dụng kết nối Redis. Các giá trị cấu hình khác cần đối chiếu dashboard. Không ghi giá trị secret vào tài liệu.

| Biến                    | Nguồn giá trị                                                         |
| ----------------------- | --------------------------------------------------------------------- |
| `PORT`                  | Railway cấp; Dockerfile đọc lúc khởi động                             |
| `AGENT_API_KEY`         | Khóa riêng do chủ repo tạo và đặt trong Variables của agent           |
| `REDIS_URL`             | Reference `${{Redis.REDIS_URL}}`; đổi `Redis` nếu service có tên khác |
| `RATE_LIMIT_PER_MINUTE` | `10`                                                                  |
| `MONTHLY_BUDGET_USD`    | `10.0`                                                                |
| `LOG_LEVEL`             | `INFO`                                                                |

## Kiểm tra bằng PowerShell

Nhập domain thật khi được hỏi:

```powershell
$deployUrl = (Read-Host 'Public URL HTTPS của agent').TrimEnd('/')
curl.exe -i "$deployUrl/health"
curl.exe -i "$deployUrl/ready"
'{"question":"Hello"}' | curl.exe -i -X POST "$deployUrl/ask" -H 'Content-Type: application/json' --data-binary '@-'
```

Mong đợi lần lượt: 200, 200, 401.

Để test có xác thực, đặt `DEPLOY_API_KEY` trong `.env` cục bộ bằng khóa của
service cloud; không commit `.env`. Sau khi cập nhật Public URL:

```powershell
.venv/Scripts/python.exe -m pytest tests/test_cp5.py -v
```

Đảm bảo `LOCAL_FALLBACK` không bật khi kiểm tra cloud. Test có key phải chạy,
không bị skip, để xác nhận `/ask` trả câu trả lời trên cloud.

## Kết quả chạy thật

Kết quả HTTP thật trên Railway ngày 29/09/2026:

| Kiểm tra                       | Kết quả                        |
| ------------------------------ | ------------------------------ |
| GET `/health`                  | 200, status=ok                 |
| GET `/ready`                   | 200, status=ready, redis=true  |
| POST `/ask` không key          | 401                            |
| POST `/ask` có key             | 200, có answer/tokens/cost_usd |
| 11 request liên tiếp cùng user | 10 lần 200, lần 11 trả 429     |

Xem output và thông tin kiểm thử trong [VERIFICATION.md](VERIFICATION.md).

## Ảnh minh chứng cần bổ sung

- `screenshots/dashboard.png`: service đang chạy trên Railway.

- `screenshots/health.png`: kết quả gọi `/health` trên domain công khai.
