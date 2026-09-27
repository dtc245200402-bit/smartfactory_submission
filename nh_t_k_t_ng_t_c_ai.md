# Nhật Ký Tương Tác AI (AI Prompt Log) - SmartFactory Project

---

### Prompt 1: Tìm hiểu cơ chế Clustered Index vs Secondary Index trong InnoDB
* **Người hỏi:** "Giải thích cách MySQL InnoDB lưu trữ dữ liệu trên Clustered Index (Primary Key) và Secondary Index. Tại sao Covering Index lại tránh được bước Bookmark Lookup?"
* **AI trả lời tóm tắt:**
  * **Clustered Index:** Chứa toàn bộ dữ liệu thực tế của bảng tại các lá (Leaf nodes) của cây B-Tree, sắp xếp theo Primary Key.
  * **Secondary Index:** Các lá của cây B-Tree chỉ chứa các cột nằm trong Index + Giá trị Primary Key tương ứng.
  * **Covering Index:** Nếu tất cả các cột xuất hiện trong câu lệnh `SELECT` đều nằm sẵn trong Secondary Index, MySQL lấy ngay dữ liệu trên lá mà không cần tra ngược về Clustered Index (tránh Bookmark Lookup / Table Access).

---

### Prompt 2: Phân tích chi phí Write Penalty & Page Split khi INSERT
* **Người hỏi:** "Tại sao một Composite Index gồm 5 cột lại gây nghẽn cổ chai luồng INSERT nặng nề hơn Composite Index 2 cột trong hệ thống IoT lưu hàng triệu record?"
* **AI trả lời tóm tắt:**
  * Thao tác INSERT vào Secondary Index đòi hỏi tìm đúng vị trí con trỏ theo thứ tự lexicographical của cả 5 cột.
  * Kích thước bản ghi lớn khiến một trang dữ liệu 16KB (InnoDB Page) chứa được ít bản ghi hơn.
  * Khi trang bị đầy, MySQL phải thực hiện **Page Split** (chia đôi trang và ghi lại con trỏ), sinh ra đợt ghi đĩa ngẫu nhiên (Random I/O) vô cùng tốn kém CPU và làm đơ luồng pipeline.

---

### Prompt 3: Tính toán byte dữ liệu để tối ưu hóa bộ nhớ
* **Người hỏi:** "Hãy tính toán dung lượng chênh lệch bộ nhớ tính theo Byte giữa cột status kiểu VARCHAR(20) utf8mb4 và TINYINT khi nằm trên cây B-Tree của Secondary Index."
* **AI trả lời tóm tắt:**
  * `VARCHAR(20)` bảng mã `utf8mb4`: $20 \text{ ký tự} \times 4 \text{ bytes/ký tự} + 1 \text{ byte prefix} = 81 \text{ bytes}$.
  * `TINYINT`: $1 \text{ byte}$.
  * **Chênh lệch:** 80 bytes/row. Trên 10,000,000 dòng dữ liệu, việc đưa `TINYINT` vào Index tiết kiệm xấp xỉ **800MB dung lượng đĩa và RAM Buffer Pool**.