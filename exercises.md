# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng giữ chỗ bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đào Kiều Thịnh Quang  Mã học viên: 2A202601014

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu deploy mà quên `API_TOKEN`, app sẽ dừng ngay và báo lỗi để tôi sửa sớm.
> Nếu để mặc định `"changeme"`, app vẫn chạy với một token yếu, dễ bị đoán và
> khiến endpoint `/chat` có nguy cơ bị người khác sử dụng trái phép.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng log thực tế tôi thu được khi gọi `/chat` là:

```json
{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T10:00:25.000302+00:00", "client_id": "sv01", "prompt_tokens": 3, "completion_tokens": 37, "usd_cost": 2.265e-05}
```

Với log JSON này, tôi có thể lọc và thống kê theo `client_id`, `severity` hoặc
`usd_cost`; đồng thời đưa dữ liệu vào hệ thống monitoring để tạo dashboard và
cảnh báo. `print("đã trả lời xong")` không có các trường dữ liệu có cấu trúc để
làm những việc đó.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 296 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build và đo trực tiếp hai image trên cùng máy. Bản multi-stage nhỏ hơn vì
> stage runtime dùng base image `slim` và chỉ nhận kết quả cài đặt từ builder,
> không mang theo base image đầy đủ, compiler, build tools và cache chỉ cần trong
> quá trình build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Nếu chỉ sửa `app/main.py`, các layer copy `requirements.txt` và cài dependency
> trước đó vẫn dùng cache; layer copy source và các layer phía sau phải tạo lại.
> Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi source đều làm mất cache
> của layer cài dependency và khiến `pip install` chạy lại không cần thiết.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu app có lỗ hổng thực thi code, kẻ tấn công nhận quyền của user đang chạy
> process. Nếu process chạy bằng root, mức độ nguy hiểm cao hơn và kẻ tấn công có
> thể kết hợp thêm lỗ hổng container runtime để tìm cách chiếm quyền cao trên host.
> `USER appuser` cắt chuỗi này ở bước thực thi code: code bị chiếm quyền chỉ chạy
> với quyền user thường trong container.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> Header `WWW-Authenticate: Bearer` cho client biết API yêu cầu xác thực bằng
> Bearer token theo chuẩn HTTP. Trả cùng một thông báo cho thiếu header, sai scheme
> và sai token giúp tránh tiết lộ cho kẻ tấn công phần nào của thông tin xác thực
> đã đúng, khiến việc dò token khó hơn.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Với `capacity=10`, client im lặng 10 phút vẫn chỉ tích tối đa 10 token nên gửi
> được 10 request liên tiếp trước khi nhận 429. Nếu bỏ `min(capacity, ...)`, tốc độ
> 10 token/phút trong 10 phút có thể tích thành khoảng 100 token, cho phép burst
> khoảng 100 request và làm mất ý nghĩa giới hạn sức chứa của xô.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức `$30/tháng`, sự cố có thể tiêu hết tối đa $30 và client phải chờ kỳ
> tháng tiếp theo mới tự có ngân sách lại. Với `$1/ngày`, thiệt hại của cùng sự cố
> bị giới hạn ở $1 trong ngày đó và quota tự phục hồi khi sang ngày UTC tiếp theo.
> Vì vậy hạn mức ngày thu hẹp đáng kể thiệt hại của một sự cố xảy ra từ 2 giờ sáng.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp hai probe và cho health check kiểm tra Redis, khi Redis mất kết nối cả
> ba container cùng trả 503. Orchestrator sẽ coi cả ba process đã chết, rút chúng
> khỏi cụm và restart đồng loạt, làm gián đoạn cả request đang xử lý và có thể tạo
> vòng lặp restart. Khi tách riêng, `/healthz` vẫn báo process sống để tránh restart
> oan, còn `/readyz` trả 503 để load balancer tạm ngừng gửi traffic cho đến khi
> Redis phục hồi.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi deploy lần đầu, log Railway báo `Invalid value for '--port': '$PORT' is not
> a valid integer`. Tôi đọc deployment log và thấy `startCommand` trong
> `railway.toml` được chạy không qua shell nên Uvicorn nhận nguyên chuỗi `$PORT`
> thay vì số cổng. Tôi bỏ `startCommand` override để Railway dùng `CMD` của
> Dockerfile, nơi lệnh chạy qua `sh -c` và đọc `${PORT:-8000}`, rồi deploy lại
> thành công.
