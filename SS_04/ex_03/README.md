# Báo Cáo Thực Hành Exercise 03 - Bảo Mật Biến Môi Trường & Custom Network

## 1. Thông tin bài tập
* **Mục tiêu:**
  - Tách các thông tin cấu hình nhạy cảm (Credentials/Passwords) ra khỏi file `docker-compose.yml` thông qua file biến môi trường `.env`.
  - Thiết kế hệ thống mạng cô lập bằng **Custom Network (`storex-net`)**.
  - Áp dụng nguyên tắc đóng cổng bảo mật: Loại bỏ việc mở port Database (5432) trực tiếp ra ngoài máy Host.

---

## 2. Phân tích 3 lỗ hổng bảo mật ban đầu

| STT | Lỗ hổng trong cấu hình cũ | Hậu quả tiềm ẩn | Giải pháp khắc phục |
| :---: | :--- | :--- | :--- |
| **1** | **Hardcode mật khẩu:** `DB_PASS=supersecret_dont_share` nằm trực tiếp trong compose file | Khi mã nguồn được push lên GitHub công khai, mật khẩu bị lộ hoàn toàn. | Đưa vào file `.env`, dùng file `.env.example` làm template trên Git. Thêm `.env` vào `.gitignore`. |
| **2** | **Mở cổng Database ra Host:** `ports: - "5432:5432"` | Bất kỳ ai có địa chỉ IP của máy chủ đều có thể quét port 5432 và tấn công Brute-force vào PostgreSQL. | **Xóa bỏ khai báo `ports` của PostgreSQL**. Backend vẫn gọi được DB qua mạng nội bộ Docker. |
| **3** | **Dùng mạng Default Bridge của Docker** | Các container không liên quan chạy trên cùng máy host nếu vào default network đều có thể giao tiếp với Database. | Tạo mạng riêng **Custom Network (`storex-net`)** để chỉ có `backend` và `postgres` mới nhìn thấy nhau. |

---

## 3. Nội dung cấu hình các file

### 3.1. File `docker-compose.yml` đã được sửa hoàn chỉnh
File cấu hình tại: [docker-compose.yml](./docker-compose.yml)

```yaml
version: '3.8'

services:
  backend:
    image: storex-backend:v3
    ports:
      - "${BACKEND_PORT:-8080}:8080"
    environment:
      - DB_HOST=postgres
      - DB_USER=${DB_USER}
      - DB_PASS=${DB_PASS}
    networks:
      - storex-net
    depends_on:
      - postgres
    restart: on-failure

  postgres:
    image: postgres:13
    # BẢO MẬT: Đã xóa mục 'ports: - "5432:5432"' để ngăn truy cập trái phép từ máy host hoặc Internet
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASS}
      - POSTGRES_DB=${DB_NAME:-storex_db}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - storex-net
    restart: always

networks:
  # Khai báo Custom Network riêng cho dự án StoreX
  storex-net:
    driver: bridge

volumes:
  postgres_data:
    driver: local
```

### 3.2. File `.env.example` (Template để nộp lên Git)
File cấu hình tại: [.env.example](./.env.example)

```env
# ==============================================================================
# STOREX CONFIGURATION TEMPLATE (.env.example)
# Hướng dẫn: Tạo bản sao thành file .env và điền thông tin thực tế:
# cp .env.example .env
# ==============================================================================

# Cấu hình cổng dịch vụ Backend
BACKEND_PORT=8080

# Cấu hình Database PostgreSQL
DB_NAME=storex_db
DB_USER=your_db_username_here
DB_PASS=your_strong_password_here
```

### 3.3. File `.gitignore` (Ngăn chặn lộ file `.env` lên Git)
File cấu hình tại: [.gitignore](./.gitignore)

```gitignore
.env
```

---

## 4. Hướng dẫn kiểm thử và xác minh

### Bước 1: Kiểm tra việc inject biến môi trường
Chạy lệnh kiểm tra cấu hình được render từ Docker Compose:
```bash
docker compose config
```
*Kết quả:* Docker Compose tự động nạp các giá trị từ `.env` vào các placeholder `${DB_USER}`, `${DB_PASS}`, v.v.

### Bước 2: Khởi động hệ thống
```bash
docker compose up -d
```

### Bước 3: Kiểm tra tính cô lập của mạng (Custom Network)
```bash
# Kiểm tra sự tồn tại của custom network
docker network ls | grep storex-net

# Xem các container đang nằm trong mạng storex-net
docker network inspect storex_storex-net
```
*Kết quả mong đợi:* Chỉ có 2 container `backend` và `postgres` được gán vào mạng này.

### Bước 4: Xác nhận cổng 5432 không bị lộ ra ngoài Host
```bash
docker compose ps
```
*Kết quả:* Cột `PORTS` của `postgres` chỉ hiển thị `5432/tcp` (nội bộ container), hoàn toàn không có `0.0.0.0:5432->5432/tcp`. Máy tính bên ngoài không thể can thiệp trực tiếp vào database.
