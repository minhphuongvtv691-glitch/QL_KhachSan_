# BÁO CÁO TRIỂN KHAI HỆ THỐNG QUẢN LÝ KHÁCH SẠN

---

## I. Tổng quan kiến trúc hệ thống

Hệ thống Quản lý Khách Sạn được triển khai theo mô hình **microservice** với 4 thành phần độc lập, tất cả đều được đóng gói và vận hành thông qua **Docker container** trên máy chủ Linux (Ubuntu Server). Kiến trúc này đảm bảo tính nhất quán, dễ triển khai và dễ mở rộng.

```
┌─────────────────────────────────────────────────────────┐
│                  MÁY CHỦ LINUX (Ubuntu)                 │
│                  IP: 192.168.56.102                     │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │
│  │  Nginx   │  │  Java    │  │ MariaDB  │  │Adminer │  │
│  │  :3000   │→ │ Backend  │→ │  :3307   │  │ :8082  │  │
│  │ Web UI   │  │  :8081   │  │  DB      │  │ DB GUI │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘  │
│                                                         │
│              Docker Network (bridge)                    │
└─────────────────────────────────────────────────────────┘
```

---

## II. Cài đặt và Cấu hình Máy chủ Linux

### 1. Môi trường triển khai

| Thông tin | Chi tiết |
|---|---|
| Hệ điều hành | Ubuntu Server 22.04 LTS |
| Phần mềm ảo hóa | VirtualBox / VMware |
| Địa chỉ IP máy ảo | 192.168.56.102 |
| Chế độ mạng | Host-only / Bridged |

---

### 2. Cấu hình dịch vụ SSH

SSH (Secure Shell) được cài đặt và cấu hình để cho phép quản trị viên kết nối từ xa vào máy chủ Linux một cách bảo mật.

**Cài đặt SSH Server:**
```bash
sudo apt update
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
```

**Kiểm tra trạng thái:**
```bash
sudo systemctl status ssh
# Kết quả: active (running) ✓
```

**Cấu hình bảo mật SSH** (`/etc/ssh/sshd_config`):
```bash
# Cài đặt cơ bản
Port 22
ListenAddress 0.0.0.0

# Khóa xác thực máy chủ
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_ed25519_key

# Bảo mật đăng nhập
PermitRootLogin no          # Cấm đăng nhập bằng tài khoản root
MaxAuthTries 3              # Tối đa 3 lần thử sai mật khẩu
LoginGraceTime 60           # Hết thời gian sau 60 giây nếu chưa đăng nhập
PasswordAuthentication yes  # Cho phép đăng nhập bằng mật khẩu
PubkeyAuthentication yes    # Cho phép đăng nhập bằng SSH key

# Phân quyền truy cập
AllowUsers TOAN             # Chỉ cho phép user TOAN SSH vào server

# Quản lý kết nối
ClientAliveInterval 300     # Gửi tín hiệu kiểm tra kết nối mỗi 5 phút
ClientAliveCountMax 2       # Ngắt kết nối nếu không phản hồi 2 lần
MaxSessions 5               # Tối đa 5 phiên SSH cùng lúc

# Ghi log
SyslogFacility AUTH
LogLevel INFO
```

**Áp dụng cấu hình:**
```bash
sudo sshd -t                    # Kiểm tra cú pháp
sudo systemctl restart ssh      # Khởi động lại SSH
```

**Kết nối SSH từ máy Windows:**
```powershell
ssh TOAN@192.168.56.102
```

---

### 3. Cài đặt FTP/SFTP (Vsftpd)

Vsftpd (Very Secure FTP Daemon) được cài đặt để hỗ trợ truyền file giữa máy Windows và máy chủ Linux thông qua giao thức SFTP.

```bash
sudo apt install vsftpd -y
sudo systemctl enable vsftpd
sudo systemctl start vsftpd
sudo systemctl status vsftpd
# Kết quả: active (running) ✓
```

**Kết nối SFTP từ Windows** (dùng FileZilla):
- Host: `192.168.56.102`
- Protocol: SFTP
- Port: `22`
- Username/Password: thông tin user Linux

---

## III. Triển khai Cơ sở Dữ liệu bằng Docker Container

### Lý do chọn Docker thay cho cài đặt trực tiếp

Thay vì cài MySQL/MariaDB trực tiếp lên máy chủ, nhóm triển khai cơ sở dữ liệu thông qua **Docker container** vì các ưu điểm:

| Tiêu chí | Cài trực tiếp | Docker Container |
|---|---|---|
| Tính nhất quán | Phụ thuộc OS | Giống nhau mọi môi trường |
| Khởi động lại | Phức tạp | `docker-compose up` |
| Backup/Restore | Thủ công | Volume mapping |
| Xung đột phiên bản | Có thể xảy ra | Không xảy ra |

### Cấu hình container MariaDB

```yaml
db:
  image: mariadb:11                    # Dùng MariaDB phiên bản 11
  environment:
    MYSQL_ROOT_PASSWORD: rootpass      # Mật khẩu root
    MYSQL_DATABASE: hotel_prod_db      # Tên database tự động tạo
  ports:
    - "3307:3306"                      # Ánh xạ cổng ra ngoài
  volumes:
    - db_data:/var/lib/mysql           # Volume lưu dữ liệu bền vững
    - ./database/hotel.sql:/docker-entrypoint-initdb.d/init.sql  # Khởi tạo DB
  restart: unless-stopped             # Tự khởi động lại nếu bị lỗi
```

**Cơ chế khởi tạo dữ liệu tự động:**
Khi container `db` khởi động lần đầu với volume trống, MariaDB tự động thực thi file `hotel.sql` đặt trong thư mục `/docker-entrypoint-initdb.d/`. File SQL này tạo toàn bộ cấu trúc bảng và dữ liệu mẫu cho hệ thống khách sạn.

**Quản lý dữ liệu qua Adminer:**
Adminer là công cụ quản lý CSDL qua giao diện web, được triển khai song song:
```
Truy cập: http://192.168.56.102:8082
Server:   db
Username: root
Password: rootpass
Database: hotel_prod_db
```

---

## IV. Triển khai Web Server bằng Docker Container

### 1. Web UI - Nginx Container

```yaml
web:
  image: nginx:alpine                          # Image Nginx nhẹ, hiệu năng cao
  volumes:
    - ./web-ui/public:/usr/share/nginx/html    # Mount thư mục static files
  ports:
    - "3000:80"                                # Truy cập qua cổng 3000
  depends_on:
    - backend                                  # Chờ Backend khởi động xong
  restart: unless-stopped
```

### 2. Backend API - Java Container (Dockerfile)

```dockerfile
FROM eclipse-temurin:17-jdk          # Java 17 JDK

WORKDIR /app
ENV TZ=Asia/Ho_Chi_Minh

COPY src/ ./src/
COPY lib/ ./lib/

# Biên dịch toàn bộ mã nguồn Java
RUN mkdir -p build/classes && \
    find src -name "*.java" > sources.txt && \
    javac -encoding utf-8 -cp "lib/*:build/classes" -d build/classes @sources.txt

EXPOSE 8081
CMD ["java", "-cp", "lib/*:build/classes", "quanlykhachsan.backend.Main"]
```

---

## V. Vận hành Container

### Khởi động hệ thống

```bash
# Build image và khởi động tất cả container ở chế độ nền
sudo docker-compose up --build -d
```

### Kiểm tra trạng thái

```bash
# Xem tất cả container đang chạy
sudo docker-compose ps

# Xem log của từng service
sudo docker-compose logs backend
sudo docker-compose logs db

# Truy cập shell bên trong container
sudo docker exec -it ql_khachsan-db-1 bash
```

### Dừng hệ thống

```bash
sudo docker-compose down         # Dừng, giữ nguyên dữ liệu
sudo docker-compose down -v      # Dừng và xóa toàn bộ dữ liệu
```

### Kết quả sau khi triển khai

| Dịch vụ | Địa chỉ truy cập | Trạng thái |
|---|---|---|
| Web UI (Nginx) | http://192.168.56.102:3000 | ✅ Hoạt động |
| Backend API (Java) | http://192.168.56.102:8081/api | ✅ Hoạt động |
| Database (MariaDB) | 192.168.56.102:3307 | ✅ Hoạt động |
| Quản lý DB (Adminer) | http://192.168.56.102:8082 | ✅ Hoạt động |

---

## VI. Phân quyền và Bảo mật

### Phân quyền người dùng hệ thống web

| Vai trò | Username | Quyền hạn |
|---|---|---|
| Quản trị viên | `admin_main` | Toàn quyền: phòng, đặt phòng, khách hàng, nhân viên, báo cáo |
| Nhân viên | `staff_01` | Sơ đồ phòng, đặt phòng, thanh toán, hóa đơn |
| Khách hàng | `customer_test` | Khám phá phòng, đặt phòng, lịch sử |

### Bảo mật API

- Xác thực bằng JWT Token
- Phân quyền theo role qua header `X-User-Role`
- Mật khẩu được mã hóa bằng BCrypt

