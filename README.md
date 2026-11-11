# Homework Session 04 - Docker Compose & Microservices Networking (StoreX)

Kho lưu trữ bài tập thực hành **Session 04 - DevOps** cho dự án StoreX, bao gồm các kỹ thuật Docker Compose nâng cao, cơ chế mạng, phân giải DNS nội bộ, Healthcheck, quản lý Volumes và Reverse Proxy với Nginx.

---

## Danh Mục Bài Tập

| Bài tập | Nội dung nghiệp vụ | Thư mục bài làm | File nộp bài chính | Trạng thái |
| :--- | :--- | :--- | :--- | :---: |
| **Exercise 01** | Khắc phục lỗi `Connection refused` bằng Docker Compose Embedded DNS | [`SS_04/exercise_01/`](SS_04/exercise_01/) hoặc [`ex_01/`](ex_01/) | `docker-compose.yml`, `README.md` | Hoàn thành |
| **Exercise 02** | Cơ chế Healthcheck (`pg_isready`, `redis-cli ping`) & Data Persistence với Docker Volumes | [`SS_04/exercise_02/`](SS_04/exercise_02/) hoặc [`ex_02/`](ex_02/) | `docker-compose.yml`, `README.md` | Hoàn thành |
| **Exercise 03** | Bảo mật biến môi trường `.env.example` & Cách ly mạng với Custom Network (`storex-net`) | [`SS_04/exercise_03/`](SS_04/exercise_03/) hoặc [`ex_03/`](ex_03/) | `docker-compose.yml`, `.env.example`, `README.md` | Hoàn thành |
| **Exercise 04** | Xử lý lỗi `502 Bad Gateway` với Docker Internal DNS & Nginx Reverse Proxy routing | [`SS_04/exercise_04/`](SS_04/exercise_04/) hoặc [`ex_04/`](ex_04/) | `nginx.conf`, `docker-compose.yml`, `README.md` | Hoàn thành |

---

## Chi tiết từng bài tập

### 1. Exercise 01: Docker Compose Network & Embedded DNS
* **Vấn đề:** Backend gọi `jdbc:postgresql://localhost:5432/storex` bị `Connection refused` do `localhost` trỏ về chính container backend.
* **Khắc phục:** Thay `localhost` bằng DNS Service Name `postgres` (`jdbc:postgresql://postgres:5432/storex`). Bổ sung `depends_on: - postgres` và volume.

### 2. Exercise 02: Healthcheck & Volume Persistence
* **Vấn đề:** `depends_on` mặc định không chờ DB/Redis sẵn sàng; dữ liệu bị mất khi chạy `down` rồi `up`.
* **Khắc phục:**
  - Thêm `healthcheck` cho PostgreSQL (`pg_isready -U admin -d storex_db`) và Redis (`redis-cli ping`).
  - Thiết lập `depends_on` với `condition: service_healthy`.
  - Khai báo Named Volume `postgres_data:/var/lib/postgresql/data` để lưu trữ dữ liệu bền vững.

### 3. Exercise 03: Environment Isolation & Custom Network
* **Vấn đề:** Mật khẩu DB bị hardcode trong compose; port 5432 bị mở ra ngoài Host; dùng default bridge network.
* **Khắc phục:**
  - Tách mật khẩu vào `.env`, cung cấp template `.env.example` để push lên Git, bỏ qua `.env` trong `.gitignore`.
  - Đóng cổng 5432 phía Host (xóa `ports: - "5432:5432"` khỏi postgres).
  - Khai báo Custom Network `storex-net` (driver: bridge) để cô lập các container của StoreX.

### 4. Exercise 04: Nginx Reverse Proxy & Internal Routing
* **Vấn đề:** Nginx cấu hình `proxy_pass http://localhost:8081/;` dẫn tới `502 Bad Gateway` vì Nginx không kết nối được upstream.
* **Khắc phục:**
  - Thay `localhost` bằng DNS service name: `http://user_service:8081/` và `http://product_service:8082/`.
  - Thêm các HTTP headers chuẩn (`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`).
  - Phân tích cơ chế Rewrite Path với Trailing Slash `/` trong Nginx.
