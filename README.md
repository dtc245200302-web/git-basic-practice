# Bài thực hành: View, Index, Store Procedure

## 📌 Mục tiêu
Luyện tập sử dụng View, Index và Store Procedure trên bảng dữ liệu.

## 🛠️ Công nghệ sử dụng
- MySQL 8.0
- MySQL Workbench

## 📂 Nội dung bài làm
| File | Mô tả |
|------|-------|
| `01_create_database.sql` | Tạo database `demo` |
| `02_create_table_products.sql` | Tạo bảng `Products` + dữ liệu mẫu |
| `03_index_and_explain.sql` | Tạo Unique Index, Composite Index, so sánh EXPLAIN |
| `04_views.sql` | Tạo / sửa / xoá View |
| `05_store_procedures.sql` | 4 Store Procedure: lấy tất cả, thêm, sửa, xoá |

## 📊 Kết quả đạt được

### 1. Index
- **Trước khi tạo Index:** `EXPLAIN` cho thấy `type=ALL`, `rows=7` (quét toàn bộ bảng)
- **Sau khi tạo Index:** `type=const/ref`, `rows=1` (chỉ quét dòng cần thiết)
- → Tốc độ truy vấn được cải thiện rõ rệt

### 2. View
- Tạo được view `v_product_info` hiển thị 4 cột: `productCode`, `productName`, `productPrice`, `productStatus`
- Sửa view thành công (thêm cột `productAmount`)
- Xoá view thành công

### 3. Store Procedure
- ✅ `sp_get_all_products` — lấy tất cả sản phẩm
- ✅ `sp_add_product` — thêm sản phẩm mới
- ✅ `sp_update_product` — sửa sản phẩm theo Id
- ✅ `sp_delete_product` — xoá sản phẩm theo Id

## 👤 Tác giả
- Họ tên: [Tên của bạn]
- Lớp: [Tên lớp]
- GitHub: [@dtc245200302-web](https://github.com/dtc245200302-web)
- -- ============================================
-- Bước 1: Tạo cơ sở dữ liệu demo
-- ============================================

DROP DATABASE IF EXISTS demo;
CREATE DATABASE demo
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE demo;

SELECT 'Database demo created successfully!' AS message;
-- ============================================
-- Bước 2: Tạo bảng Products + dữ liệu mẫu
-- ============================================

USE demo;

DROP TABLE IF EXISTS Products;

CREATE TABLE Products (
    Id                  INT AUTO_INCREMENT PRIMARY KEY,
    productCode         VARCHAR(20)  NOT NULL,
    productName         VARCHAR(100) NOT NULL,
    productPrice        DECIMAL(10,2) NOT NULL,
    productAmount       INT NOT NULL,
    productDescription  TEXT,
    productStatus       VARCHAR(20) DEFAULT 'active'
);

-- Chèn dữ liệu mẫu
INSERT INTO Products 
    (productCode, productName, productPrice, productAmount, productDescription, productStatus) 
VALUES
    ('SP001', 'Laptop Dell XPS',      25000000, 10, 'Laptop cao cấp cho lập trình viên', 'active'),
    ('SP002', 'iPhone 15 Pro',        30000000,  5, 'Điện thoại flagship Apple',         'active'),
    ('SP003', 'Chuột Logitech',         500000, 50, 'Chuột không dây',                   'active'),
    ('SP004', 'Bàn phím cơ',           1500000, 20, 'Bàn phím cơ RGB',                   'inactive'),
    ('SP005', 'Màn hình LG 27 inch',   5000000, 15, 'Màn hình 2K',                       'active'),
    ('SP006', 'Tai nghe Sony',         3000000,  8, 'Tai nghe chống ồn',                 'active'),
    ('SP007', 'Laptop Asus',          18000000, 12, 'Laptop văn phòng',                  'active');

-- Kiểm tra
SELECT * FROM Products;
-- ============================================
-- Bước 3: Tạo Index + EXPLAIN + So sánh
-- ============================================

USE demo;

-- --------------------------------------------
-- 3.1. TRƯỚC KHI TẠO INDEX
-- --------------------------------------------
SELECT '=== TRƯỚC KHI TẠO INDEX ===' AS step;

EXPLAIN SELECT * FROM Products WHERE productCode = 'SP002';

EXPLAIN SELECT * FROM Products 
WHERE productName = 'Laptop Dell XPS' AND productPrice = 25000000;

-- Nhận xét: type = ALL, rows = 7 (quét toàn bộ bảng)


-- --------------------------------------------
-- 3.2. TẠO UNIQUE INDEX trên productCode
-- --------------------------------------------
CREATE UNIQUE INDEX idx_product_code ON Products(productCode);

-- --------------------------------------------
-- 3.3. TẠO COMPOSITE INDEX trên (productName, productPrice)
-- --------------------------------------------
CREATE INDEX idx_name_price ON Products(productName, productPrice);


-- --------------------------------------------
-- 3.4. SAU KHI TẠO INDEX
-- --------------------------------------------
SELECT '=== SAU KHI TẠO INDEX ===' AS step;

EXPLAIN SELECT * FROM Products WHERE productCode = 'SP002';

EXPLAIN SELECT * FROM Products 
WHERE productName = 'Laptop Dell XPS' AND productPrice = 25000000;

-- Nhận xét: type = const/ref, key = idx_..., rows = 1
-- → Truy vấn nhanh hơn nhờ tận dụng index


-- --------------------------------------------
-- 3.5. Xem danh sách index trên bảng
-- --------------------------------------------
SHOW INDEX FROM Products;
-- ============================================
-- Bước 4: View
-- ============================================

USE demo;

-- --------------------------------------------
-- 4.1. Tạo View
-- --------------------------------------------
CREATE VIEW v_product_info AS
SELECT 
    productCode, 
    productName, 
    productPrice, 
    productStatus
FROM Products;

-- Xem dữ liệu
SELECT * FROM v_product_info;


-- --------------------------------------------
-- 4.2. Sửa View (thêm cột productAmount)
-- --------------------------------------------
CREATE OR REPLACE VIEW v_product_info AS
SELECT 
    productCode, 
    productName, 
    productPrice, 
    productAmount,
    productStatus
FROM Products;

-- Kiểm tra lại
SELECT * FROM v_product_info;


-- --------------------------------------------
-- 4.3. Xoá View
-- --------------------------------------------
DROP VIEW v_product_info;

-- Kiểm tra đã xoá
SHOW FULL TABLES WHERE Table_type = 'VIEW';
# 1. Tạo thư mục
mkdir bai-tap-sql
cd bai-tap-sql

# 2. Copy 6 file trên vào thư mục
# 3. Tạo thư mục screenshots và copy ảnh vào

# 4. Khởi tạo Git
git init
git add .
git commit -m "Hoan thanh bai thuc hanh View Index Store Procedure"
git branch -M main

# 5. Tạo repo trên GitHub tên: bai-tap-sql (Public, KHÔNG tích README)

# 6. Kết nối + push
git remote add origin https://github.com/dtc245200302-web/bai-tap-sql.git
git push -u origin main
