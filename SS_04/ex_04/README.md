# Báo Cáo Thực Hành Exercise 04 - Docker Compose Internal DNS & Nginx Reverse Proxy

## 1. Thông tin bài tập
* **Mục tiêu:** Phân tích và khắc phục lỗi `502 Bad Gateway` trong kiến trúc Microservices thông qua cơ chế phân giải tên miền nội bộ (Embedded DNS) của Docker Compose kết hợp định tuyến Reverse Proxy bằng Nginx.
* **Kiến trúc:** 
  - `nginx_proxy` (Port 80): Đóng vai trò API Gateway / Reverse Proxy nhận toàn bộ request từ người dùng.
  - `user_service` (Port nội bộ 8081): Xử lý các request liên quan đến người dùng (`/users/*`).
  - `product_service` (Port nội bộ 8082): Xử lý các request liên quan đến sản phẩm (`/products/*`).

---

## 2. Phân tích nguyên nhân Nginx trả về lỗi `502 Bad Gateway`

### 2.1. Bản chất của lỗi `502 Bad Gateway`
- Mã trạng thái HTTP `502 Bad Gateway` biểu thị rằng Nginx (hoạt động với tư cách là Proxy Server / Gateway) đã nhận một phản hồi không hợp lệ hoặc **hoàn toàn không thể thiết lập kết nối** tới máy chủ đích phía sau (Upstream Server).

### 2.2. Nguyên nhân cụ thể trong cấu hình ban đầu
Trong file `nginx.conf` ban đầu:
```nginx
location /users/ {
    proxy_pass http://localhost:8081/;
}
```
1. **Sự nhầm lẫn về ngữ cảnh `localhost`:**
   - Tiến trình Nginx đang thực thi bên trong một container độc lập (`nginx_proxy`).
   - Do đó, `localhost` (hay `127.0.0.1`) trong file cấu hình Nginx sẽ trỏ thẳng vào **chính mạng loopback của container Nginx**, chứ **không phải** máy host và **không phải** container `user_service`.
2. **Từ chối kết nối (Connection Refused):**
   - Bên trong container `nginx_proxy` chỉ có tiến trình Nginx lắng nghe ở cổng 80, hoàn toàn không có tiến trình nào lắng nghe ở cổng 8081.
   - Khi request gửi tới `/users/`, Nginx cố gắng mở socket kết nối tới `127.0.0.1:8081` và bị từ chối ngay lập tức:
     ```text
     [error] 29#29: *1 connect() failed (111: Connection refused) while connecting to upstream, client: 172.x.x.x, server: localhost, request: "GET /users/ HTTP/1.1", upstream: "http://127.0.0.1:8081/"
     ```
   - Vì không thể kết nối tới Upstream, Nginx buộc phải trả mã lỗi `502 Bad Gateway` về cho Client.

---

## 3. Nội dung cấu hình các file đã sửa hoàn chỉnh

### 3.1. File `nginx.conf` đã sửa
File cấu hình tại: [nginx.conf](./nginx.conf)

```nginx
server {
    listen 80;
    server_name localhost;

    # ==========================================================================
    # Định tuyến tới User Service (chạy cổng nội bộ 8081)
    # ==========================================================================
    location /users/ {
        # KHẮC PHỤC LỖI 502: Dùng DNS Service Name 'user_service'
        # Dấu gạch chéo '/' ở cuối URI giúp tự động rewrite/strip tiền tố '/users/'
        proxy_pass http://user_service:8081/;

        # Chuyển tiếp các HTTP Headers chuẩn
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }

    # ==========================================================================
    # Định tuyến tới Product Service (chạy cổng nội bộ 8082)
    # ==========================================================================
    location /products/ {
        # KHẮC PHỤC LỖI 502: Dùng DNS Service Name 'product_service'
        proxy_pass http://product_service:8082/;

        # Chuyển tiếp các HTTP Headers chuẩn
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }

    # Endpoint kiểm tra trạng thái Nginx
    location /health {
        return 200 '{"status":"UP","message":"Nginx Reverse Proxy is running"}';
        add_header Content-Type application/json;
    }
}
```

### 3.2. File `docker-compose.yml` đã sửa
File cấu hình tại: [docker-compose.yml](./docker-compose.yml)

```yaml
version: '3.8'

services:
  nginx_proxy:
    image: nginx:alpine
    container_name: storex-nginx-proxy
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - user_service
      - product_service
    networks:
      - storex-network
    restart: always

  user_service:
    image: storex-user:latest
    container_name: storex-user-service
    expose:
      - "8081"
    networks:
      - storex-network
    restart: on-failure

  product_service:
    image: storex-product:latest
    container_name: storex-product-service
    expose:
      - "8082"
    networks:
      - storex-network
    restart: on-failure

networks:
  storex-network:
    driver: bridge
```

---

## 4. Giải thích cơ chế Rewrite Path của Nginx (`proxy_pass` với Trailing Slash)

Trong chỉ thị `proxy_pass`:
* **Trường hợp 1 (Có dấu gạch chéo `/` ở cuối): `proxy_pass http://user_service:8081/;`**
  - Nginx sẽ thực hiện cơ chế **URI Normalization**: Phần đường dẫn khớp với `location /users/` sẽ bị **cắt bỏ (strip)**, chỉ giữ lại phần đuôi đằng sau để gửi tới backend.
  - *Ví dụ:* Client gọi `http://localhost/users/api/info` $\rightarrow$ Nginx chuyển tiếp tới `http://user_service:8081/api/info`.
  - Phù hợp khi backend thiết kế API ở root path (`/api/info`).

* **Trường hợp 2 (Không có dấu gạch chéo `/` ở cuối): `proxy_pass http://user_service:8081;`**
  - Nginx sẽ giữ nguyên toàn bộ URI gốc của client và chuyển tiếp nguyên vẹn tới backend.
  - *Ví dụ:* Client gọi `http://localhost/users/api/info` $\rightarrow$ Nginx chuyển tiếp tới `http://user_service:8081/users/api/info`.
  - Phù hợp khi backend đã cấu hình sẵn tiền tố `/users/` trong Controller (ví dụ: `@RequestMapping("/users")`).

---

## 5. Hướng dẫn kiểm thử và xác minh

1. **Khởi chạy hệ thống:**
   ```bash
   docker compose up -d
   ```
2. **Kiểm tra trạng thái các container:**
   ```bash
   docker compose ps
   ```
3. **Kiểm tra log của Nginx:**
   ```bash
   docker compose logs -f nginx_proxy
   ```
4. **Kiểm thử routing qua Nginx:**
   ```bash
   # Test User Service qua Proxy
   curl -i http://localhost/users/

   # Test Product Service qua Proxy
   curl -i http://localhost/products/

   # Test Health Endpoint của Nginx
   curl -i http://localhost/health
   ```
   *Kết quả mong đợi:* Nhận về mã phản hồi HTTP `200 OK` thay vì `502 Bad Gateway`.
