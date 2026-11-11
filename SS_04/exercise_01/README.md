# Báo Cáo Thực Hành Exercise 01 - Docker Compose Network (StoreX)

## 1. Thông tin bài tập
* **Dự án:** StoreX (Spring Boot Backend + PostgreSQL Database)
* **Mục tiêu:** Hiểu và áp dụng cơ chế mạng (Docker Network & Service Discovery) trong `docker-compose.yml` để kết nối nhiều container.

---

## 2. Xác định và giải thích nguyên nhân gây ra lỗi kết nối `Connection refused`

### 2.1. Phân tích lỗi thực tế
Trong cấu hình ban đầu của developer:
```yaml
environment:
  - SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/storex
```
Khi khởi chạy, ứng dụng Spring Boot bên trong container `backend` liên tục báo lỗi:
```text
org.postgresql.util.PSQLException: Connection to localhost:5432 refused.
Check that the hostname and port are correct and that the postmaster is accepting TCP/IP connections.
```

### 2.2. Giải thích bản chất dựa trên cơ chế mạng của Docker Compose
1. **Tính cô lập của Network Namespace (Container Isolation):**
   - Mỗi Docker container khi khởi chạy được cấp phát một **Network Namespace riêng biệt**.
   - Do đó, địa chỉ `localhost` (hoặc `127.0.0.1`) bên trong container `backend` chỉ đại diện cho **chính container backend đó** (Loopback Interface nội bộ của nó), hoàn toàn **không trỏ về máy host** và **không trỏ về container postgres**.
   - Do bên trong container `backend` không hề có tiến trình PostgreSQL nào lắng nghe ở cổng 5432, hệ điều hành trả về lỗi `Connection refused` (Từ chối kết nối).

2. **Cơ chế mạng mặc định của Docker Compose (Default Bridge Network):**
   - Khi chạy `docker compose up`, Docker Compose tự động tạo một mạng bridge riêng (ví dụ: `storex_default`) và gắn tất cả các service (`backend`, `postgres`) vào chung mạng này.
   - Mỗi container sẽ được cấp một IP nội bộ riêng trong dải mạng đó.

3. **Cơ chế phân giải tên miền nội bộ (Docker Embedded DNS - Service Discovery):**
   - Docker tích hợp sẵn một máy chủ DNS nội bộ tại địa chỉ `127.0.0.11`.
   - DNS server này tự động đăng ký và ánh xạ **Tên Service (Service Name)** được khai báo trong `docker-compose.yml` thành **Địa chỉ IP nội bộ** của container tương ứng.
   - Do đó, để container `backend` giao tiếp được với container `postgres`, ta phải sử dụng tên service là `postgres` làm hostname thay vì `localhost`.

---

## 3. File `docker-compose.yml` đã được sửa hoàn chỉnh

File cấu hình tại: [docker-compose.yml](./docker-compose.yml)

```yaml
version: '3.8'

services:
  backend:
    image: storex-backend:latest
    ports:
      - "8080:8080"
    environment:
      # KHẮC PHỤC LỖI: Đổi 'localhost' thành tên service 'postgres'
      # Docker Embedded DNS sẽ tự động phân giải hostname 'postgres' thành IP nội bộ của database container
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/storex
      - SPRING_DATASOURCE_USERNAME=postgres
      - SPRING_DATASOURCE_PASSWORD=secret
    depends_on:
      - postgres
    restart: on-failure

  postgres:
    image: postgres:13
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=storex
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=secret
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: always

volumes:
  postgres_data:
```

### Các điểm cải tiến quan trọng:
1. **`SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/storex`**: Chuyển hostname sang `postgres` để kích hoạt cơ chế Service Discovery.
2. **`depends_on: - postgres`**: Đảm bảo container cơ sở dữ liệu được khởi động trước khi container backend bắt đầu chạy.
3. **`volumes: - postgres_data:/var/lib/postgresql/data`**: Đảm bảo dữ liệu của database không bị mất khi container dừng hoặc bị xóa (Data Persistence).

---

## 4. Các bước kiểm thử và xác minh kết quả

### Bước 1: Khởi động hệ thống
```bash
docker compose up -d
```

### Bước 2: Kiểm tra trạng thái các container
```bash
docker compose ps
```
*Kết quả mong đợi:* Cả hai service `backend` và `postgres` đều hiển thị trạng thái `Up` hoặc `running`.

### Bước 3: Kiểm tra log kết nối thành công
```bash
# Kiểm tra log của database
docker compose logs postgres

# Kiểm tra log của backend
docker compose logs backend
```
*Kết quả mong đợi trong log backend:*
```text
HikariPool-1 - Starting...
HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@...
HikariPool-1 - Start completed.
Started StoreXApplication in ... seconds
```
*(Không còn lỗi `Connection refused`, HikariCP đã tạo kết nối thành công tới `postgres:5432`).*

---

## 5. Hướng dẫn nộp bài vào Repository
* Thư mục bài nộp: `ex_01` hoặc `exercise_01`
* File nộp gồm:
  - `docker-compose.yml` (đã sửa)
  - `README.md` (giải thích chi tiết)
  - Ảnh chụp màn hình kết quả chạy log
