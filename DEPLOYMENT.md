# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Lâm Hải Dương |
| Mã học viên | 2A202602676 |
| Repo | https://github.com/ElysiaTheElysier/K4-L3B-DAY12-LamHaiDuong-2A202602676-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://ai-agent-production-f1a4.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của Railway (`${{Redis.REDIS_URL}}`) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Các lệnh kiểm tra với Public URL `https://ai-agent-production-f1a4.up.railway.app`:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://ai-agent-production-f1a4.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://ai-agent-production-f1a4.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://ai-agent-production-f1a4.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://ai-agent-production-f1a4.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://ai-agent-production-f1a4.up.railway.app/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Output thực tế khi gọi vào `https://ai-agent-production-f1a4.up.railway.app`:

```text
1. GET /health:
HTTP/2 200 
content-type: application/json
{"status":"ok","service":"day12-agent","version":"1.0.0"}

2. GET /ready:
HTTP/2 200 
content-type: application/json
{"status":"ready","redis":true}

3. POST /ask (không có API key):
HTTP/2 401 
content-type: application/json
{"detail":"invalid or missing API key"}

4. POST /ask (có API key hợp lệ):
HTTP/2 200 
content-type: application/json
{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"sv-test","history_length":0,"cost_usd":2.145e-05,"tokens":{"in":3,"out":35}}

5. Rate limit (gọi 15 lần liên tiếp):
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform Railway
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
