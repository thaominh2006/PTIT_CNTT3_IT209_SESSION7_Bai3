## 0. Ghi chú về môi trường thực hiện

- Môi trường: Windows 10 + WSL2 (Ubuntu 22.04.5 LTS), đã bật Systemd (`systemd=true` trong `/etc/wsl.conf`).
- MySQL Server 8.0.46, OpenJDK 17.
- Do chưa có tệp `app.jar` của lớp, em dùng một `app.jar` mẫu tự biên dịch (HTTP server đơn giản bằng `com.sun.net.httpserver`, lắng nghe cổng 8082) để thực hành. Tệp mẫu này không kết nối MySQL; tệp `spring-app.service` không phụ thuộc vào nội dung `app.jar` nên có thể dùng nguyên cho `app.jar` thật.

## 1. Khởi tạo MySQL

```sql
CREATE DATABASE springboot_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'spring-admin'@'localhost' IDENTIFIED BY 'SpringSecure@123';
GRANT ALL PRIVILEGES ON springboot_db.* TO 'spring-admin'@'localhost';
FLUSH PRIVILEGES;
SHOW GRANTS FOR 'spring-admin'@'localhost';
```

Kết quả `SHOW GRANTS`:

```
+---------------------------------------------------------------------+
| Grants for spring-admin@localhost                                   |
+---------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `spring-admin`@`localhost`                    |
| GRANT ALL PRIVILEGES ON `springboot_db`.* TO `spring-admin`@`localhost` |
+---------------------------------------------------------------------+
2 rows in set (0.01 sec)
```

## 2. Tạo user hệ thống không có shell login

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin spring-runner
id spring-runner
getent passwd spring-runner
```

Kết quả:

```
uid=999(spring-runner) gid=999(spring-runner) groups=999(spring-runner)
spring-runner:x:999:999::/home/spring-runner:/usr/sbin/nologin
```

## 3. Chuẩn bị thư mục và tệp jar

```bash
sudo mkdir -p /opt/spring-app
sudo cp ~/app.jar /opt/spring-app/app.jar
sudo chown -R spring-runner:spring-runner /opt/spring-app
sudo chmod 500 /opt/spring-app/app.jar
ls -l /opt/spring-app
```

Kết quả:

```
total 4
-r-x------ 1 spring-runner spring-runner 1581 Oct  8 20:03 app.jar
```

## 4. Tệp dịch vụ /etc/systemd/system/spring-app.service

Tệp được tự viết từ đầu (không dùng tệp mẫu), có trong cùng thư mục này với tên `spring-app.service`.

```ini
[Unit]
Description=Spring Boot Application (app.jar)
After=network.target mysql.service
Wants=mysql.service

[Service]
Type=simple
User=spring-runner
Group=spring-runner
WorkingDirectory=/opt/spring-app

# Cau hinh ket noi database va cong ung dung
Environment="SERVER_PORT=8082"
Environment="SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/springboot_db?useSSL=false&serverTimezone=UTC"
Environment="SPRING_DATASOURCE_USERNAME=spring-admin"
Environment="SPRING_DATASOURCE_PASSWORD=SpringSecure@123"

ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar

# Tu dong khoi dong lai sau 10 giay neu ung dung bi crash
Restart=on-failure
RestartSec=10

# Java thoat voi ma 143 khi nhan SIGTERM, coi day la thoat binh thuong
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

Giải thích các tham số chính:

| Tham số | Ý nghĩa |
|---|---|
| `User=spring-runner` | Chạy ứng dụng bằng tài khoản giới hạn (non-root) |
| `ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar` | Lệnh thực thi ứng dụng theo đúng yêu cầu |
| `Restart=on-failure` | Tự khởi động lại khi tiến trình thoát bất thường |
| `RestartSec=10` | Chờ 10 giây trước khi khởi động lại |
| `SERVER_PORT=8082` | Cấu hình cổng lắng nghe 8082 |
| `SPRING_DATASOURCE_*` | Thông tin kết nối tới `springboot_db` bằng user `spring-admin` |

## 5. Nạp lại và khởi động dịch vụ

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now spring-app.service
```

## 6. Kết quả kiểm tra

### 6.1. Trạng thái dịch vụ (lần chạy đầu)

```bash
sudo systemctl status spring-app.service --no-pager
```

```
● spring-app.service - Spring Boot Application (app.jar)
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2026-10-08 20:07:31 +07; 37s ago
   Main PID: 3742 (java)
      Tasks: 21 (limit: 4577)
     Memory: 21.2M
        CPU: 172ms
     CGroup: /system.slice/spring-app.service
             └─3742 /usr/bin/java -jar /opt/spring-app/app.jar

Oct 08 20:07:31 DESKTOP-OFRUK8L systemd[1]: Started Spring Boot Application (app.jar).
Oct 08 20:07:31 DESKTOP-OFRUK8L java[3742]: Server started on port 8082
```

### 6.2. Cổng lắng nghe và user chạy tiến trình (lần chạy đầu)

```bash
sudo ss -tlnp | grep 8082
ps -o user:20,pid,cmd -C java
```

```
LISTEN 0      50                 *:8082            *:*    users:(("java",pid=3742,fd=5))

USER                  PID CMD
spring-runner        3742 /usr/bin/java -jar /opt/spring-app/app.jar
```

### 6.3. Kiểm tra tự khởi động lại khi crash

Giả lập crash bằng cách buộc dừng tiến trình Java, chờ 12 giây rồi kiểm tra lại:

```bash
sudo pkill -9 java
sleep 12
sudo systemctl status spring-app.service --no-pager
sudo ss -tlnp | grep 8082
ps -o user:20,pid,cmd -C java
```

```
● spring-app.service - Spring Boot Application (app.jar)
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2026-10-08 20:11:36 +07; 29s ago
   Main PID: 3807 (java)
      Tasks: 21 (limit: 4577)
     Memory: 20.9M
        CPU: 160ms
     CGroup: /system.slice/spring-app.service
             └─3807 /usr/bin/java -jar /opt/spring-app/app.jar

Oct 08 20:11:36 DESKTOP-OFRUK8L systemd[1]: Started Spring Boot Application (app.jar).
Oct 08 20:11:36 DESKTOP-OFRUK8L java[3807]: Server started on port 8082

LISTEN 0      50                 *:8082            *:*    users:(("java",pid=3807,fd=5))

USER                  PID CMD
spring-runner        3807 /usr/bin/java -jar /opt/spring-app/app.jar
```

## 7. Nhận xét

- Dịch vụ `spring-app.service` ở trạng thái `active (running)`: [PASS]
- Cổng 8082 được tiến trình `java` lắng nghe, chạy dưới quyền `spring-runner` (non-root, không có shell login): [PASS]
- Sau khi tiến trình bị `kill -9`, Systemd tự khởi động lại sau khoảng 10 giây với PID mới (3742 chuyển thành 3807), chứng tỏ `Restart=on-failure` và `RestartSec=10` hoạt động đúng: [PASS]
- Dịch vụ được `enable` nên tự chạy cùng hệ thống ở target `multi-user.target`.
- Lưu ý bảo mật: mật khẩu đang đặt trực tiếp trong tệp service theo yêu cầu bài tập. Trong môi trường thực tế nên dùng `EnvironmentFile=` với tệp có quyền `600` để không lộ mật khẩu.