# Bài tập: Log Rotation với RollingFileAppender (StoreX)

**Ngày thực hiện:** 08/11/2026
**Dự án:** StoreX (Spring Boot 3.3, Java 17), sửa trên `order-service`
**Môi trường chạy:** AWS EC2 (Ubuntu), Docker Compose

---

## 1. Cấu hình gốc và vấn đề

### Tình huống

Server VPS của StoreX bị sập vì Disk 100%. Nguyên nhân là file `application.log` phình lên 40GB sau 3 tháng chạy.

### Cấu hình gốc

```xml
<appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>logs/application.log</file>
    <encoder>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
    </encoder>
</appender>
```

### Vấn đề

- `FileAppender` chỉ ghi nối tiếp vào một file duy nhất, không có cơ chế cắt hay xóa file cũ. File tăng mãi cho đến khi hết đĩa.
- Không giới hạn kích thước, không giới hạn thời gian lưu trữ.
- Khi đã phình lên hàng chục GB thì việc đọc, tìm kiếm, sao chép file cũng rất khó.

### Lỗi phát hiện thêm trong cấu hình gốc

Pattern `%c{1.}` của đề **không hợp lệ trong Logback** (đây là cú pháp rút gọn tên logger của Log4j2). Khi chạy thật, log khởi động báo:

```text
|-ERROR in ch.qos.logback.classic.pattern.LoggerConverter@5cbf9e9f - failed to parse integer string [1.]
java.lang.NumberFormatException: For input string: "1."
```

Encoder của FILE appender không khởi động được. Cách sửa là dùng `%logger{36}` (tên logger rút gọn tối đa 36 ký tự), giống pattern console.

---

## 2. Yêu cầu và cách thực hiện

### Yêu cầu

Đổi sang `RollingFileAppender`, xoay vòng theo ngày, mỗi file tối đa 10MB, lưu tối đa 30 ngày. Tên file có dạng `application-2026-07-10.0.log`.

### Bước 1. Sửa `logback-spring.xml` của order-service

```xml
<!-- Kích thước tối đa mỗi file, mặc định 10MB theo đề -->
<springProperty scope="context" name="LOG_MAX_FILE_SIZE" source="storex.log.max-file-size" defaultValue="10MB"/>

<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/application.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>logs/application-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
        <maxFileSize>${LOG_MAX_FILE_SIZE}</maxFileSize>
        <maxHistory>30</maxHistory>
        <totalSizeCap>3GB</totalSizeCap>
    </rollingPolicy>
    <encoder>
        <pattern>%d %p %logger{36} [%t] %m%n</pattern>
    </encoder>
</appender>

<root level="INFO">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="LOGSTASH"/>
    <appender-ref ref="FILE"/>
</root>
```

| Thành phần | Tác dụng |
|---|---|
| `RollingFileAppender` | Thay `FileAppender`, cho phép đổi tên và tạo file mới khi đến điều kiện |
| `SizeAndTimeBasedRollingPolicy` | Cắt theo cả ngày (`%d`) lẫn kích thước (`%i`) |
| `fileNamePattern` | `%d{yyyy-MM-dd}` là ngày, `%i` là số thứ tự file trong ngày (bắt đầu từ 0) |
| `maxFileSize` 10MB | File vượt ngưỡng thì cắt, tạo file mới |
| `maxHistory` 30 | Giữ 30 ngày, file cũ hơn tự bị xóa |
| `totalSizeCap` 3GB | Lưới an toàn do mình thêm: tổng dung lượng log không vượt 3GB, kể cả khi một ngày sinh nhiều file |

### Bước 2. Mount thư mục log ra ngoài container (`docker-compose.yml`)

```yaml
  order-service:
    volumes:
      - ./logs/order-service:/app/logs
```

Để xem file log trực tiếp trên EC2 (`~/storex/logs/order-service`).

### Bước 3. Kiểm chứng

Vì cần ghi tới 10MB mới thấy cắt, khi test mình hạ ngưỡng xuống 50KB (đổi `defaultValue` thành `50KB`), build lại, bắn 100 request checkout rồi xem thư mục log. Sau khi test, đổi lại `10MB` và build lại.

```bash
docker compose up -d --build order-service
python3 scripts/load_checkout.py 100
ls -lh logs/order-service
```

---

## 3. Kết quả

### Kết quả test với ngưỡng 50KB

```text
total 520K
-rw-r--r-- 1 root root 349K Nov  8 15:08 application-2026-11-08.0.log
-rw-r--r-- 1 root root  51K Nov  8 15:08 application-2026-11-08.1.log
-rw-r--r-- 1 root root  62K Nov  8 15:08 application-2026-11-08.2.log
-rw-r--r-- 1 root root  48K Nov  8 15:08 application.log
```

Log được cắt thành nhiều file đúng mẫu `application-<ngày>.<số thứ tự>.log`, file đang ghi vẫn là `application.log`.

Giải thích các kích thước:

- **`.0` nặng 349K:** đây là file `application.log` cũ của lần test trước (298K) chưa được xóa, bị đổi tên thành `.0` khi có dòng log đầu tiên, cộng với khoảng 50KB log mới. Đây là dữ liệu cũ nên không phản ánh giới hạn.
- **`.1` nặng 51K, `.2` nặng 62K:** hơi vượt ngưỡng 50KB vì Logback chỉ kiểm tra kích thước sau mỗi nhóm dòng log và chỉ cắt khi file đã vượt, không cắt chính xác từng byte.

### Cấu hình cuối cùng (10MB theo đề)

```text
5:  <springProperty ... name="LOG_MAX_FILE_SIZE" source="storex.log.max-file-size" defaultValue="10MB"/>
33: <fileNamePattern>logs/application-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
34: <maxFileSize>${LOG_MAX_FILE_SIZE}</maxFileSize>
35: <maxHistory>30</maxHistory>
36: <totalSizeCap>3GB</totalSizeCap>
```

Sau khi đổi lại 10MB và build lại, bắn 100 request, thư mục log có một file `application.log` nặng 210K và dòng đầu đã theo pattern mới (chưa cắt vì chưa tới 10MB):

```text
total 212K
-rw-r--r-- 1 root root 210K Nov  8 15:11 application.log

2026-11-08 15:10:52,935 INFO o.h.validator.internal.util.Version [background-preinit] HV000001: Hibernate Validator 8.0.1.Final
2026-11-08 15:10:53,190 INFO c.s.order.OrderServiceApplication [main] Starting OrderServiceApplication v0.0.1-SNAPSHOT using Java 17.0.21 ...
```

Dòng log theo pattern `%d %p %logger{36} [%t] %m%n` (ngày giờ, mức, logger, thread, nội dung) và không còn lỗi `NumberFormatException`.

### So sánh trước và sau

| | Trước (`FileAppender`) | Sau (`RollingFileAppender`) |
|---|---|---|
| Số file | Một file duy nhất | Nhiều file theo ngày và theo số thứ tự |
| Kích thước | Không giới hạn (40GB sau 3 tháng) | Mỗi file tối đa 10MB |
| Lưu trữ | Mãi mãi | Tối đa 30 ngày, tổng tối đa 3GB |
| Nguy cơ đầy đĩa | Cao | Có chặn trên rõ ràng |

### Nhận xét và lưu ý

- **Đạt yêu cầu:** log được cắt thành nhiều file tên `application-<ngày>.<n>.log`, kích thước và thời gian lưu đều có giới hạn.
- **Phần cắt theo ngày chưa được kiểm chứng trực tiếp**, vì phải đợi qua nửa đêm. Bằng chứng thực tế chỉ có cho phần cắt theo kích thước; phần theo ngày dựa vào mẫu tên file có `%d` và cơ chế của Logback.
- **Múi giờ:** container chạy giờ UTC nên log ghi 15:10 ứng với 22:10 giờ Việt Nam, và thời điểm cắt theo ngày là 00:00 UTC (7 giờ sáng giờ Việt Nam). Muốn cắt theo ngày Việt Nam cần đặt múi giờ cho container (`TZ=Asia/Ho_Chi_Minh`).
- **Truyền ngưỡng bằng biến môi trường chưa chạy được trong lần test:** khi đặt `STORX_LOG_MAX_FILE_SIZE=50KB` cho container, file 298K vẫn không bị cắt, chỉ khi ghi thẳng 50KB vào `defaultValue` của XML thì mới cắt. Mình chưa xác định được nguyên nhân, nên cấu hình cuối không dựa vào biến môi trường này (dòng đó trong `docker-compose.yml` hiện không có tác dụng đã kiểm chứng).
- **Ghi log ra file trong container:** vì thư mục `logs/` được mount ra host nên file thuộc user `root`; muốn xóa từ host cần `sudo`.
- **Chỉ áp dụng cho order-service.** user-service và payment-service vẫn chưa ghi file; muốn áp dụng chỉ cần chép cùng khối cấu hình và thêm volume tương ứng.
