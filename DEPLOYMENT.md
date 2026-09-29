# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

**Trạng thái: chưa deploy lên cloud, CP5 chưa hoàn tất.** Cấu hình Render
Blueprint đã được chuẩn bị. Phiên làm việc hiện chưa có trình duyệt kết nối
để truy cập tài khoản cloud, Railway CLI hoặc Docker để chạy fallback.
Chưa có Public URL, output HTTP từ cloud hoặc ảnh chụp xác minh.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đỗ Đình Long (theo tên repository) |
| Mã học viên | 2A202602673 |
| Repo | https://github.com/long2110d/K4-L3B-DAY12-DoDinhLong-2A202602673-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://TODO-thay-bang-url-that.up.railway.app |
| Platform | Render — dự kiến, chưa triển khai |
| Ngày deploy | (điền ngày) |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | Chưa xác minh | platform tự gán, không ghi đè |
| `AGENT_API_KEY` | Chưa set trên cloud | nhập khi tạo Blueprint, `sync: false` |
| `REDIS_URL` | Chưa set trên cloud | Blueprint liên kết connectionString của Render Key Value |
| `RATE_LIMIT_PER_MINUTE` | Đã khai báo trong Blueprint | 10; chưa deploy |
| `MONTHLY_BUDGET_USD` | Đã khai báo trong Blueprint | 10.0; chưa deploy |
| `LOG_LEVEL` | Đã khai báo trong Blueprint | INFO; chưa deploy |

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
(điền output)
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

```
(điền lý do nếu dùng phương án dự phòng, ngược lại xóa mục này)
```
