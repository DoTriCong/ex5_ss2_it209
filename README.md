# Bài 5: Quản lý quyền sở hữu thư mục Web (Directory Permission Management)

## 1. Mục tiêu

- Hiểu cơ chế phân quyền tệp tin/thư mục trên Linux (`chown`, `chmod`).
- Cho phép user `devops` sửa mã nguồn trong `/var/www/ptit-web/` **không cần sudo**, đồng thời Nginx (user `www-data`) vẫn đọc được để phục vụ web tĩnh, không bị lỗi `403 Forbidden`.

## 2. Kiến thức nền

Quyền trên Linux gồm 3 nhóm đối tượng (**u**ser / **g**roup / **o**thers) và 3 loại quyền:

| Ký hiệu | Số | Với file | Với thư mục |
|---|---|---|---|
| `r` | 4 | Đọc nội dung | Liệt kê file bên trong |
| `w` | 2 | Ghi/sửa nội dung | Tạo/xóa/đổi tên file bên trong |
| `x` | 1 | Thực thi | **Đi vào (traverse)** thư mục |

> **Lưu ý quan trọng:** Thư mục bắt buộc phải có quyền `x` thì Nginx mới truy cập được file bên trong. Nếu chạy `chmod -R 640` cho mọi thứ, thư mục sẽ mất quyền `x` và Nginx trả về **403 Forbidden**. Vì vậy phải phân quyền **tách riêng** cho thư mục và file.

### Quyền đề xuất

| Đối tượng | Thư mục | File | Ý nghĩa |
|---|---|---|---|
| Owner `devops` | `rwx` (7) | `rw-` (6) | Đọc, ghi, vào thư mục |
| Group `www-data` | `r-x` (5) | `r--` (4) | Nginx chỉ đọc |
| Others | `---` (0) | `---` (0) | Không truy cập |

→ Thư mục: **`750`**, File: **`640`**.

## 3. Các bước thực hiện

### Bước 1: Kiểm tra hiện trạng

```bash
ls -la /var/www/ptit-web/
id devops
id www-data
```

Hiện tại thư mục thuộc `root:root` nên `devops` không ghi được.

### Bước 2: Đổi chủ sở hữu và nhóm sở hữu (đệ quy)

```bash
sudo chown -R devops:www-data /var/www/ptit-web/
```

- `-R`: áp dụng đệ quy cho toàn bộ nội dung bên trong.
- `devops:www-data`: owner = `devops`, group = `www-data`.

### Bước 3: Phân quyền chmod (tách thư mục và file)

```bash
# Thư mục: rwx cho owner, r-x cho group, không cho others
sudo find /var/www/ptit-web/ -type d -exec chmod 750 {} \;

# File: rw- cho owner, r-- cho group, không cho others
sudo find /var/www/ptit-web/ -type f -exec chmod 640 {} \;
```

> (Tùy chọn) Để file mới tạo trong thư mục tự động thuộc nhóm `www-data`, bật bit setgid:
> ```bash
> sudo find /var/www/ptit-web/ -type d -exec chmod 2750 {} \;
> ```

### Bước 4: Đảm bảo đường dẫn cha cho phép đi qua

```bash
namei -l /var/www/ptit-web/html/index.html
```

Các thư mục cha (`/`, `/var`, `/var/www`) cần có quyền `x` cho others (mặc định là `755`).

### Bước 5: Kiểm tra phân quyền

```bash
ls -la /var/www/ptit-web/
```

**Kết quả** *(ví dụ — thay bằng output thực tế trên máy của bạn)*:

```text
total 12
drwxr-x--- 3 devops www-data 4096 Oct  1 10:00 .
drwxr-xr-x 3 root   root     4096 Oct  1 09:30 ..
drwxr-x--- 2 devops www-data 4096 Oct  1 10:00 html
```

```bash
ls -la /var/www/ptit-web/html/
```

```text
total 12
drwxr-x--- 2 devops www-data 4096 Oct  1 10:00 .
drwxr-x--- 3 devops www-data 4096 Oct  1 10:00 ..
-rw-r----- 1 devops www-data  612 Oct  1 10:00 index.html
```

Giải thích: owner = `devops`, group = `www-data`; thư mục `drwxr-x---` (750), file `-rw-r-----` (640).

### Bước 6: Kiểm tra ghi file bằng user `devops` (không dùng sudo)

```bash
su - devops
whoami
echo "Update" >> /var/www/ptit-web/html/index.html
echo "Exit code: $?"
tail -n 3 /var/www/ptit-web/html/index.html
```

**Kết quả mong đợi** *(ví dụ)*:

```text
devops
Exit code: 0
...
Update
```

Ghi file thành công, không có lỗi `Permission denied`.

### Bước 7: Kiểm tra Nginx vẫn phục vụ được web

```bash
# Kiểm tra www-data đọc được file
sudo -u www-data cat /var/www/ptit-web/html/index.html

# Kiểm tra cấu hình và reload Nginx
sudo nginx -t
sudo systemctl reload nginx

# Gọi thử website, kỳ vọng HTTP 200 và có nội dung "Update"
curl -I http://localhost
curl http://localhost
```

**Kết quả mong đợi** *(ví dụ)*:

```text
HTTP/1.1 200 OK
Server: nginx
...
```

Nội dung mới (`Update`) hiển thị bình thường, không bị lỗi 403.

## 4. Xử lý sự cố (nếu gặp 403 Forbidden)

| Nguyên nhân | Cách kiểm tra / khắc phục |
|---|---|
| Thư mục thiếu quyền `x` cho group | `find /var/www/ptit-web -type d -exec chmod 750 {} \;` |
| Group sở hữu không phải `www-data` | `sudo chown -R devops:www-data /var/www/ptit-web/` |
| Nginx chạy bằng user khác | Kiểm tra dòng `user` trong `/etc/nginx/nginx.conf` |
| Thư mục cha chặn truy cập | `namei -l /var/www/ptit-web/html/index.html` |
| Xem log lỗi | `sudo tail -f /var/log/nginx/error.log` |

## 5. Kết luận

- Chủ sở hữu thư mục là `devops`, nhóm là `www-data`.
- `devops` có quyền đọc/ghi (sửa mã nguồn không cần sudo).
- `www-data` (Nginx) chỉ có quyền đọc, đủ để phục vụ web tĩnh.
- Người dùng khác không có quyền truy cập.
- Website hoạt động bình thường, không bị lỗi `403 Forbidden`.
