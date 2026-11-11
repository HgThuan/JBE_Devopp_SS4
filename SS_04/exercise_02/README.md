# Báo Cáo Thực Hành Exercise 02 - Healthcheck & Data Persistence trong Docker Compose

## 1. Thông tin bài tập
* **Mục tiêu:**
  - Thiết lập **Volumes** để bảo toàn dữ liệu PostgreSQL (Data Persistence).
  - Sử dụng **Healthcheck** kết hợp với **`depends_on: condition: service_healthy`** để kiểm soát thứ tự khởi động của các service, đảm bảo Backend chỉ chạy khi Database và Redis đã thực sự sẵn sàng nhận kết nối.

---

## 2. File `docker-compose.yml` hoàn chỉnh

File cấu hình tại: [docker-compose.yml](./docker-compose.yml)

```yaml
version: '3.8'

services:
  backend:
    image: storex-backend:v2
    ports:
      - "8080:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/storex_db
      - SPRING_DATASOURCE_USERNAME=admin
      - SPRING_DATASOURCE_PASSWORD=admin
      - SPRING_REDIS_HOST=redis
      - SPRING_REDIS_PORT=6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: on-failure

  postgres:
    image: postgres:13
    environment:
      - POSTGRES_DB=storex_db
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=admin
    volumes:
      # Lưu trữ bền vững dữ liệu cơ sở dữ liệu trên Docker Volume
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      # pg_isready kiểm tra server PostgreSQL đã sẵn sàng nhận kết nối socket/TCP
      test: ["CMD-SHELL", "pg_isready -U admin -d storex_db"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
    restart: always

  redis:
    image: redis:alpine
    healthcheck:
      # redis-cli ping kiểm tra Redis đã sẵn sàng phản hồi PONG
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s
    restart: always

volumes:
  postgres_data:
    driver: local
```

---

## 3. Giải thích ngắn gọn cơ chế Healthcheck & Thứ tự khởi động

### 3.1. Hạn chế của `depends_on` mặc định
- Mặc định, `depends_on` dạng danh sách (`- postgres`, `- redis`) chỉ theo dõi trạng thái **Container Started** (tiến trình chính đã được kích hoạt trong container).
- Tuy nhiên, PostgreSQL và Redis cần từ vài giây để nạp cấu hình, khởi tạo socket và mở cổng lắng nghe.
- Hậu quả: Backend khởi động gần như cùng lúc và cố kết nối ngay dẫn đến crash (`Connection refused`).

### 3.2. Cơ chế `healthcheck` được áp dụng
Healthcheck định kỳ chạy một lệnh kiểm tra bên trong container để xác định trạng thái sức khỏe:

1. **Với PostgreSQL:**
   - Lệnh kiểm tra: `pg_isready -U admin -d storex_db`
   - Công cụ `pg_isready` có sẵn trong image PostgreSQL, gửi tín hiệu kiểm tra xem server đã mở port 5432 và sẵn sàng tiếp nhận truy vấn với database `storex_db` hay chưa.
2. **Với Redis:**
   - Lệnh kiểm tra: `redis-cli ping`
   - Gửi lệnh `PING` tới server Redis. Khi Redis đã sẵn sàng xử lý, nó trả về `PONG` (exit code `0` biểu thị healthy).

### 3.3. Ý nghĩa các tham số Healthcheck:
* `interval: 5s`: Cứ mỗi 5 giây chạy lệnh kiểm tra một lần.
* `timeout: 5s`: Thời gian tối đa chờ lệnh kiểm tra phản hồi.
* `retries: 5`: Nếu thất bại liên tiếp 5 lần thì container mới bị đánh dấu là `unhealthy`.
* `start_period: 5s - 10s`: Khoảng thời gian ân hạn ban đầu để dịch vụ boot, trong giai đoạn này các lần check thất bại không tính vào `retries`.

### 3.4. Phối hợp với `condition: service_healthy`
- Khi cấu hình:
  ```yaml
  depends_on:
    postgres:
      condition: service_healthy
    redis:
      condition: service_healthy
  ```
- Docker Compose sẽ giữ container `backend` ở trạng thái chờ (chưa khởi động) cho đến khi cả `postgres` và `redis` chuyển sang trạng thái **`healthy`**. Khi đó Backend mới boot, loại bỏ 100% lỗi crash do thiếu database/cache.

---

## 4. Cơ chế lưu trữ dữ liệu bền vững (Data Persistence với Volumes)

- **Vấn đề ban đầu:** Mỗi container có một lớp ghi (Writable Layer) tạm thời gắn liền với vòng đời của nó. Khi gõ `docker compose down`, container bị hủy dẫn đến toàn bộ dữ liệu database bên trong `/var/lib/postgresql/data` bị xóa sạch.
- **Giải pháp:** Khai báo **Named Volume** `postgres_data` và mount vào thư mục `/var/lib/postgresql/data`.
- **Hiệu quả:**
  - Dữ liệu vật lý được lưu trữ độc lập trên ổ đĩa của máy host (tại `/var/lib/docker/volumes/...`).
  - Khi thực hiện `docker compose down` rồi `docker compose up -d`, container mới được tạo ra sẽ gắn lại chính xác volume này, giúp toàn bộ bảng và dữ liệu vẫn còn nguyên vẹn.

---

## 5. Hướng dẫn kiểm thử thực tế

1. **Khởi động hệ thống:**
   ```bash
   docker compose up -d
   ```
2. **Quan sát trạng thái healthy:**
   ```bash
   docker compose ps
   ```
   *Kết quả mong đợi:* Cột `STATUS` của `postgres` và `redis` hiển thị `Up ... (healthy)`.
3. **Kiểm tra dữ liệu bền vững:**
   - Thêm một bảng hoặc dữ liệu mẫu vào PostgreSQL.
   - Chạy lệnh hạ hệ thống: `docker compose down` (không dùng cờ `-v`).
   - Bật lại hệ thống: `docker compose up -d`.
   - Kiểm tra lại dữ liệu $\rightarrow$ Dữ liệu vẫn tồn tại nguyên vẹn.
