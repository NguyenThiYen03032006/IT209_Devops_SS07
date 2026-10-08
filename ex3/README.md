# Bài 3: Thiết lập Cơ sở dữ liệu và Tự cấu hình dịch vụ Systemd cho Spring Boot

## 1. Thông tin bài thực hành

* Hệ điều hành: Ubuntu 22.04
* Java: OpenJDK 17
* Database: MySQL 8.0
* Database name: `springboot_db`
* MySQL user: `spring-admin`
* Application user: `spring-runner`
* Application: Spring Boot
* Application file: `/opt/spring-app/app.jar`
* Application port: `8082`
* Systemd service: `spring-app.service`

---

## 2. Khởi tạo cơ sở dữ liệu MySQL

### 2.1. Tạo database

Đăng nhập MySQL:

```bash
mysql
```

Tạo database:

```sql
CREATE DATABASE springboot_db;
```

Kiểm tra:

```sql
SHOW DATABASES;
```

Database `springboot_db` xuất hiện trong danh sách.

### 2.2. Tạo user MySQL

Tạo user:

```sql
CREATE USER 'spring-admin'@'localhost'
IDENTIFIED BY 'SpringSecure@123';
```

### 2.3. Cấp toàn quyền cho database

```sql
GRANT ALL PRIVILEGES ON springboot_db.*
TO 'spring-admin'@'localhost';

FLUSH PRIVILEGES;
```

Kiểm tra quyền:

```sql
SHOW GRANTS FOR 'spring-admin'@'localhost';
```

Kết quả cho thấy user `spring-admin` được cấp toàn quyền trên database `springboot_db`.

---

## 3. Tạo user hệ thống chạy ứng dụng

Tạo tài khoản hệ thống:

```bash
sudo useradd --system --no-create-home \
--shell /usr/sbin/nologin spring-runner
```

Kiểm tra:

```bash
id spring-runner
```

Kết quả:

```text
uid=999(spring-runner) gid=999(spring-runner) groups=999(spring-runner)
```

Kiểm tra shell:

```bash
getent passwd spring-runner
```

Kết quả có dạng:

```text
spring-runner:x:999:999::/home/spring-runner:/usr/sbin/nologin
```

Tài khoản `spring-runner` không có quyền đăng nhập shell thông thường và được sử dụng riêng để chạy ứng dụng Spring Boot.

---

## 4. Chuẩn bị thư mục ứng dụng

Tạo thư mục:

```bash
sudo mkdir -p /opt/spring-app
```

Đặt ứng dụng Spring Boot tại:

```text
/opt/spring-app/app.jar
```

Cấp quyền sở hữu cho tài khoản chạy ứng dụng:

```bash
sudo chown -R spring-runner:spring-runner /opt/spring-app
```

Kiểm tra:

```bash
ls -la /opt/spring-app/
```

Kết quả cho thấy `app.jar` thuộc sở hữu của `spring-runner`.

---

## 5. Cấu hình Spring Boot chạy cổng 8082

Ứng dụng Spring Boot được cấu hình sử dụng cổng:

```properties
server.port=8082
```

Ứng dụng được triển khai tại:

```text
/opt/spring-app/app.jar
```

Có thể kiểm tra ứng dụng bằng:

```bash
curl http://localhost:8082/
```

---

## 6. Tự viết Systemd Service

Tạo file:

```bash
sudo nano /etc/systemd/system/spring-app.service
```

Nội dung:

```ini
[Unit]
Description=Spring Boot Application
After=network.target

[Service]
User=spring-runner
Group=spring-runner
WorkingDirectory=/opt/spring-app
ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar
Restart=on-failure
RestartSec=10
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

### Giải thích các tham số chính

`User=spring-runner`

Ứng dụng được chạy dưới tài khoản giới hạn `spring-runner`, không chạy bằng `root`.

`ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar`

Đây là lệnh thực thi ứng dụng Spring Boot theo đúng yêu cầu đề bài.

`Restart=on-failure`

Systemd tự động khởi động lại ứng dụng khi ứng dụng bị crash hoặc kết thúc bất thường.

`RestartSec=10`

Systemd chờ 10 giây trước khi khởi động lại ứng dụng.

`WantedBy=multi-user.target`

Cho phép service được kích hoạt cùng hệ thống.

---

## 7. Nạp lại Systemd

Sau khi tạo service:

```bash
sudo systemctl daemon-reload
```

Khởi động service:

```bash
sudo systemctl start spring-app.service
```

Cho phép service tự động khởi động khi VPS khởi động:

```bash
sudo systemctl enable spring-app.service
```

---

## 8. Kiểm tra trạng thái Systemd

Sử dụng:

```bash
sudo systemctl status spring-app.service
```

Kết quả mong đợi:

```text
Active: active (running)
```

Service đang chạy ứng dụng:

```text
/usr/bin/java -jar /opt/spring-app/app.jar
```

### Minh chứng

![Systemd status](images/systemctl-status.png)

---

## 9. Kiểm tra cổng 8082

Thực hiện:

```bash
ss -tlnp | grep 8082
```

Kết quả mong đợi cho thấy Java đang LISTEN trên cổng `8082`.

Ví dụ:

```text
LISTEN 0 100 0.0.0.0:8082 0.0.0.0:* users:(("java",pid=1234,fd=...))
```

### Minh chứng

![Port 8082](images/ss-8082.png)

---

## 10. Kiểm tra ứng dụng chạy dưới spring-runner

Lấy PID của ứng dụng:

```bash
pgrep -f '/opt/spring-app/app.jar'
```

Sau đó kiểm tra:

```bash
ps -o user,pid,cmd -p <PID>
```

Kết quả mong đợi:

```text
USER           PID CMD
spring-runner  1234 /usr/bin/java -jar /opt/spring-app/app.jar
```

Điều này chứng minh ứng dụng không chạy dưới quyền `root`.

---

## 11. Kiểm tra tự động khởi động lại

Service được cấu hình:

```ini
Restart=on-failure
RestartSec=10
```

Khi ứng dụng kết thúc bất thường, Systemd sẽ tự động khởi động lại sau 10 giây.

Kiểm tra cấu hình:

```bash
sudo systemctl cat spring-app.service
```

Có thể kiểm tra trạng thái service bằng:

```bash
sudo systemctl status spring-app.service
```

---

## 12. Kết quả đạt được

Các yêu cầu của bài thực hành đã được hoàn thành:

* [x] Tạo database `springboot_db`.
* [x] Tạo MySQL user `spring-admin`.
* [x] Cấp toàn quyền cho `spring-admin` trên `springboot_db`.
* [x] Tạo Linux user `spring-runner`.
* [x] Cấu hình `spring-runner` không có shell login.
* [x] Đặt ứng dụng tại `/opt/spring-app/app.jar`.
* [x] Ứng dụng chạy bằng `/usr/bin/java -jar /opt/spring-app/app.jar`.
* [x] Ứng dụng lắng nghe trên port `8082`.
* [x] Systemd service chạy dưới user `spring-runner`.
* [x] Cấu hình `Restart=on-failure`.
* [x] Cấu hình `RestartSec=10`.
* [x] Service được cấu hình tự động khởi động cùng hệ thống.
* [x] Kiểm tra trạng thái bằng `systemctl status`.
* [x] Kiểm tra port bằng `ss -tlnp`.

---

## 13. Minh chứng nộp bài

Các minh chứng được sử dụng trong báo cáo:

### Minh chứng 1 — MySQL

Kiểm tra database và quyền:

```bash
mysql -u spring-admin -p
```

```sql
SHOW DATABASES;
SHOW GRANTS FOR 'spring-admin'@'localhost';
```

### Minh chứng 2 — Systemd

```bash
sudo systemctl status spring-app.service
```

Ảnh: `images/systemctl-status.png`

### Minh chứng 3 — Port 8082

```bash
ss -tlnp | grep 8082
```

Ảnh: `images/ss-8082.png`

### Minh chứng 4 — User chạy ứng dụng

```bash
ps -o user,pid,cmd -p <PID>
```

Kết quả xác nhận Java chạy dưới `spring-runner`.

---

## 14. Cấu trúc bài nộp

```text
ex3/
├── spring-app.service
└── README.md
```

Bài nộp được đặt tại:

```text
homework/session_07/ex3/
```
