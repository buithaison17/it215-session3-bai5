# Bài 5: Thiết lập thư mục Web an toàn tại phân vùng hệ thống (/opt/)

> **Lưu ý:** Các khối output (`ls -la`, `tail`, `curl`...) bên dưới là kết quả mẫu. Hãy chạy lại các lệnh trên máy chủ của bạn và thay bằng output thực tế (kèm ảnh chụp màn hình nếu cần) trước khi nộp.

## 1. Mục tiêu

- Chuyển mã nguồn website tĩnh và logs sang `/opt/my-app/` để cô lập khỏi các đường dẫn mặc định (`/var/www`, `/var/log/nginx`).
- Phân quyền rõ ràng: `devops` quản lý mã nguồn, `www-data` (Nginx) chỉ đọc web và ghi log.

## 2. Thiết kế phân quyền

| Đường dẫn | Owner:Group | Thư mục | File | Lý do |
|---|---|---|---|---|
| `/opt/my-app/` | `devops:www-data` | `750` | - | Group cần `x` để đi vào |
| `/opt/my-app/html/` | `devops:www-data` | `750` | `640` | devops đọc/ghi, www-data chỉ đọc, others không truy cập |
| `/opt/my-app/logs/` | `devops:www-data` | `2770` | `660` | www-data ghi log; bit setgid giữ group `www-data` cho file mới |

## 3. Các bước thực hiện

### Bước 1: Tạo cấu trúc thư mục

```bash
sudo mkdir -p /opt/my-app/html /opt/my-app/logs
```

Nếu đã có website ở `/var/www/ptit-web/`, di chuyển mã nguồn sang:

```bash
sudo cp -a /var/www/ptit-web/html/. /opt/my-app/html/
```

Nếu chưa có, tạo trang thử:

```bash
echo "<h1>My App - served from /opt/my-app</h1>" | sudo tee /opt/my-app/html/index.html
```

### Bước 2: Đổi chủ sở hữu và nhóm (đệ quy)

```bash
sudo chown -R devops:www-data /opt/my-app/
```

### Bước 3: Phân quyền

```bash
# Thư mục: 750
sudo find /opt/my-app -type d -exec chmod 750 {} \;

# File trong html: 640
sudo find /opt/my-app/html -type f -exec chmod 640 {} \;

# Thư mục logs: group www-data được ghi, kèm setgid
sudo chmod 2770 /opt/my-app/logs

# Tạo sẵn file log để chắc chắn đúng quyền
sudo touch /opt/my-app/logs/access.log /opt/my-app/logs/error.log
sudo chown devops:www-data /opt/my-app/logs/*.log
sudo chmod 660 /opt/my-app/logs/*.log
```

> **Tại sao không `chmod -R 640` cho tất cả?** Thư mục thiếu quyền `x` sẽ khiến Nginx không đi vào được, dẫn đến lỗi `403 Forbidden`.

### Bước 4: Cấu hình Nginx

Sao chép file `my-app.conf` (nộp kèm bài) vào máy chủ:

```bash
sudo cp my-app.conf /etc/nginx/sites-available/my-app.conf
sudo ln -s /etc/nginx/sites-available/my-app.conf /etc/nginx/sites-enabled/my-app.conf

# Gỡ site mặc định để tránh xung đột port 80
sudo rm -f /etc/nginx/sites-enabled/default

sudo nginx -t
sudo systemctl reload nginx
```

Nội dung các chỉ thị chính:

```nginx
root /opt/my-app/html;
access_log /opt/my-app/logs/access.log;
error_log  /opt/my-app/logs/error.log warn;
```

Kết quả `nginx -t` mong đợi:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

## 4. Kiểm tra

### 4.1. Trạng thái phân quyền

```bash
ls -la /opt/my-app/
```

```text
total 16
drwxr-x--- 4 devops www-data 4096 Oct  1 10:00 .
drwxr-xr-x 3 root   root     4096 Oct  1 09:50 ..
drwxr-x--- 2 devops www-data 4096 Oct  1 10:00 html
drwxrws--- 2 devops www-data 4096 Oct  1 10:05 logs
```

```bash
ls -la /opt/my-app/html/ /opt/my-app/logs/
```

```text
/opt/my-app/html/:
total 12
drwxr-x--- 2 devops www-data 4096 Oct  1 10:00 .
drwxr-x--- 4 devops www-data 4096 Oct  1 10:00 ..
-rw-r----- 1 devops www-data   45 Oct  1 10:00 index.html

/opt/my-app/logs/:
total 8
drwxrws--- 2 devops www-data 4096 Oct  1 10:05 .
drwxr-x--- 4 devops www-data 4096 Oct  1 10:00 ..
-rw-rw---- 1 devops www-data    0 Oct  1 10:05 access.log
-rw-rw---- 1 devops www-data    0 Oct  1 10:05 error.log
```

Owner/group là `devops:www-data`. Thư mục logs có `s` (setgid) ở nhóm.

### 4.2. Ghi file bằng user `devops` (không sudo)

```bash
su - devops
whoami
echo "Update Test" >> /opt/my-app/html/index.html
echo "Exit code: $?"
tail -n 2 /opt/my-app/html/index.html
```

```text
devops
Exit code: 0
<h1>My App - served from /opt/my-app</h1>
Update Test
```

Ghi thành công, không có lỗi `Permission denied`.

### 4.3. Nginx hoạt động và ghi log

```bash
# www-data đọc được file
sudo -u www-data cat /opt/my-app/html/index.html

# Truy cập web
curl -I http://localhost
curl http://localhost
```

```text
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
...
```

Xem log phát sinh:

```bash
tail -n 5 /opt/my-app/logs/access.log
```

```text
127.0.0.1 - - [01/Oct/2026:10:10:21 +0700] "HEAD / HTTP/1.1" 200 0 "-" "curl/8.5.0"
127.0.0.1 - - [01/Oct/2026:10:10:25 +0700] "GET / HTTP/1.1" 200 58 "-" "curl/8.5.0"
```

Theo dõi trực tiếp (mở terminal khác và gọi `curl http://localhost`):

```bash
tail -f /opt/my-app/logs/access.log
```

## 5. Xử lý sự cố

| Hiện tượng | Nguyên nhân / Cách xử lý |
|---|---|
| `403 Forbidden` | Thiếu `x` trên thư mục: `sudo find /opt/my-app -type d -exec chmod 750 {} \;`; kiểm tra bằng `namei -l /opt/my-app/html/index.html` |
| `nginx -t` báo `Permission denied` ở log | Kiểm tra quyền thư mục `logs/` (2770) và group `www-data` |
| Log không ghi | `sudo tail /var/log/nginx/error.log`, kiểm tra đường dẫn `access_log` trong `my-app.conf` |
| Trang mặc định Nginx vẫn hiện | Chưa gỡ `sites-enabled/default` hoặc chưa reload |

## 6. Kết luận

- Mã nguồn và logs được cô lập tại `/opt/my-app/`, tách khỏi phân vùng mặc định.
- Toàn bộ thư mục thuộc `devops:www-data`; `devops` sửa mã nguồn không cần sudo.
- `www-data` chỉ đọc thư mục web, và được ghi vào `logs/` để Nginx ghi log.
- Nginx phục vụ web bình thường (không lỗi 403), `access_log` và `error_log` ghi vào `/opt/my-app/logs/`.

## 7. Tệp nộp

- `my-app.conf`: cấu hình Nginx server block
- `README.md`: báo cáo này
