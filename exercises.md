# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Đình Long — Mã học viên: 2A202602673
>
> Ghi chú: Các giải thích dưới đây dựa trên mã nguồn hiện tại. Những phần
> chưa có bằng chứng chạy thực tế được đánh dấu để bổ sung trước khi nộp.

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy một bản mới lên cloud, nếu quên khai báo `AGENT_API_KEY`, việc
`Settings` báo lỗi ngay khi được khởi tạo giúp phát hiện cấu hình thiếu trước
khi phục vụ API có xác thực. Nếu đặt mặc định là `"changeme"`, ứng dụng có thể
tiếp tục chạy với khóa dễ đoán, khiến người khác gọi API và tiêu ngân sách.

Trong code hiện tại, lỗi xuất hiện khi `get_settings()` được gọi lần đầu.
Lệnh `python -m app.main` đọc Settings trước khi chạy server, nhưng Docker
chạy `uvicorn app.main:app` và lifespan chưa đọc Settings, nên chưa bảo đảm
fail fast ngay lúc khởi động bằng Docker.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Chưa có dòng log thu từ lần chạy service thực tế để dán vào đây. Theo
`app/main.py` và `app/logging_utils.py`, sự kiện `ask_completed` có các trường
`event`, `level`, `timestamp`, `user_id`, `tokens_in`, `tokens_out`, `cost_usd`.

Hai việc có thể làm với log này:

1. Lọc các request của một `user_id` theo thời gian để truy vết hoạt động.
2. Cộng `cost_usd` và số token theo người dùng hoặc theo ngày để theo dõi
   mức sử dụng, phát hiện chi phí tăng bất thường.

Chuỗi `print("đã trả lời xong")` không chứa các dữ liệu cần thiết cho hai việc
trên. **Cần bổ sung:** một dòng JSON thật sau khi gọi thành công `/ask`.

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
| 1 stage (bản đầu) | Chưa đo |
| Multi-stage | Chưa đo |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Chưa có kết quả build hai image nên chưa thể ghi chênh lệch MB thực tế.
Dockerfile hiện tại dùng `python:3.11-slim` cho cả builder và runtime, cài
dependency vào `/opt/venv`, rồi chỉ copy virtualenv cùng `app/` và `utils/`
sang runtime.

So với bản một stage dùng `python:3.11` đầy đủ, phần giảm dung lượng chủ yếu
có thể đến từ base image slim không mang nhiều công cụ và thư viện hệ thống.
Multi-stage còn giúp loại các file chỉ phục vụ build khỏi runtime. Tuy nhiên,
Dockerfile hiện tại không cài thêm compiler ở builder, và multi-stage không
tự động nhỏ hơn một bản single-stage được tối ưu tương đương. Cần đo thật
để kết luận; không thể quy toàn bộ chênh lệch cho compiler hay pip cache,
vì lệnh pip hiện tại đã dùng `--no-cache-dir`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Dựa trên thứ tự lệnh hiện tại, nếu chỉ sửa `app/main.py` và các đầu vào khác
không đổi, toàn bộ stage builder có thể dùng cache, gồm tạo virtualenv,
`COPY requirements.txt` và `RUN pip install`. Trong runtime, các bước trước
`COPY app/ ./app/`, kể cả copy virtualenv từ builder, cũng có thể dùng cache.

`COPY app/ ./app/` bị mất cache; các bước sau đó như `COPY utils/ ./utils/`
và `RUN useradd` phải được xử lý lại theo chuỗi layer, còn các chỉ thị cấu
hình phía sau được áp dụng vào image mới. Nếu đặt `COPY . .` trước
`RUN pip install`, một thay đổi source cũng làm mất cache bước cài dependency,
khiến build tốn thời gian dù `requirements.txt` không đổi.

Đây là phân tích Dockerfile; chưa có output rebuild để xác nhận cache thực tế.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Một lỗ hổng thực thi mã trong ứng dụng có thể cho kẻ tấn công chạy lệnh với
quyền của process Python. Nếu process chạy root, kẻ tấn công có quyền root
trong container. Nếu còn có cấu hình nguy hiểm như mount tài nguyên host
nhạy cảm, hoặc có lỗ hổng kernel/container runtime cho phép thoát container,
kẻ tấn công có thể tiếp tục tác động tới host với quyền cao.

Lệnh `USER appuser` trong Dockerfile cho process chạy với UID 10001, hạn chế
quyền ngay ở bước thực thi mã trong container, chẳng hạn không được tùy ý sửa
file thuộc root. Root trong container không tự động đồng nghĩa với root trên
host; chạy non-root giảm mức thiệt hại nhưng không bảo đảm chặn mọi đường
leo thang đặc quyền hoặc thoát container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa 20 request trong hai giây nằm hai bên ranh giới phút: gửi 10 request
trong giây 12:00:59, rồi gửi thêm 10 request trong giây 12:01:00 sau khi bộ
đếm reset. Mỗi phút vẫn chỉ có 10 request nhưng khoảng thời gian rất ngắn
đã nhận 20 request.

Sliding window xét 60 giây gần nhất tại thời điểm gửi nên 10 request đầu
vẫn nằm trong cửa sổ khi nhóm thứ hai đến; nhóm thứ hai bị chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số request theo từng user trong 60 giây để tránh gọi quá
nhanh, trả HTTP 429 khi vượt mức. Cost guard theo dõi tổng chi phí của từng
user trong tháng UTC, trả HTTP 402 nếu chi phí đã ghi nhận cộng chi phí ước
tính vượt ngân sách.

- Rate limit cho qua, cost guard chặn: user chỉ gửi 1 request trong phút này,
  nhưng đã tiêu 10,01 USD trong tháng với ngân sách 10 USD.
- Cost guard còn cho phép, rate limit chặn: user mới tiêu 0,01 USD nhưng gửi
  request thứ 11 trong 60 giây với hạn mức 10 request/phút. Trong `/ask`,
  rate limit chạy trước nên request bị trả 429 trước khi gọi cost guard.

Ở code hiện tại, `/ask` không truyền `estimated_cost`, và điều kiện chặn là
`>` thay vì `>=`. Vì vậy, tổng chi phí đúng bằng ngân sách vẫn được cho qua;
request tiếp theo có thể khiến tổng vượt mức rồi mới bị chặn ở lần gọi sau.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu endpoint gộp được dùng cho cả liveness và readiness, thứ tự có thể là:

1. Redis mất kết nối; các probe của cả ba container không kiểm tra được Redis.
2. Khi đạt ngưỡng lỗi readiness, bộ điều phối loại các container khỏi danh
   sách nhận traffic, nên có thể không còn instance sẵn sàng.
3. Nếu đạt cả ngưỡng lỗi liveness trong khoảng mất kết nối, bộ điều phối
   restart các container dù process ứng dụng vẫn hoạt động. Restart không
   sửa được kết nối Redis và còn làm gián đoạn request đang xử lý.
4. Khi Redis phục hồi, các instance phải vượt qua kiểm tra readiness mới
   nhận traffic lại; instance bị restart còn phải chờ khởi động xong.

Việc có restart trong 30 giây hay không phụ thuộc chu kỳ probe, timeout và
ngưỡng lỗi. Riêng Docker Compose hiện tại chỉ đánh dấu health status khi
healthcheck lỗi, không tự restart container chỉ vì trạng thái `unhealthy`.
Tách `/health` chỉ kiểm tra process và `/ready` kiểm tra Redis giúp tránh
restart ứng dụng chỉ vì dependency tạm thời không sẵn sàng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis dùng chung, nếu gửi tuần tự các request thành công cho cùng user
có lịch sử ban đầu rỗng, `history_length` dự kiến là `0, 2, 4, 6, ...` và
tối đa 20. Response dùng độ dài lịch sử trước request hiện tại; mỗi lần hỏi
thêm hai message gồm câu hỏi và câu trả lời. Kết quả không phụ thuộc request
được chuyển tới instance nào, miễn các instance cùng kết nối Redis đó.

Nếu dùng dict riêng của mỗi process, lịch sử bị chia thành ba phần. Chẳng
hạn khi request luân phiên qua A, B, C, có thể thấy `0, 0, 0, 2, 2, 2, ...`.
Nếu phân phối không đều, con số có thể giảm khi chuyển sang instance có ít
lịch sử hơn; restart process cũng làm mất lịch sử trong dict.

Chưa có kết quả chạy ba replica thực tế. `docker-compose.yml` hiện map cố
định `8000:8000`, nên scale ba agent sẽ xung đột cổng host. Cần điều chỉnh
port mapping và thêm load balancer trước khi kiểm chứng. `REDIS_URL=fake://`
không thay thế Redis dùng chung giữa các process.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Chưa có log lỗi deploy thực tế được lưu trong tài liệu để xác nhận một sự cố.
`DEPLOYMENT.md` hiện ghi đã gửi deployment lên Railway và đang chờ xác minh
HTTP, nên chưa đủ dữ liệu để khẳng định nguyên nhân hoặc cách sửa một lỗi.

**Cần bổ sung từ lần deploy thực tế:**

- Thông báo lỗi nguyên văn và bước xảy ra lỗi: build, startup hoặc gọi API.
- Bằng chứng tìm nguyên nhân từ build log, runtime log hoặc kết quả gọi
  `/health` và `/ready`.
- Thay đổi đã thực hiện để sửa, cùng kết quả kiểm tra sau khi deploy lại.

Ví dụ để đối chiếu, không phải lỗi đã quan sát: nếu `/health` trả 200 nhưng
`/ready` trả 503 với `redis: false`, cần kiểm tra Redis có hoạt động và
`REDIS_URL` có trỏ đúng service hay không. Chỉ ghi đó là sự cố thực tế khi
có log hoặc response xác nhận; không đưa mật khẩu hay API key vào bài.
