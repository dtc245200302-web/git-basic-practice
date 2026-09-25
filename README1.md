-- ============================================================
-- SMARTFACTORY REINDEX SCRIPT
-- Tác giả: Database Optimization Expert
-- Mục tiêu: Gỡ bỏ "Fat Index", thay bằng "Lean Index"
-- ============================================================

USE smartfactory_db;

-- ------------------------------------------------------------
-- BƯỚC 1: Đo lường dung lượng Index TRƯỚC khi tối ưu
-- ------------------------------------------------------------
SELECT 
    TABLE_NAME,
    ROUND(DATA_LENGTH / 1024 / 1024, 2)  AS data_size_MB,
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_size_MB,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_MB
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'smartfactory_db' 
  AND TABLE_NAME = 'SensorLogs';

-- Xem chi tiết các index hiện có
SHOW INDEX FROM SensorLogs;


-- ------------------------------------------------------------
-- BƯỚC 2: Phân tích query plan TRƯỚC khi tối ưu
-- ------------------------------------------------------------
SELECT '=== EXPLAIN TRƯỚC KHI TỐI ƯU ===' AS step;

EXPLAIN SELECT temperature, humidity, status 
FROM SensorLogs
WHERE sensor_id = 105 
  AND recorded_at >= '2026-06-20';

-- Nhận xét: 
--   key = idx_fat_covering
--   Extra = "Using index" (Covering Index → đọc thẳng trên cây Index)
--   → SELECT cực nhanh NHƯNG INSERT rất chậm


-- ------------------------------------------------------------
-- BƯỚC 3: XÓA "FAT INDEX" khổng lồ
-- ------------------------------------------------------------
ALTER TABLE SensorLogs DROP INDEX idx_fat_covering;

SELECT 'Fat Index dropped!' AS message;


-- ------------------------------------------------------------
-- BƯỚC 4: TẠO "LEAN INDEX" tinh gọn
-- Chỉ giữ 2 cột dùng trong WHERE + ORDER BY
-- ------------------------------------------------------------
CREATE INDEX idx_lean_search ON SensorLogs(sensor_id, recorded_at);

SELECT 'Lean Index created!' AS message;


-- ------------------------------------------------------------
-- BƯỚC 5: Phân tích query plan SAU khi tối ưu
-- ------------------------------------------------------------
SELECT '=== EXPLAIN SAU KHI TỐI ƯU ===' AS step;

EXPLAIN SELECT temperature, humidity, status 
FROM SensorLogs
WHERE sensor_id = 105 
  AND recorded_at >= '2026-06-20';

-- Nhận xét:
--   key = idx_lean_search  ✅ (MySQL dùng Index mới)
--   type = range           ✅ (vẫn tối ưu)
--   Extra = "Using where"  (KHÔNG còn "Using index" 
--          → MySQL phải lookup vào table để lấy 
--            temperature, humidity, status)
--   → Truy vấn chậm hơn vài phần nghìn giây NHƯNG:
--     + INSERT nhanh hơn 5 lần
--     + Dung lượng Index giảm ~70%


-- ------------------------------------------------------------
-- BƯỚC 6: Đo lường dung lượng SAU khi tối ưu
-- ------------------------------------------------------------
SELECT 
    TABLE_NAME,
    ROUND(DATA_LENGTH / 1024 / 1024, 2)  AS data_size_MB,
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_size_MB,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_MB
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'smartfactory_db' 
  AND TABLE_NAME = 'SensorLogs';

SHOW INDEX FROM SensorLogs;
# Báo cáo Đánh đổi Index — SmartFactory

## 1. Vấn đề của "Fat Index"

Index cũ `idx_fat_covering` chứa **5 cột**: `(sensor_id, recorded_at, temperature, humidity, status)`.

**Hệ quả:**
- **Write Penalty nặng:** Mỗi lần INSERT, MySQL phải sắp xếp lại cây B-Tree trên cả 5 cột. Vì `temperature`, `humidity`, `status` **thay đổi liên tục** → cây Index bị xáo trộn liên tục → INSERT chậm, Data Pipeline không theo kịp tốc độ cảm biến.
- **Storage phình to:** Cột `status VARCHAR(20)` chiếm 20 byte + `DECIMAL(5,2)` chiếm 5 byte × 2 cột + con trỏ → mỗi entry Index tốn hàng chục byte. Với hàng triệu bản ghi/ngày → Index còn **lớn hơn cả bảng gốc** → hóa đơn AWS tăng 4 lần.

## 2. Giải pháp "Lean Index"

Index mới `idx_lean_search` chỉ chứa **2 cột**: `(sensor_id, recorded_at)`.

**Nguyên tắc:** Index sinh ra để **LỌC** và **SẮP XẾP**, không phải để **LƯU TRỮ**.
- `sensor_id` → dùng trong `WHERE`
- `recorded_at` → dùng trong `WHERE` + `ORDER BY`

## 3. Đánh đổi chấp nhận được

| Tiêu chí | Fat Index cũ | Lean Index mới |
|----------|:------------:|:--------------:|
| Số cột | 5 | **2** |
| SELECT | ⚡ Siêu nhanh (Covering) | ⚡ Nhanh (cần lookup table) |
| INSERT | 🐢 Rất chậm | 🚀 **Nhanh gấp ~5 lần** |
| Dung lượng Index | 💾 Lớn | 💾 **Giảm ~70%** |
| Hóa đơn Cloud | 💸 Cao | 💰 **Tiết kiệm** |

**Kết luận:** Chấp nhận SELECT chậm hơn **vài phần nghìn giây** (do phải lookup table) để đổi lấy:
- ✅ INSERT nhanh hơn 5 lần → không rớt dữ liệu
- ✅ Dung lượng Index giảm 70% → hóa đơn cloud giảm mạnh
- ✅ Hệ thống vẫn dùng Index (`type = range`) → truy vấn vẫn tối ưu

## 4. Trả lời câu hỏi vấn đáp

### Q1: Bảng "Countries" (ít sửa) có nên dùng Covering Index?
**CÓ.** Vì bảng Countries:
- **Hiếm khi INSERT/UPDATE** → Write Penalty không đáng kể
- **Đọc nhiều** → Covering Index giúp SELECT siêu nhanh
- → Trade-off **hoàn toàn hợp lý** trong bối cảnh read-heavy

→ **Covering Index không phải "tội ác" tuyệt đối** — nó chỉ tệ trong **write-heavy** (như IoT).

### Q2: Write Penalty là gì? Tại sao thêm cột vào Index làm INSERT chậm?
**Write Penalty** = chi phí phụ trội khi INSERT/UPDATE/DELETE vì phải cập nhật cả Index.

Mỗi cột thêm vào Index:
- Tăng **kích thước entry** → nhiều Data Page hơn
- Tăng **số phép so sánh** khi chèn vào cây B-Tree → chậm hơn
- Nếu cột đó **hay thay đổi** → cây Index phải **tái cấu trúc liên tục**

→ **Càng nhiều cột → INSERT càng chậm** (theo cấp số nhân, không phải cộng).

### Q3: Đổi `status VARCHAR(20)` → `TINYINT` tác động thế nào?
- `VARCHAR(20)` = **20 byte** (hoặc hơn nếu utf8mb4) cho mỗi giá trị
- `TINYINT` = **1 byte**
- **Tiết kiệm 19 byte/entry** × hàng triệu bản ghi = tiết kiệm **hàng chục MB**
- Nếu đưa vào Index → Index nhỏ hơn → INSERT nhanh hơn, Storage giảm

→ **Đây là best practice** cho cột có ít giá trị (enum-like): dùng `TINYINT` thay `VARCHAR`.
# AI Prompt Log — Tìm hiểu Clustered & Secondary Index

## Prompt 1: Clustered Index là gì?

**Hỏi AI:**
> Giải thích Clustered Index trong MySQL InnoDB là gì? Nó khác gì Secondary Index?

**AI trả lời (tóm tắt):**
- **Clustered Index** = cây B-Tree chứa **toàn bộ dữ liệu** của bảng, khóa là **PRIMARY KEY**.
- Trong InnoDB, **mọi bảng đều có Clustered Index** (mặc định là PK).
- **Secondary Index** = cây B-Tree riêng, **leaf node chứa PK** (không chứa toàn bộ dữ liệu).
- Khi query dùng Secondary Index → MySQL tìm trong Secondary Index → lấy PK → **lookup vào Clustered Index** để lấy các cột còn lại.

**Ứng dụng vào bài:**
- `log_id` là PK → Clustered Index
- `idx_fat_covering` và `idx_lean_search` là Secondary Index
- Vì Fat Index chứa đủ cột → không cần lookup Clustered → "Using index"
- Lean Index thiếu cột → phải lookup Clustered → "Using where"

---

## Prompt 2: Byte calculation cho các kiểu dữ liệu

**Hỏi AI:**
> Tính dung lượng byte của các kiểu dữ liệu sau trong MySQL: BIGINT, INT, DATETIME, DECIMAL(5,2), VARCHAR(20).

**AI trả lời:**

| Kiểu | Byte |
|------|:----:|
| BIGINT | 8 |
| INT | 4 |
| DATETIME | 8 |
| DECIMAL(5,2) | 5 |
| VARCHAR(20) utf8mb4 | 20–80 |

**Ứng dụng vào bài — tính dung lượng entry Index:**

**Fat Index** `(sensor_id, recorded_at, temperature, humidity, status)`:
- INT (4) + DATETIME (8) + DECIMAL (5) + DECIMAL (5) + VARCHAR(20) (20)
- = **42 byte/entry** (chưa tính overhead)

**Lean Index** `(sensor_id, recorded_at)`:
- INT (4) + DATETIME (8) = **12 byte/entry**

→ **Giảm ~71%** dung lượng mỗi entry!

Với 100 triệu bản ghi:
- Fat Index: ~4.2 GB
- Lean Index: ~1.2 GB
- **Tiết kiệm ~3 GB** → hóa đơn AWS giảm đáng kể.

---

## Prompt 3: Write Penalty chi tiết

**Hỏi AI:**
> Tại sao thêm cột vào Index lại làm INSERT chậm hơn? Giải thích cơ chế B-Tree.

**AI trả lời (tóm tắt):**
1. Mỗi INSERT → phải chèn 1 entry mới vào **mọi Index** của bảng
2. Cây B-Tree có **thứ tự** → phải tìm vị trí chèn → so sánh nhiều lần
3. Nếu Data Page đầy → **page split** → phân mảnh, chậm
4. Càng nhiều cột → entry càng lớn → càng ít entry/page → **page split sớm hơn**
5. Cột **hay thay đổi** → UPDATE cũng phải sửa Index → Write Penalty càng nặng

**Ứng dụng:** Fat Index có 5 cột → mỗi INSERT phải chèn entry 42 byte → page split liên tục → Data Pipeline rớt.

---

## Kết luận từ AI

Sau khi trao đổi, tôi rút ra:
1. **Covering Index** tốt cho **read-heavy**, tệ cho **write-heavy**
2. **Clustered Index** = nơi lưu dữ liệu thật; **Secondary Index** chỉ chứa PK
3. **Write Penalty** tăng theo **số cột** và **tần suất thay đổi** của cột
4. **TINYINT** thay `VARCHAR` cho cột enum → tiết kiệm dung lượng
5. **Lean Index** = chỉ chứa cột WHERE + ORDER BY → cân bằng Read/Write/Storage
