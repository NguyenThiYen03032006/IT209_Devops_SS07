# Bài 5: Chẩn đoán và xử lý xung đột cổng mạng (Address Already in Use)

## 1. Mục tiêu

-   Kiểm tra các cổng mạng đang được lắng nghe trên Linux.
-   Xác định PID của tiến trình đang chiếm cổng 8082.
-   Sử dụng `ps` và `kill` để kiểm tra và giải phóng tiến trình.
-   Khởi động lại `spring-app.service` và xác nhận Java đã quản lý cổng
    8082.

## 2. Bối cảnh

Ứng dụng Spring Boot `spring-app.service` sử dụng cổng `8082`.

Để mô phỏng lỗi `Address already in use`, cổng 8082 được chiếm trước bởi
một tiến trình Python chạy nền:

``` bash
python3 -m http.server 8082 > /dev/null 2>&1 &
```

Khi đó nếu Spring Boot cũng cố gắng sử dụng cổng 8082, ứng dụng có thể
không khởi động được do cổng đã được sử dụng.

## 3. Giả lập tiến trình chiếm cổng 8082

Lệnh đã sử dụng:

``` bash
python3 -m http.server 8082 > /dev/null 2>&1 &
```

Kiểm tra tiến trình Python:

``` bash
ps aux | grep '[p]ython3 -m http.server 8082'
```

Kết quả thực tế cần ghi nhận:

``` text
root      <PYTHON_PID>  ... python3 -m http.server 8082
```

Trong đó `<PYTHON_PID>` là PID của tiến trình Python đang chiếm cổng
8082.

## 4. Chẩn đoán cổng 8082

Kiểm tra bằng `ss`:

``` bash
sudo ss -tlnp | grep 8082
```

Kết quả thực tế dự kiến:

``` text
LISTEN 0 5 0.0.0.0:8082 0.0.0.0:* users:(("python3",pid=<PYTHON_PID>,fd=3))
```

Có thể kiểm tra thêm bằng `lsof`:

``` bash
sudo lsof -i :8082
```

Kết quả dự kiến:

``` text
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
python3  <PYTHON_PID> root  3u  IPv4  ...      TCP *:8082 (LISTEN)
```

### PID xác định được

``` text
PID: <PYTHON_PID>
Process: python3 -m http.server 8082
Port: 8082
Status: LISTEN
```

## 5. Kiểm tra thông tin tiến trình bằng ps

Sau khi xác định PID:

``` bash
ps -p <PYTHON_PID> -f
```

Kết quả dự kiến:

``` text
UID        PID  PPID  C STIME TTY          TIME CMD
root   <PYTHON_PID>     1  0 ... ?        00:00:00 python3 -m http.server 8082
```

Lệnh này dùng để xác nhận chính xác tiến trình cần kết thúc trước khi
thực hiện `kill`.

## 6. Giải phóng cổng bằng kill

Ưu tiên kết thúc tiến trình một cách bình thường:

``` bash
sudo kill <PYTHON_PID>
```

Sau đó kiểm tra lại:

``` bash
sudo ss -tlnp | grep 8082
```

Nếu tiến trình vẫn còn tồn tại, sử dụng:

``` bash
sudo kill -9 <PYTHON_PID>
```

Kiểm tra lại:

``` bash
sudo ss -tlnp | grep 8082
```

Nếu không còn kết quả, cổng 8082 đã được giải phóng.

## 7. Khởi động lại Spring Boot

Sau khi giải phóng cổng:

``` bash
sudo systemctl restart spring-app.service
```

Kiểm tra trạng thái dịch vụ:

``` bash
sudo systemctl status spring-app.service
```

Kết quả mong đợi:

``` text
● spring-app.service - Spring Boot Application
     Loaded: loaded (...)
     Active: active (running)
```

## 8. Xác nhận Java đang quản lý cổng 8082

Thực hiện:

``` bash
sudo ss -tlnp | grep 8082
```

Kết quả mong đợi tương tự:

``` text
LISTEN 0 100 0.0.0.0:8082 0.0.0.0:* users:(("java",pid=<JAVA_PID>,fd=...))
```

Kiểm tra tiến trình Java:

``` bash
ps -p <JAVA_PID> -f
```

Kết quả cần thể hiện ứng dụng Java đang chạy `app.jar`, ví dụ:

``` text
UID              PID  PPID  C STIME TTY          TIME CMD
spring-runner  <JAVA_PID>    1  0 ... ?        00:00:10 /usr/bin/java -jar /opt/spring-app/app.jar
```

Như vậy cổng `8082` đã được giải phóng khỏi tiến trình Python và được
chuyển cho ứng dụng Spring Boot quản lý.

## 9. Tổng hợp quy trình chẩn đoán

Quy trình thực hiện:

``` text
Python chiếm port 8082
        ↓
sudo ss -tlnp | grep 8082
        ↓
Xác định PID của python3
        ↓
ps -p <PID> -f
        ↓
sudo kill <PID>
        ↓
Kiểm tra lại port 8082
        ↓
sudo systemctl restart spring-app.service
        ↓
sudo ss -tlnp | grep 8082
        ↓
Java/app.jar LISTEN trên port 8082
```

## 10. Kết luận

Nguyên nhân của lỗi `Address already in use` là cổng `8082` đang bị một
tiến trình khác sử dụng.

Bằng `ss` hoặc `lsof`, có thể xác định PID và tên tiến trình đang lắng
nghe trên cổng. Sau khi xác nhận đúng tiến trình, sử dụng `kill` để kết
thúc tiến trình và giải phóng cổng. Nếu `kill` thông thường không hiệu
quả thì mới sử dụng `kill -9`.

Sau khi cổng được giải phóng, `spring-app.service` được khởi động lại
thành công và `ss -tlnp | grep 8082` xác nhận tiến trình `java` đang
LISTEN trên cổng 8082.

## 11. Minh chứng nên chụp

Nên chụp 3 ảnh minh chứng:

1.  **Trước khi xử lý:** kết quả

    ``` bash
    sudo ss -tlnp | grep 8082
    ```

    thể hiện `python3` và PID đang chiếm cổng.

2.  **Sau khi kill:** kết quả

    ``` bash
    sudo ss -tlnp | grep 8082
    ```

    không còn tiến trình Python chiếm cổng.

3.  **Sau khi khởi động Spring Boot:** kết quả

    ``` bash
    sudo systemctl status spring-app.service
    sudo ss -tlnp | grep 8082
    ```

    thể hiện service `active (running)` và `java` đang LISTEN trên 8082.

> Lưu ý: Các giá trị `<PYTHON_PID>` và `<JAVA_PID>` phải được thay bằng
> PID thực tế trên VPS. Không nên tự bịa PID trong bài nộp nếu giảng
> viên yêu cầu minh chứng từ máy thực tế.
