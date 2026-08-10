# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Trần Quốc Việt |
| Mã học viên | 2A202601369 |
| Repo | https://github.com/viettqhe194224-cmd/K3-Day12-2A202601369-TranQuocViet |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-ss9b.onrender.com/ |
| Platform | Render |
| Ngày deploy | 10/08/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Render Key Value `day12-redis`, được liên kết tự động qua `render.yaml` |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
1. Liveness
{"status":"ok","service":"day12-agent","version":"1.0.0"}

2. Readiness
{"status":"ready","redis":true}

3. Không có API key
HTTP/1.1 401 Unauthorized
Date: Mon, 10 Aug 2026 04:51:39 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 313f0f94-ee28-4302
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a28c62f95fe35168-HKG
alt-svc: h3=":443"; ma=86400

{"detail":"invalid or missing API key"}
4. Có API key
answer         : Ngáº¯n gá»n: Hello phá»¥ thuá»c vÃo ba yáº¿u tá» â cáº¥u hÃ¬nh qua biáº¿n mÃ´i trÆ°á»á» orchestrator biáº¿t tráº¡ng thÃ¡i, vÃ giá» háº¡n tÃi nguyÃªn.
user_id        : sv-test
history_length : 0
cost_usd       : 2.115E-05
tokens         : @{in=1; out=35}

5. Rate limit
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429


## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

