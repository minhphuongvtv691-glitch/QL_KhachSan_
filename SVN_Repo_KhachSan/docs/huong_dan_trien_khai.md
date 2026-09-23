# HƯỚNG DẪN TRIỂN KHAI PROJECT LÊN MÁY ẢO LINUX

> **Project:** Hệ thống Quản lý Khách Sạn  
> **Repository:** https://github.com/TL2505/QL_KhachSan.git  
> **Nhánh:** `feature/web-ui-imp`  
> **IP Máy ảo:** `192.168.56.102`

---

## BƯỚC 0 — Chuẩn bị (Thực hiện trên máy Windows)

Trước khi đưa lên máy ảo, đảm bảo code mới nhất đã được đẩy lên GitHub:

```powershell
# Mở terminal trong thư mục project
cd C:\Users\Admin\OneDrive\Documents\NetBeansProjects\QL_KhachSan0

# Kiểm tra các file chưa commit
git status

# Commit và push code mới nhất
git add .
git commit -m "Cập nhật code trước khi triển khai"
git push origin feature/web-ui-imp
```

---

## BƯỚC 1 — Cài đặt Docker trên Ubuntu (Chỉ làm 1 lần)

SSH vào máy ảo từ Windows:
```powershell
ssh TOAN@192.168.56.102
```

Cài Docker và Docker Compose:
```bash
sudo apt update
sudo apt install docker.io docker-compose -y
```

Kiểm tra cài đặt thành công:
```bash
docker --version
docker-compose --version
```

---

## BƯỚC 2 — Clone Project từ GitHub

### Nếu là lần ĐẦU TIÊN (chưa có code trên máy ảo):

```bash
cd ~
git clone https://github.com/TL2505/QL_KhachSan.git
cd QL_KhachSan
git fetch origin
git checkout feature/web-ui-imp
```

### Nếu đã clone rồi (lần cập nhật tiếp theo):

```bash
cd ~/QL_KhachSan
git stash
git pull origin feature/web-ui-imp
```

---

## BƯỚC 3 — Khởi động hệ thống bằng Docker

```bash
cd ~/QL_KhachSan
sudo docker-compose down
sudo docker-compose up --build -d
```

Lệnh này tự động:
1. Build Java Backend từ mã nguồn
2. Khởi động MariaDB + nạp file `database/hotel.sql`
3. Khởi động Nginx phục vụ Web UI
4. Khởi động Adminer quản lý DB

> ⏱️ Lần đầu mất khoảng 2–5 phút do cần tải image và biên dịch Java.

---

## BƯỚC 4 — Kiểm tra hệ thống

```bash
sudo docker-compose ps
```

Kết quả mong đợi:
```
Name                    Command    State    Ports
--------------------------------------------------------------
ql_khachsan-db-1       mysqld      Up    0.0.0.0:3307->3306/tcp
ql_khachsan-backend-1  java -cp    Up    0.0.0.0:8081->8081/tcp
ql_khachsan-web-1      nginx       Up    0.0.0.0:3000->80/tcp
ql_khachsan-adminer-1  php         Up    0.0.0.0:8082->8080/tcp
```

### Truy cập từ máy Windows:

| Dịch vụ | Địa chỉ | Mô tả |
|---|---|---|
| 🌐 Web UI | http://192.168.56.102:3000 | Giao diện chính |
| ⚙️ API Backend | http://192.168.56.102:8081/api | API Server |
| 🗄️ Quản lý DB | http://192.168.56.102:8082 | Adminer |

### Tài khoản đăng nhập:

| Vai trò | Username | Password |
|---|---|---|
| Admin | `admin_main` | `123456` |
| Nhân viên | `staff_01` | `123456` |
| Khách hàng | `customer_test` | `123456` |

---

## BƯỚC 5 — Import dữ liệu qua Adminer (Nếu cần)

1. Vào `http://192.168.56.102:8082`
2. Đăng nhập: Server=`db`, User=`root`, Pass=`rootpass`, DB=`hotel_prod_db`
3. Chọn **Import** → Upload file `.sql` export từ XAMPP

---

## XỬ LÝ LỖI THƯỜNG GẶP

| Lỗi | Lệnh xử lý |
|---|---|
| `docker-compose not found` | `sudo apt install docker-compose -y` |
| Local changes overwritten | `git stash` rồi `git pull` |
| Network has active endpoints | `sudo docker stop $(sudo docker ps -aq)` rồi `docker-compose up` |
| Backend không kết nối DB | `sudo docker-compose logs backend` để xem nguyên nhân |

---

## CÁC LỆNH VẬN HÀNH HÀNG NGÀY

```bash
# Xem log realtime
sudo docker-compose logs -f

# Xem log 1 service cụ thể
sudo docker-compose logs -f backend

# Truy cập shell trong container DB
sudo docker exec -it ql_khachsan-db-1 bash

# Dừng hệ thống (GIỮ dữ liệu)
sudo docker-compose down

# Dừng và XÓA SẠCH dữ liệu
sudo docker-compose down -v

# Khởi động lại 1 service
sudo docker-compose restart web
```
