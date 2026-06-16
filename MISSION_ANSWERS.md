# Day 12 Lab - Mission Answers

**Họ và tên:** Nguyễn Đức Tâm  
**Mã sinh viên:** 2A202600946  
**Ngày thực hiện:** 16/06/2026  

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in `develop/app.py`
Qua phân tích file [app.py](file:///c:/Users/VTM/Documents/GitHub/2A202600946-NguyenDucTam-Day12/01-localhost-vs-production/develop/app.py), tôi phát hiện ra 5 vấn đề anti-pattern chính:
1. **Hardcoded Secrets:** API key (`OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"`) và Database URL (`DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"`) bị ghi cứng trực tiếp vào mã nguồn. Nếu đẩy lên các hệ thống quản lý mã nguồn công khai như GitHub, các thông tin bảo mật này sẽ lập tức bị rò rỉ.
2. **Thiếu Config Management:** Các cài đặt cấu hình như `DEBUG = True` và `MAX_TOKENS = 500` bị định nghĩa cứng dưới dạng hằng số toàn cục mà không đọc từ các biến môi trường (Environment Variables).
3. **Log không có cấu trúc và lộ thông tin nhạy cảm:** Ứng dụng sử dụng hàm `print()` thay vì thư viện logging chuyên dụng. Điều này làm log không có timestamp, log level và đặc biệt nguy hiểm khi log trực tiếp thông tin nhạy cảm ra ngoài: `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`.
4. **Không có Health Check Endpoint:** Ứng dụng thiếu các endpoint phục vụ cho việc kiểm tra trạng thái hoạt động như `/health` và `/ready`. Khi triển khai lên các cloud platform, hệ thống sẽ không tự phát hiện được nếu tiến trình bị treo hoặc lỗi để thực hiện restart tự động.
5. **Cấu hình Port và Host cứng:** Máy chủ Uvicorn được chỉ định chạy cố định trên `host="localhost"` và `port=8000`. Điều này khiến container không thể nhận kết nối từ môi trường bên ngoài (bắt buộc phải bind `0.0.0.0`) và không thể tự động thay đổi port theo yêu cầu của nhà cung cấp dịch vụ cloud (thường inject qua biến môi trường `PORT`).

### Exercise 1.3: Comparison table

| Feature | Basic (`develop/app.py`) | Advanced (`production/app.py`) | Tại sao quan trọng? |
| :--- | :--- | :--- | :--- |
| **Config** | Hardcoded trực tiếp trong file code. | Đọc tập trung qua thư viện `pydantic-settings` và `.env` file. | Giúp phân tách cấu hình ra khỏi mã nguồn, dễ dàng thay đổi cài đặt giữa các môi trường (Dev, Staging, Prod) mà không cần chỉnh sửa code. |
| **Secrets** | Ghi cứng (`sk-hardcoded-...`). | Sử dụng `os.getenv("OPENAI_API_KEY")`. | Tránh việc vô tình commit các token bảo mật quan trọng lên GitHub và bảo mật thông tin tối đa. |
| **Port & Host** | Cố định `localhost:8000`. | Đọc động qua `PORT` và `HOST` (mặc định `0.0.0.0`). | Giúp container lắng nghe kết nối từ bên ngoài và tương thích với cơ chế cấp phát port động của các cloud platform. |
| **Health Check** | Không hỗ trợ. | Định nghĩa 2 endpoint `/health` và `/ready` rõ ràng. | Giúp các công cụ giám sát (Monitoring) và điều phối (Kubernetes, Cloud Run) biết khi nào tiến trình bị lỗi để restart hoặc ngắt traffic. |
| **Logging** | Sử dụng hàm `print()` thông thường. | Sử dụng structured JSON logging. | Dễ dàng thu thập và phân tích log tự động qua các log aggregator như Loki, Datadog hay CloudWatch. |
| **Shutdown** | Dừng tiến trình đột ngột ngay lập tức. | Sử dụng `signal.signal(signal.SIGTERM, ...)` xử lý graceful shutdown. | Giúp hoàn thành nốt các request đang xử lý dở dang (in-flight requests), đóng kết nối cơ sở dữ liệu an toàn trước khi dừng hẳn. |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. **Base image là gì?**  
   Base image của Dockerfile cơ bản là `python:3.11` (Bản phân phối Python đầy đủ, dung lượng lớn ~1 GB). Base image của Dockerfile nâng cao là `python:3.11-slim` (Bản thu gọn, chỉ có các thư viện chạy Python tối giản, dung lượng ~150 MB).
2. **Working directory là gì?**  
   Working directory trong container được cấu hình là `/app` thông qua câu lệnh `WORKDIR /app`. Đây là thư mục gốc nơi toàn bộ mã nguồn của ứng dụng sẽ được lưu trữ và thực thi bên trong container.
3. **Tại sao COPY requirements.txt trước?**  
   Đây là kỹ thuật tối ưu cơ chế cache layer của Docker. Docker build chạy tuần tự từng câu lệnh và tạo ra các image layers. Bằng cách copy `requirements.txt` và chạy `pip install` trước khi copy toàn bộ code, Docker sẽ không phải cài đặt lại toàn bộ dependencies từ đầu mỗi khi bạn sửa code ở các file như `app.py`. Tiến trình build chỉ chạy lại phần cài đặt thư viện khi file `requirements.txt` có sự thay đổi.
4. **CMD vs ENTRYPOINT khác nhau thế nào?**  
   - `ENTRYPOINT`: Xác định file thực thi chính sẽ chạy khi container khởi động. Các tham số truyền vào lệnh chạy container sẽ được nối tiếp vào sau lệnh này. Rất khó để ghi đè trừ khi chỉ định rõ flag `--entrypoint`.
   - `CMD`: Xác định lệnh mặc định hoặc cung cấp tham số mặc định cho `ENTRYPOINT`. Lệnh trong `CMD` có thể dễ dàng bị ghi đè hoàn toàn bằng cách truyền một lệnh khác ở cuối câu lệnh `docker run`.

### Exercise 2.3: Image size comparison
- **Develop Image (Single-stage, `python:3.11`):** ~854 MB
- **Production Image (Multi-stage, `python:3.11-slim`):** ~162 MB
- **Chênh lệch:** Giảm khoảng **81%** dung lượng.
- **Lợi ích:** Image nhỏ giúp tối ưu hóa dung lượng lưu trữ trên Registry, giảm thời gian download/upload (pull/push) lên cloud, tăng tốc độ khởi động dịch vụ và giảm thiểu các rủi ro bảo mật do lược bỏ bớt các công cụ biên dịch không cần thiết ở môi trường runtime.

### Exercise 2.4: Architecture diagram details
Hệ thống chạy trên Docker Compose bao gồm các dịch vụ phối hợp như sau:
```
                    [ Port 80 / 443 ]
                            │
                            ▼
                    ┌───────────────┐
                    │ Nginx (Proxy) │ (Cân bằng tải / reverse proxy)
                    └───────┬───────┘
                            │
                    ┌───────┴───────┐
                    ▼               ▼
             ┌─────────────┐ ┌─────────────┐
             │   Agent 1   │ │   Agent 2   │ (FastAPI app, chạy stateless)
             └──────┬──────┘ └──────┬──────┘
                    │               │
                    └───────┬───────┘
                            ▼
                    ┌───────────────┐
                    │     Redis     │ (Lưu trữ session history & rate-limiting)
                    └───────────────┘
```
- **Nginx** lắng nghe ở port public `80`, nhận request từ client và thực hiện cân bằng tải (Round-robin) chuyển tiếp đến các instances của `agent` (port `8000`).
- Các instances của **Agent** liên lạc nội bộ với **Redis** (port `6379`) để đọc/ghi lịch sử hội thoại và kiểm tra giới hạn request. Toàn bộ giao tiếp nội bộ này diễn ra trong mạng bridge cô lập `internal`.

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- **Public URL:** `https://your-agent.up.railway.app` (Sinh viên thay bằng link thật sau khi deploy)
- **Các lệnh triển khai chính:**
  ```bash
  railway login
  railway init
  railway variables set PORT=8000 AGENT_API_KEY=my-secret-key
  railway up
  ```

### Exercise 3.2: Configuration files comparison (Railway vs Render)
- **`railway.toml`:** File cấu hình tập trung vào mô tả quá trình Build và Deploy ở mức dịch vụ đơn lẻ. Nó cấu hình builder (DOCKERFILE) và lệnh chạy `startCommand`, đồng thời quản lý thời gian timeout cho healthcheck trực tiếp trên platform Railway.
- **`render.yaml`:** Đây là file Blueprint triển khai theo dạng **Infrastructure as Code (IaC)**. Nó định nghĩa toàn bộ mô hình hạ tầng bao gồm định nghĩa dịch vụ Web Service, tên, vùng địa lý triển khai (region), gói dịch vụ (plan starter), đường dẫn healthcheck, và danh sách các biến môi trường đi kèm (có hỗ trợ tự động sinh giá trị ngẫu nhiên qua thuộc tính `generateValue: true` như `AGENT_API_KEY` hay `JWT_SECRET`).

---

## Part 4: API Security

### Exercise 4.4: Cost guard implementation
Logic kiểm tra và giới hạn chi phí sử dụng API được thực hiện tại [cost_guard.py](file:///c:/Users/VTM/Documents/GitHub/2A202600946-NguyenDucTam-Day12/06-lab-complete/app/main.py#L70-L84):
1. **Theo dõi chi tiêu theo ngày:** Chương trình duy trì biến toàn cục `_daily_cost` và kiểm tra xem ngày hiện tại có trùng với `_cost_reset_day` không. Nếu bước sang ngày mới, chi tiêu sẽ được tự động reset về `0.0`.
2. **Kiểm tra budget trước khi gọi LLM:** Mỗi khi nhận request mới, hệ thống ước tính token đầu vào, nếu tổng chi phí đã dùng cộng với chi phí ước tính vượt quá `daily_budget_usd` (mặc định `$5.0`), hệ thống sẽ trả về lỗi `503 Service Unavailable` kèm thông báo *"Daily budget exhausted"*.
3. **Tính toán và cập nhật chi phí sau khi hoàn tất:**
   - Giá token đầu vào (Input): `$0.00015 / 1000 tokens`
   - Giá token đầu ra (Output): `$0.0006 / 1000 tokens`
   - Sau khi nhận response từ LLM, chi phí thực tế sẽ được cộng dồn vào `_daily_cost`.

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health / Readiness endpoints details
Hai endpoint này được định nghĩa tại [main.py](file:///c:/Users/VTM/Documents/GitHub/2A202600946-NguyenDucTam-Day12/06-lab-complete/app/main.py#L230-L252):
- **`/health` (Liveness Check):** Trả về trạng thái `status: "ok"` kèm theo các thông tin hệ thống như phiên bản, môi trường chạy, thời gian uptime (`uptime_seconds`) và tổng số request đã xử lý. Nếu endpoint này gặp lỗi (không phản hồi hoặc trả về status code không phải 2xx), platform cloud sẽ hiểu là ứng dụng bị treo và tiến hành khởi động lại container.
- **`/ready` (Readiness Check):** Sử dụng biến cờ trạng thái `_is_ready` (được thiết lập là `True` sau khi hoàn tất startup logic và chuyển thành `False` ngay khi nhận được tín hiệu shutdown). Nếu endpoint này trả về lỗi `503`, Nginx hoặc Load Balancer của cloud platform sẽ tạm dừng phân phối request tới instance này để tránh gây lỗi cho người dùng.

### Exercise 5.3: Stateless transition description
- **Vấn đề của Stateful:** Khi chạy nhiều instances (ví dụ: `agent=3`), nếu lịch sử chat được lưu trong memory của từng container, một request tiếp theo của cùng một user bị chuyển tới container khác sẽ làm mất hoàn toàn ngữ cảnh hội thoại cũ.
- **Cách giải quyết (Stateless):** Chuyển toàn bộ dữ liệu lịch sử hội thoại (`session history`), thông tin lượt dùng (`rate limiting metrics`) và thông tin chi tiêu (`budget tracking`) ra khỏi bộ nhớ RAM của tiến trình FastAPI và lưu trữ tập trung vào một cơ sở dữ liệu dùng chung là **Redis**.
- **Kết quả:** Bất kỳ container nào cũng có thể xử lý request từ bất kỳ người dùng nào bằng cách truy vấn trạng thái từ Redis, cho phép tắt/mở hoặc scale ngang các instance một cách an toàn mà không làm ảnh hưởng tới trải nghiệm của người dùng.
