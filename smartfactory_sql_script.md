-- ========================================================
-- BÀI THỰC HÀNH: SMARTFACTORY - TỐI ƯU HÓA INDEX IOT
-- File: smartfactory_reindex.sql
-- ========================================================

-- Bước 1: Khởi tạo CSDL và Bảng mẫu
CREATE DATABASE IF NOT EXISTS smartfactory_db;
USE smartfactory_db;

DROP TABLE IF EXISTS SensorLogs;
CREATE TABLE SensorLogs (
    log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    sensor_id INT NOT NULL,
    recorded_at DATETIME NOT NULL,
    temperature DECIMAL(5,2),
    humidity DECIMAL(5,2),
    status VARCHAR(20) -- 'NORMAL', 'WARNING', 'CRITICAL'
);

-- ========================================================
-- TRẠNG THÁI BAN ĐẦU: "FAT COVERING INDEX"
-- ========================================================
CREATE INDEX idx_fat_covering ON SensorLogs(sensor_id, recorded_at, temperature, humidity, status);

-- Kiểm tra dung lượng khi dùng Fat Index
SELECT 
    table_name AS 'Table',
    ROUND(((data_length) / 1024 / 1024), 2) AS 'Data_Size_MB',
    ROUND(((index_length) / 1024 / 1024), 2) AS 'Index_Size_MB',
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Total_Size_MB'
FROM information_schema.TABLES
WHERE table_schema = 'smartfactory_db' AND table_name = 'SensorLogs';

-- Kiểm tra kế hoạch thực thi EXPLAIN với Fat Index
EXPLAIN SELECT temperature, humidity, status 
FROM SensorLogs 
WHERE sensor_id = 105 AND recorded_at >= '2026-06-20';
-- Ghi nhận cột Extra: "Using index" (Covering Index - không cần đọc bảng gốc)

-- ========================================================
-- BƯỚC 2: CHUYỂN ĐỔI SANG LEAN INDEX (TỐI ƯU)
-- ========================================================

-- Xóa Fat Index phình to dung lượng và gây nghẽn luồng INSERT
ALTER TABLE SensorLogs DROP INDEX idx_fat_covering;

-- Tạo Lean Index chỉ chứa các cột dùng để lọc (WHERE / ORDER BY)
CREATE INDEX idx_lean_search ON SensorLogs(sensor_id, recorded_at);

-- ========================================================
-- BƯỚC 3: ĐỐI CHIẾU KẾT QUẢ SAU KHI TỐI ƯU
-- ========================================================

-- Kiểm tra lại kế hoạch thực thi EXPLAIN với Lean Index
EXPLAIN SELECT temperature, humidity, status 
FROM SensorLogs 
WHERE sensor_id = 105 AND recorded_at >= '2026-06-20';
-- Ghi nhận: 
-- 1. Key sử dụng: idx_lean_search
-- 2. Cột Extra: Không còn "Using index" (phải thực hiện Bookmark Lookup vào Clustered Index để lấy temperature, humidity, status)
-- 3. Cột type: range (vẫn truy vấn theo khoảng thời gian vô cùng nhanh chóng)

-- Kiểm tra dung lượng bộ nhớ Index đã giải phóng
SELECT 
    table_name AS 'Table',
    ROUND(((data_length) / 1024 / 1024), 2) AS 'Data_Size_MB',
    ROUND(((index_length) / 1024 / 1024), 2) AS 'Index_Size_MB',
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Total_Size_MB'
FROM information_schema.TABLES
WHERE table_schema = 'smartfactory_db' AND table_name = 'SensorLogs';