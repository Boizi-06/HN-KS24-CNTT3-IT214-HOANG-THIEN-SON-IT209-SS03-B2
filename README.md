# Báo Cáo Bài Tập 2: Cấu Hình Trang Lỗi Tùy Chỉnh (Custom Error Page 404)

- **Môn học:** IT209
- **Buổi học:** Session 03 - Server Security & Hardening
- **Bài tập:** Bài 2 - Cấu hình trang lỗi tùy chỉnh (Custom Error Page 404)
- **Học viên:** Hoang Thien Son (hoangthienson06@gmail.com)

---

## 1. Mục tiêu bài thực hành
- **Cải thiện trải nghiệm người dùng (UX):** Tùy biến trang thông báo lỗi `404 Not Found` với giao diện thẩm mỹ, thân thiện thay thế cho trang lỗi trắng đen mặc định của Nginx.
- **Bảo mật tệp tin tài nguyên tĩnh:** Ứng dụng chỉ thị **`internal`** của Nginx nhằm ngăn chặn người dùng từ Internet truy cập trực tiếp vào URL của tệp lỗi (`/404.html`), chỉ cho phép Nginx gọi tệp tin này thông qua cơ chế chuyển hướng nội bộ (internal redirect).
- **Quản lý Server Block chuẩn hóa:** Cấu hình document root độc lập tại `/var/www/my-web/html/` và kiểm soát vòng đời dịch vụ Nginx an toàn.

---

## 2. Thông số thiết lập dịch vụ

| Thành phần | Giá trị cấu hình | Ý nghĩa kỹ thuật |
| :--- | :--- | :--- |
| **Thư mục Web Root** | `/var/www/my-web/html/` | Thư mục chứa mã nguồn trang web & trang lỗi |
| **Tệp tin trang lỗi** | `/var/www/my-web/html/404.html` | Trang HTML hiển thị thông báo lỗi tùy biến |
| **File cấu hình Server Block** | `/etc/nginx/sites-available/my-web.conf` | Khai báo Virtual Host cho website |
| **Chỉ thị chuyển hướng lỗi** | `error_page 404 /404.html;` | Nginx tự động phục vụ file khi gặp mã lỗi 404 |
| **Chỉ thị bảo mật** | `location = /404.html { internal; }` | Chặn truy cập trực tiếp từ người dùng bên ngoài |

---

## 3. Các bước triển khai chi tiết

### Bước 1: Khởi tạo thư mục và tệp tin trang lỗi `404.html`
Tạo thư mục web root theo đúng yêu cầu:
```bash
sudo mkdir -p /var/www/my-web/html
```

Tạo file `404.html` với nội dung giao diện tùy biến thân thiện:
```bash
sudo nano /var/www/my-web/html/404.html
```

Phân quyền chuẩn cho thư mục web:
```bash
sudo chown -R www-data:www-data /var/www/my-web/html
sudo chmod -R 755 /var/www/my-web
```

---

### Bước 2: Cấu hình Server Block Nginx (`my-web.conf`)
Tạo tệp cấu hình Server Block tại `/etc/nginx/sites-available/my-web.conf`:
```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /var/www/my-web/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    # Cấu hình trang lỗi 404 tùy chỉnh
    error_page 404 /404.html;

    # Bảo vệ tệp tin lỗi bằng chỉ thị internal
    location = /404.html {
        root /var/www/my-web/html;
        internal;
    }

    access_log /var/log/nginx/my-web.access.log;
    error_log /var/log/nginx/my-web.error.log;
}
```

---

### Bước 3: Kích hoạt cấu hình và nạp lại dịch vụ Nginx
Tạo liên kết tượng trưng (symlink) sang thư mục `sites-enabled`:
```bash
sudo ln -sf /etc/nginx/sites-available/my-web.conf /etc/nginx/sites-enabled/
```

Kiểm tra cú pháp cấu hình Nginx trước khi nạp lại:
```bash
sudo nginx -t
```
> **Output:**
> ```text
> nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
> nginx: configuration file /etc/nginx/nginx.conf test is successful
> ```

Nạp lại cấu hình dịch vụ:
```bash
sudo systemctl reload nginx
```

---

## 4. Kết quả kiểm tra & Terminal Logs

### 4.1. Kiểm tra truy cập đường dẫn không tồn tại:
Chạy lệnh kiểm tra header HTTP:
```bash
curl -I http://165.101.46.48/invalid-path-demo
```
**Nhật ký Terminal hiển thị:**
```text
HTTP/1.1 404 Not Found
Server: nginx/1.18.0 (Ubuntu)
Date: Fri, 02 Oct 2026 00:49:41 GMT
Content-Type: text/html
Content-Length: 4082
Connection: keep-alive
ETag: "6abeff9f-ff2"
```

Kiểm tra nội dung HTML trả về:
```bash
curl -s http://165.101.46.48/invalid-path-demo | grep -C 2 "Không Tìm Thấy"
```
```html
        <div class="error-code">404</div>
        <div class="badge">Page Not Found</div>
        <h1>Không Tìm Thấy Đường Dẫn Yêu Cầu</h1>
        <p class="desc">
            Đường dẫn tĩnh bạn đang truy cập không tồn tại hoặc đã được di chuyển trên máy chủ PTIT Web Server. Vui lòng kiểm tra lại URL hoặc quay về trang chủ.
```
👉 **Đánh giá:** Trình duyệt nhận đúng mã trạng thái `404 Not Found` và hiển thị nội dung tùy biến của file `404.html`.

---

### 4.2. Kiểm tra truy cập trực tiếp tệp tin lỗi `/404.html`:
Chạy lệnh truy cập trực tiếp file lỗi:
```bash
curl -I http://165.101.46.48/404.html
```
**Nhật ký Terminal hiển thị:**
```text
HTTP/1.1 404 Not Found
Server: nginx/1.18.0 (Ubuntu)
Date: Fri, 02 Oct 2026 00:49:41 GMT
Content-Type: text/html
Content-Length: 4082
Connection: keep-alive
ETag: "6abeff9f-ff2"
```
👉 **Đánh giá:** Khi truy cập trực tiếp URL `/404.html`, chỉ thị `internal` đã ngăn chặn truy cập thẳng và trả về mã lỗi `404 Not Found` đúng như mong đợi.

---

### 4.3. Kiểm tra trên trình duyệt thực tế:
Mở trình duyệt và truy cập bất kỳ đường dẫn không hợp lệ, ví dụ:  
👉 **`http://165.101.46.48/duong-dan-khong-ton-tai`**  
Giao diện hiển thị trang lỗi 404 hiện đại với tông màu đỏ PTIT, có nút "← Quay Về Trang Chủ" hoạt động hoàn hảo.

---

## 5. Giải thích cơ chế bảo mật của chỉ thị `internal`
- Khi khai báo `internal` trong một `location block`, Nginx chỉ cho phép khối lệnh này được thực thi bởi các yêu cầu nội bộ (**Internal Requests**), bao gồm:
  1. Chỉ thị xử lý lỗi: `error_page`.
  2. Chỉ thị chuyển hướng máy chủ nội bộ: `rewrite ... last` hoặc `fastcgi_pass`.
- Người dùng từ bên ngoài (External Client) nếu cố tình gõ trực tiếp `http://<IP>/404.html` trên trình duyệt sẽ bị Nginx từ chối và phản hồi lại mã lỗi 404. Điều này ngăn chặn kẻ tấn công dò tìm các tệp tin tĩnh nhạy cảm hoặc bypass luồng xử lý của ứng dụng.

---

## 6. Danh mục tệp tin nộp bài
Các tệp đã được tạo và lưu trữ đầy đủ trong thư mục:
```text
Session_03/
└── ex2/
    ├── 404.html          # Mã nguồn trang lỗi 404 tùy biến
    ├── my-web.conf       # File cấu hình Server Block Nginx
    └── README.md         # Báo cáo thực hiện chi tiết
```
