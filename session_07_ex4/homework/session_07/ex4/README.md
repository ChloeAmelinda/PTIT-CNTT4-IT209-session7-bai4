# Bài 4 - Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot

## Thông tin học viên

- **Họ tên:** Đoàn Trung Nguyên
- **Mã lớp:** HN-K24-CNTT4
- **Bài:** Session 07 - Bài 4
- **Backend Spring Boot:** `127.0.0.1:8082`
- **Nginx:** Port `80`

---

## 1. Mục tiêu

Cấu hình Nginx làm Reverse Proxy:

- Truy cập `/` → Nginx phục vụ trang tĩnh tại `/var/www/html/`.
- Truy cập `/api/` → Nginx chuyển request tới Spring Boot tại `http://127.0.0.1:8082/`.

Kiến trúc:

```text
Client
  |
  | HTTP :80
  v
Nginx
  |
  +-- /        --> /var/www/html/index.html
  |
  +-- /api/**  --> Spring Boot :8082
```

---

## 2. Tạo trang tĩnh

Tạo thư mục:

```bash
sudo mkdir -p /var/www/html
```

Tạo file:

```bash
sudo nano /var/www/html/index.html
```

Nội dung mẫu đã được nộp kèm trong file `index.html`.

Có thể sao chép file bằng:

```bash
sudo cp index.html /var/www/html/index.html
```

Kiểm tra:

```bash
ls -l /var/www/html/
```

---

## 3. Tạo cấu hình Nginx

Tạo file cấu hình:

```bash
sudo nano /etc/nginx/sites-available/spring-proxy.conf
```

Nội dung:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8082/;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Hoặc chép file cấu hình đi kèm bài nộp:

```bash
sudo cp spring-proxy.conf /etc/nginx/sites-available/spring-proxy.conf
```

---

## 4. Kích hoạt Server Block

Tạo symbolic link:

```bash
sudo ln -s /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/spring-proxy.conf
```

Nếu link đã tồn tại:

```bash
sudo rm -f /etc/nginx/sites-enabled/spring-proxy.conf

sudo ln -s /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/spring-proxy.conf
```

Để tránh Server Block mặc định chiếm cổng 80, có thể tắt cấu hình mặc định:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

---

## 5. Kiểm tra cấu hình Nginx

Chạy:

```bash
sudo nginx -t
```

### Kết quả mong đợi

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Chỉ reload sau khi `nginx -t` thành công:

```bash
sudo systemctl reload nginx
```

Kiểm tra trạng thái:

```bash
sudo systemctl status nginx --no-pager
```

---

## 6. Kiểm tra trang Static

Chạy:

```bash
curl -I http://localhost/
```

### Kết quả mong đợi

```text
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
```

Kiểm tra nội dung đầy đủ:

```bash
curl http://localhost/
```

Trang phải chứa:

```text
Họ tên: Đoàn Trung Nguyên
Mã lớp: HN-K24-CNTT4
```

---

## 7. Kiểm tra Spring Boot Backend

Trước tiên kiểm tra Spring Boot có đang chạy tại port `8082`:

```bash
sudo ss -tlnp | grep :8082
```

Có thể kiểm tra trực tiếp backend:

```bash
curl -i http://127.0.0.1:8082/health
```

hoặc nếu ứng dụng sử dụng endpoint:

```text
/api/health
```

thì kiểm tra phù hợp với Controller thực tế.

---

## 8. Kiểm tra Reverse Proxy

Theo cấu hình:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8082/;
}
```

Request:

```text
/api/health
```

sẽ được Nginx chuyển thành:

```text
http://127.0.0.1:8082/health
```

Kiểm tra:

```bash
curl -i http://localhost/api/health
```

### Kết quả mong đợi

```text
HTTP/1.1 200 OK
```

và nhận được response từ ứng dụng Spring Boot.

Không được xuất hiện:

```text
502 Bad Gateway
```

---

## 9. Lưu ý quan trọng về `proxy_pass`

Cấu hình bài này sử dụng:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8082/;
}
```

Do `proxy_pass` có dấu `/` ở cuối nên:

```text
http://localhost/api/health
```

được chuyển thành:

```text
http://127.0.0.1:8082/health
```

Nếu Spring Boot của bạn thực tế khai báo Controller là:

```text
/api/health
```

và muốn giữ nguyên `/api/`, hãy đổi thành:

```nginx
proxy_pass http://127.0.0.1:8082;
```

Khi đó:

```text
http://localhost/api/health
```

sẽ được chuyển thành:

```text
http://127.0.0.1:8082/api/health
```

Trong bài nộp này giữ cấu hình đúng theo đề yêu cầu:

```nginx
proxy_pass http://127.0.0.1:8082/;
```

---

## 10. Các HTTP Header chuyển tiếp

Cấu hình:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

Ý nghĩa:

- `Host`: gửi hostname ban đầu tới backend.
- `X-Real-IP`: gửi địa chỉ IP thực của client.
- `X-Forwarded-For`: lưu chuỗi IP khi request đi qua proxy.
- `X-Forwarded-Proto`: cho backend biết request ban đầu sử dụng HTTP hay HTTPS.

---

## 11. Các lệnh kiểm tra tổng hợp

```bash
sudo nginx -t

sudo systemctl reload nginx

sudo systemctl status nginx --no-pager

curl -I http://localhost/

curl http://localhost/

sudo ss -tlnp | grep :8082

curl -i http://localhost/api/health
```

---

## 12. Minh chứng cần chụp

Nên chụp màn hình các kết quả sau để nộp:

1. Nội dung file `/etc/nginx/sites-available/spring-proxy.conf`.
2. Lệnh:

```bash
sudo nginx -t
```

hiển thị:

```text
syntax is ok
test is successful
```

3. Lệnh:

```bash
curl -I http://localhost/
```

trả về HTTP `200`.

4. Lệnh:

```bash
curl http://localhost/
```

hiển thị thông tin học viên.

5. Lệnh:

```bash
curl -i http://localhost/api/health
```

trả response từ Spring Boot và không bị `502 Bad Gateway`.

6. Kiểm tra backend đang lắng nghe port `8082`:

```bash
sudo ss -tlnp | grep :8082
```

---

## 13. Cấu trúc bài nộp

```text
homework/
└── session_07/
    └── ex4/
        ├── README.md
        ├── index.html
        └── spring-proxy.conf
```

---

## 14. Đẩy bài lên GitHub

Từ thư mục repository:

```bash
git add homework/session_07/ex4/
git commit -m "feat: configure nginx reverse proxy for spring boot"
git push
```

Kiểm tra trên GitHub phải có:

```text
homework/session_07/ex4/README.md
homework/session_07/ex4/index.html
homework/session_07/ex4/spring-proxy.conf
```

---

## 15. Kết luận

Đã cấu hình Nginx Server Block để:

- phục vụ trang tĩnh tại `/`;
- chuyển tiếp request `/api/` tới Spring Boot chạy ở port `8082`;
- chuyển tiếp thông tin client qua các HTTP header;
- kiểm tra cú pháp cấu hình bằng `nginx -t`;
- reload Nginx sau khi cấu hình hợp lệ;
- kiểm tra truy cập bằng `curl`.

Mô hình cuối cùng:

```text
               +-----------------------+
Client :80 --->|        Nginx          |
               |                       |
               | /       Static HTML   |
               | /api/   Reverse Proxy |
               +-----------+-----------+
                           |
                           v
                  Spring Boot :8082
```
