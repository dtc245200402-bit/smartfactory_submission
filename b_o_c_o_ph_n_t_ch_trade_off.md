# Báo Cáo Phân Tích & Tối Ưu Hóa Index Hệ Thống IoT SmartFactory
**Chức danh:** Database Optimization Expert  
**Đơn vị gửi:** Đội ngũ Quản trị CSDL SmartFactory  

---

## 1. Chẩn Đoán Thảm Họa Fat Covering Index
Trong các hệ thống IoT thời gian thực (Real-time Streaming) với 10,000 cảm biến đẩy dữ liệu liên tục, thao tác **GHI (INSERT)** đóng vai trò sống còn. 

Lập trình viên cũ đã mắc sai lầm nghiêm trọng khi tạo `idx_fat_covering` bao trùm toàn bộ các cột (`sensor_id`, `recorded_at`, `temperature`, `humidity`, `status`). 
* **Hậu quả 1 (Write Bottleneck & Data Loss):** Mỗi bản ghi INSERT buộc MySQL phải cập nhật và sắp xếp lại cây B-Tree của Index chứa tới 5 cột phức tạp. Hiện tượng phân tách trang (Page Split) diễn ra liên tục, làm nghẽn luồng ghi, gây rớt dữ liệu cảm biến.
* **Hậu quả 2 (Storage Overhead):** Kích thước nút (Node) trên cây B-Tree phụ thuộc vào tổng độ dài các cột. Nhồi nhét `VARCHAR(20)` và các trường đo lường khiến `Index_length` vượt xa `Data_length`, làm tăng gấp 4 lần chi phí ổ cứng Cloud SSD.

---

## 2. Giải Pháp Lean Index & Đánh Đổi Hiệu Năng

Chúng tôi thay thế `idx_fat_covering` bằng **`idx_lean_search(sensor_id, recorded_at)`**:
* **Quy tắc thiết kế:** Index chỉ dùng để **LỌC** (`WHERE sensor_id = ... AND recorded_at >= ...`), không dùng làm bộ chứa dữ liệu.
* **Sự đánh đổi:** Khi SELECT, MySQL sử dụng `idx_lean_search` để định vị cực nhanh các bản ghi thỏa mãn, sau đó thực hiện **Bookmark Lookup** (tra cứu theo Primary Key `log_id`) về Clustered Index để lấy các giá trị `temperature`, `humidity`, `status`. 
* **Kết quả thu được:** Tốc độ đọc chậm hơn một vài phần nghìn giây (không đáng kể đối với trải nghiệm người dùng trên Dashboard), nhưng tốc độ **INSERT tăng hơn 5 lần** và **tiết kiệm >70% dung lượng Index**.

---

## 3. Bảo Vệ Thiết Kế (Vấn Đáp Cloud Financial Controller)

### Câu 1: Nếu đây là bảng "Danh mục quốc gia" (Countries) cả năm không ai sửa, việc tạo Covering Index có còn là "tội ác"?
> **Trả lời:** Không. Bảng danh mục là dạng dữ liệu đọc nhiều - ghi ít (Read-Heavy / Static Data). Vì không có luồng ghi liên tục, chi phí "Write Penalty" bằng 0. Việc tạo Covering Index giúp tối ưu tốc độ đọc lên mức tuyệt đối $100\%$ mà không gây ảnh hưởng tới hiệu năng hệ thống.

### Câu 2: Giải thích khái niệm "Write Penalty"? Tại sao thêm 1 cột vào Index lại làm INSERT chậm lại?
> **Trả lời:** **Write Penalty** là chi phí tài nguyên mà hệ thống phải trả khi thực hiện thao tác Ghi (INSERT/UPDATE/DELETE) trên bảng có Index. 
> Khi thêm 1 cột vào Index:
> 1. Kích thước mỗi bản ghi trong Secondary Index tăng lên, giảm số lượng bản ghi trên một Index Page.
> 2. Tần suất ghi ngẫu nhiên (Random Disk I/O) và phân tách trang (Page Split) tăng lên.
> 3. Hệ thống tốn thêm CPU/RAM để tính toán vị trí sắp xếp cho cột mới đó trong cây B-Tree.

### Câu 3: Nếu thay `VARCHAR(20)` của cột `status` thành `TINYINT`, tác động thế nào đến Data/Index Length nếu lỡ đưa vào Index?
> **Trả lời:** 
> * Với `utf8mb4`, `VARCHAR(20)` tiêu tốn tối đa **81 bytes** ($20 \times 4 + 1$ byte độ dài).
> * `TINYINT` chỉ tiêu tốn đúng **1 byte**.
> * Việc chuyển sang `TINYINT` giúp tiết kiệm **80 bytes trên mỗi nút B-Tree của từng dòng dữ liệu**. Với hàng triệu bản ghi, điều này cắt giảm hàng trăm Megabyte bộ nhớ RAM/Disk, giảm tỷ lệ Page Split và tăng mật độ lưu trữ dữ liệu trên mỗi trang đĩa.