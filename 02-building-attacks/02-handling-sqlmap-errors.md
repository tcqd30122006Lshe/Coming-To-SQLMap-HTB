# Xử Lý Lỗi Trong Quá Trình Chạy SQLMap (Handling SQLMap Errors)

> **HTB Academy Module 58: SQLMap Essentials - Section 5 / 11**  
> *Đường dẫn bài học: `02-building-attacks/02-handling-sqlmap-errors.md`*

---

## 1. Tầm Quan Trọng Của Việc Gỡ Lỗi (Debugging SQLMap)

Trong quá trình pentest tự động bằng SQLMap, có rất nhiều trường hợp công cụ báo không tìm thấy điểm tiêm (`all tested parameters do not appear to be injectable`), dù bạn biết chắc chắn trang web có lỗi SQLi. 

Nguyên nhân phổ biến thường do:
* Cú pháp payload bị vướng dấu đóng mở ngoặc `)`, `'`, `"`.
* Ứng dụng web chặn User-Agent hoặc thiếu Cookie xác thực.
* Server phản hồi mã lỗi HTTP 500 Internal Server Error hoặc 403 Forbidden mà SQLMap không hiểu nguyên do.

Để chẩn đoán chính xác nguyên nhân gốc rễ, SQLMap cung cấp **4 cơ chế gỡ lỗi (debugging mechanisms) chuyên sâu**.

---

## 2. Hiển Thị Thông Báo Lỗi CSDL Với Cờ `--parse-errors`

Bình thường, SQLMap sẽ ẩn các lỗi database thô để giữ màn hình gọn gàng. Khi bật cờ **`--parse-errors`**, SQLMap sẽ chủ động bóc tách (parse) và in thẳng toàn bộ thông báo lỗi từ DBMS ra màn hình terminal:

```bash
sqlmap -u "http://www.target.com/vuln.php?id=1" --parse-errors --batch
```

### Đầu Ra Mẫu (Terminal Output):
```console
[16:09:20] [INFO] testing if GET parameter 'id' is dynamic
[16:09:20] [INFO] GET parameter 'id' appears to be dynamic
[16:09:20] [WARNING] parsed DBMS error message: 'SQLSTATE[42000]: Syntax error or access violation: 1064 You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '))"',),)((' at line 1'
[16:09:20] [INFO] heuristic (basic) test shows that GET parameter 'id' might be injectable (possible DBMS: 'MySQL')
[16:09:20] [WARNING] parsed DBMS error message: 'SQLSTATE[42000]: Syntax error or access violation: 1064 You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''YzDZJELylInm' at line 1'
```

> [!TIP]
> **Giá trị thực chiến của `--parse-errors`**:  
> Nhìn vào đoạn lỗi: `syntax to use near '))"',),)((' at line 1`, bạn sẽ biết ngay câu truy vấn SQL gốc bên trong backend được bao bọc bởi bao nhiêu dấu ngoặc đơn `)` và nháy đơn `'`. Từ đó bạn có thể chỉ định chính xác tham số `--prefix` và `--suffix` để câu lệnh SQL hợp lệ 100%!

---

## 3. Ghi Lại Toàn Bộ Lưu Lượng Vào File Bằng Cờ `-t` (Store Traffic)

Cờ **`-t <file_path>`** chỉ thị cho SQLMap ghi lại toàn bộ nhật ký các HTTP Request gửi đi và HTTP Response nhận về vào một file văn bản:

```bash
sqlmap -u "http://www.target.com/vuln.php?id=1" --batch -t /tmp/traffic.txt
```

### Kiểm Tra File Lưu Trữ Traffic:
```console
BreakHerHeart@htb[/htb]$ cat /tmp/traffic.txt
HTTP request [#1]:
GET /?id=1 HTTP/1.1
Host: www.example.com
Cache-control: no-cache
Accept-encoding: gzip,deflate
Accept: */*
User-agent: sqlmap/1.4.9 (http://sqlmap.org)
Connection: close

HTTP response [#1] (200 OK):
Date: Thu, 24 Sep 2020 14:12:50 GMT
Server: Apache/2.4.41 (Ubuntu)
Vary: Accept-Encoding
Content-Encoding: gzip
Content-Length: 914
Connection: close
Content-Type: text/html; charset=UTF-8
URI: http://www.example.com:80/?id=1

<!DOCTYPE html>
<html lang="en">
...SNIP...
```

* **Ứng dụng**: Cho phép pentester mở file text để `grep`, tìm kiếm mã trạng thái HTTP, xem response headers, hoặc kiểm tra xem payload gửi lên có bị biến dạng hay không.

---

## 4. Điều Chỉnh Mức Độ Chi Tiết Console Bằng Cờ `-v` (Verbosity Levels)

Cờ **`-v`** kiểm soát lượng thông tin in ra màn hình terminal theo 7 cấp độ (từ 0 đến 6):

| Cấp Độ (`-v`) | Tên Mức Độ | Ý Nghĩa Kỹ Thuật & Dữ Liệu Hiển Thị |
| :---: | :--- | :--- |
| **`0`** | Silent / Critical | Chỉ hiển thị thông báo lỗi nghiêm trọng (Python tracebacks, Critical errors). |
| **`1`** | Information (Mặc định) | Hiển thị các thông tin cơ bản (`INFO`), cảnh báo (`WARNING`) và kết quả cuối cùng. |
| **`2`** | Debug | Hiển thị thêm các dòng Debug nội bộ của chương trình. |
| **`3`** | **Payloads** | **Hiển thị chính xác chuỗi payload SQLi đang được tiêm vào** (Rất hữu ích để xem SQLMap đang thử câu lệnh gì). |
| **`4`** | HTTP Request Headers | Hiển thị toàn bộ HTTP Request Headers gửi đi. |
| **`5`** | HTTP Response Headers| Hiển thị toàn bộ HTTP Response Headers nhận về từ server. |
| **`6`** | **Full Traffic (Tối Đa)**| **In toàn bộ raw HTTP Request & Response body theo thời gian thực trực tiếp trên terminal**. |

### Chạy SQLMap Ở Chế Độ Chi Tiết Nhất (`-v 6`):
```bash
sqlmap -u "http://www.target.com/vuln.php?id=1" -v 6 --batch
```

```console
[*] starting @ 16:17:40 /2020-09-24/

[16:17:40] [DEBUG] cleaning up configuration parameters
[16:17:40] [DEBUG] setting the HTTP timeout
[16:17:40] [DEBUG] setting the HTTP User-Agent header
[16:17:40] [DEBUG] creating HTTP requests opener object
[16:17:40] [DEBUG] resolving hostname 'www.example.com'
[16:17:40] [INFO] testing connection to the target URL
[16:17:40] [TRAFFIC OUT] HTTP request [#1]:
GET /?id=1 HTTP/1.1
Host: www.example.com
Cache-control: no-cache
Accept-encoding: gzip,deflate
Accept: */*
User-agent: sqlmap/1.4.9 (http://sqlmap.org)
Connection: close

[16:17:40] [DEBUG] declared web page charset 'utf-8'
[16:17:40] [TRAFFIC IN] HTTP response [#1] (200 OK):
Date: Thu, 24 Sep 2020 14:17:40 GMT
Server: Apache/2.4.41 (Ubuntu)
Vary: Accept-Encoding
Content-Encoding: gzip
Content-Length: 914
Connection: close
Content-Type: text/html; charset=UTF-8
URI: http://www.example.com:80/?id=1

<!DOCTYPE html>
<html lang="en">
...SNIP...
```

---

## 5. Chuyển Hướng Lưu Lượng Qua Proxy Bằng Cờ `--proxy` (Burp Suite Integration)

Phương pháp mạnh mẽ và trực quan nhất để bắt trọn mọi hành vi của SQLMap là đẩy toàn bộ lưu lượng qua một Man-in-the-Middle (MitM) Proxy như **Burp Suite** hoặc **OWASP ZAP**:

```bash
sqlmap -u "http://www.target.com/vuln.php?id=1" --proxy="http://127.0.0.1:8080" --batch
```

### Lợi Ích Của Việc Kết Hợp SQLMap + Burp Suite:
1. **Lịch sử trực quan (HTTP History)**: Mọi request do SQLMap tạo ra đều hiển thị rõ ràng từng dòng trong tab **Proxy $\rightarrow$ HTTP history** của Burp Suite kèm theo mã HTTP (200, 302, 403, 500) và độ dài response length.
2. **Kiểm thử thủ công linh hoạt (Repeater)**: Bạn có thể chọn bất kỳ request nào của SQLMap gửi sang **Burp Repeater** (`Ctrl + R`) để tự tay sửa đổi tham số và quan sát phản hồi.
3. **Phát hiện WAF chặn**: Nếu thấy hàng loạt request bị server trả về `403 Forbidden` hoặc trang captcha của Cloudflare, bạn biết ngay mục tiêu có WAF và cần chuyển sang dùng các kỹ thuật Bypass (Tamper Scripts).

---

## 6. Bảng Tra Cứu Nhanh Các Cờ Gỡ Lỗi (Debugging Cheatsheet)

| Cờ / Tham Số | Mục Đích Gỡ Lỗi | Lợi Ích Chính |
| :--- | :--- | :--- |
| **`--parse-errors`** | Bóc tách lỗi cú pháp DBMS | Xem lỗi chi tiết để chỉnh prefix/suffix chính xác |
| **`-t <file>`** | Ghi lưu lượng ra tệp tin | Lưu giữ bằng chứng (evidence) và phân tích offline |
| **`-v 3`** | Xem payload đang tiêm | Biết SQLMap đang gửi payload gì lên máy chủ |
| **`-v 6`** | Xem toàn bộ traffic trên terminal | Theo dõi trọn vẹn request/response real-time |
| **`--proxy="http://127.0.0.1:8080"`** | Đẩy traffic qua Burp Suite | Phân tích đồ họa trực quan và tận dụng sức mạnh Burp |

---
*Edit by me and AI.*
