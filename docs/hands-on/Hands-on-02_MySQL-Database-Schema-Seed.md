# Hands-on 02: Xây dựng cơ sở dữ liệu MySQL và dữ liệu mẫu

## 1. Mục tiêu

Sau khi hoàn thành Hands-on 02, người học có thể:

- Bổ sung MySQL vào dự án PHP bằng Docker Compose.
- Hiểu mô hình nhiều service `web` và `db`.
- Quản lý cấu hình bằng `.env`, `.env.example` và `.gitignore`.
- Sử dụng named volume để lưu dữ liệu MySQL.
- Chuyển mô hình dữ liệu thành các bảng quan hệ bằng `schema.sql`.
- Sử dụng khóa chính, khóa ngoại và các ràng buộc dữ liệu cơ bản.
- Chuẩn hóa dữ liệu tiếng Việt với `utf8mb4`.
- Quản lý nhiều hình ảnh cho một sản phẩm.
- Tạo dữ liệu mẫu bằng `seed.sql`.
- Cho MySQL tự động chạy `schema.sql` và `seed.sql` khi khởi tạo.
- Kiểm tra cấu trúc, dữ liệu và commit/push checkpoint lên GitHub.

> **Phạm vi Hands-on 02:** dừng ở việc hoàn thành cơ sở dữ liệu và nạp dữ liệu mẫu. Kết nối PHP với MySQL bằng MySQLi sẽ thực hiện ở Hands-on tiếp theo.

---

## 2. Kết quả cần đạt

```text
php-mysql-sales/
├── database/
│   ├── schema.sql
│   └── seed.sql
├── docker/
│   └── php/
│       └── Dockerfile
├── public/
│   └── index.php
├── src/
│   ├── config/
│   └── includes/
├── .env
├── .env.example
├── .gitignore
├── compose.yaml
└── README.md
```

MySQL có 9 bảng:

```text
categories
suppliers
customers
employees
shippers
products
product_images
orders
orderdetail
```

---

# 3. Khởi động lại dự án

## Mục đích

Kiểm tra source code và khởi động lại môi trường Docker sau khi mở máy.

## Lệnh thực thi

```cmd
cd /d D:\PTUDW-ST-2026\php-mysql-sales
git status
docker compose up -d
docker compose ps
```

## Kết quả mong đợi

Service `web` hoạt động và `http://localhost:8080` truy cập bình thường.

## Giải thích

- `git status`: kiểm tra trạng thái mã nguồn.
- `docker compose up -d`: tạo/khởi động service và chạy nền.
- `docker compose ps`: xem trạng thái container.

## Checkpoint

- [ ] Đúng thư mục dự án.
- [ ] Git hoạt động bình thường.
- [ ] `web` đang Up.
- [ ] `http://localhost:8080` hoạt động.

---

# 4. Cấu hình môi trường MySQL

## Mục đích

Tách thông tin cấu hình khỏi `compose.yaml` để dễ thay đổi giữa các môi trường.

Tạo `.env`:

```env
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=ql_banhang
MYSQL_USER=appuser
MYSQL_PASSWORD=apppassword
```

Cập nhật `.env.example` với các biến tương ứng:

```env
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=ql_banhang
MYSQL_USER=appuser
MYSQL_PASSWORD=apppassword
```

## Giải thích

| Biến | Ý nghĩa |
|---|---|
| `MYSQL_ROOT_PASSWORD` | Mật khẩu tài khoản quản trị MySQL |
| `MYSQL_DATABASE` | Database được tạo khi MySQL khởi tạo |
| `MYSQL_USER` | Tài khoản dành cho ứng dụng |
| `MYSQL_PASSWORD` | Mật khẩu của tài khoản ứng dụng |

`.env` là cấu hình cục bộ, còn `.env.example` là mẫu cấu hình được đưa lên GitHub.

---

# 5. Cập nhật `.gitignore`

## Nội dung cần thêm

```gitignore
# Local environment configuration
.env
```

## Giải thích

```text
.env         → cấu hình riêng → không commit
.env.example → mẫu cấu hình  → được commit
```

Trong các hands-on sau, `.gitignore` chỉ được bổ sung khi thực tế phát sinh file/thư mục cần bỏ qua.

---

# 6. Cấu hình MySQL và named volume trong `compose.yaml`

```yaml
services:
  web:
    build:
      context: .
      dockerfile: docker/php/Dockerfile
    ports:
      - "8080:80"
    volumes:
      - ./public:/var/www/html

  db:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./database/schema.sql:/docker-entrypoint-initdb.d/01-schema.sql:ro
      - ./database/seed.sql:/docker-entrypoint-initdb.d/02-seed.sql:ro

volumes:
  mysql_data:
```

> Nếu `schema.sql` và `seed.sql` chưa được tạo tại thời điểm cấu hình service `db`, bổ sung hai dòng mount tương ứng sau khi tạo hai file này.

## Giải thích `${...}`

```yaml
MYSQL_DATABASE: ${MYSQL_DATABASE}
```

Docker Compose lấy giá trị từ `.env`:

```text
.env → compose.yaml → MySQL container
```

## Giải thích named volume

```yaml
- mysql_data:/var/lib/mysql
```

`/var/lib/mysql` là nơi MySQL lưu dữ liệu. `mysql_data` là named volume được Docker quản lý bên ngoài vòng đời container.

Hai dòng cuối:

```yaml
volumes:
  mysql_data:
```

khai báo named volume ở cấp Compose project.

### Phân biệt hai lệnh

```cmd
docker compose down
```

Xóa container/network nhưng giữ volume.

```cmd
docker compose down -v
```

Xóa cả named volume.

> **Cảnh báo:** `-v` làm mất dữ liệu trong volume. Chỉ dùng khi chủ động muốn khởi tạo lại database từ đầu.

---

# 7. Tạo `database/schema.sql`

## Mục đích

`schema.sql` lưu cấu trúc CSDL dưới dạng mã nguồn để database có thể được tái tạo trên máy khác.

## Lệnh thực thi

Nếu cần tạo thư mục:

```cmd
mkdir database
```

Tạo file:

```cmd
type nul > database\schema.sql
```

## Nội dung `schema.sql`

> Lưu file bằng **UTF-8**.

```sql
SET NAMES utf8mb4;

CREATE TABLE categories (
    CategoryID INT AUTO_INCREMENT PRIMARY KEY,
    CategoryName VARCHAR(200) NOT NULL,
    Description TEXT
) CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE TABLE suppliers (
    SupplierID INT AUTO_INCREMENT PRIMARY KEY,
    SupplierName VARCHAR(200) NOT NULL,
    ContactName VARCHAR(100),
    Address VARCHAR(200),
    City VARCHAR(100),
    PostalCode VARCHAR(20),
    Country VARCHAR(100),
    Phone VARCHAR(20)
) CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE TABLE customers (
    CustomerID INT AUTO_INCREMENT PRIMARY KEY,
    CustomerName VARCHAR(100) NOT NULL,
    ContactName VARCHAR(100),
    Address VARCHAR(200),
    City VARCHAR(100),
    PostalCode VARCHAR(20),
    Country VARCHAR(100)
) CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE TABLE employees (
    EmployeeID INT AUTO_INCREMENT PRIMARY KEY,
    LastName VARCHAR(50) NOT NULL,
    FirstName VARCHAR(50) NOT NULL,
    BirthDate DATE,
    Photo VARCHAR(255),
    Notes TEXT
) CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE TABLE shippers (
    ShipperID INT AUTO_INCREMENT PRIMARY KEY,
    ShipperName VARCHAR(100) NOT NULL,
    Phone VARCHAR(20)
) CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE TABLE products (
    ProductID INT AUTO_INCREMENT PRIMARY KEY,
    ProductCode VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(255) NOT NULL,
    Description TEXT,
    Unit VARCHAR(20),
    Price DECIMAL(12,2) NOT NULL DEFAULT 0,
    StockQuantity INT NOT NULL DEFAULT 0,
    IsActive BOOLEAN NOT NULL DEFAULT TRUE,
    SupplierID INT,
    CategoryID INT,

    CONSTRAINT chk_products_price CHECK (Price >= 0),
    CONSTRAINT chk_products_stock CHECK (StockQuantity >= 0),
    CONSTRAINT fk_products_supplier
        FOREIGN KEY (SupplierID) REFERENCES suppliers(SupplierID),
    CONSTRAINT fk_products_category
        FOREIGN KEY (CategoryID) REFERENCES categories(CategoryID)
) CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE TABLE product_images (
    ProductImageID INT AUTO_INCREMENT PRIMARY KEY,
    ProductID INT NOT NULL,
    ImageFile VARCHAR(255) NOT NULL,
    AltText VARCHAR(255),
    IsPrimary BOOLEAN NOT NULL DEFAULT FALSE,
    SortOrder INT NOT NULL DEFAULT 0,

    CONSTRAINT uq_product_image UNIQUE (ProductID, ImageFile),
    CONSTRAINT chk_product_image_sort CHECK (SortOrder >= 0),
    CONSTRAINT fk_product_images_product
        FOREIGN KEY (ProductID)
        REFERENCES products(ProductID)
        ON DELETE CASCADE
) CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE TABLE orders (
    OrderID INT AUTO_INCREMENT PRIMARY KEY,
    OrderDate DATE NOT NULL,
    CustomerID INT,
    EmployeeID INT,
    ShipperID INT,

    CONSTRAINT fk_orders_customer
        FOREIGN KEY (CustomerID) REFERENCES customers(CustomerID),
    CONSTRAINT fk_orders_employee
        FOREIGN KEY (EmployeeID) REFERENCES employees(EmployeeID),
    CONSTRAINT fk_orders_shipper
        FOREIGN KEY (ShipperID) REFERENCES shippers(ShipperID)
) CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE TABLE orderdetail (
    OrderDetailID INT AUTO_INCREMENT PRIMARY KEY,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(12,2) NOT NULL,
    OrderID INT NOT NULL,
    ProductID INT NOT NULL,

    CONSTRAINT chk_orderdetail_quantity CHECK (Quantity > 0),
    CONSTRAINT chk_orderdetail_unitprice CHECK (UnitPrice >= 0),
    CONSTRAINT fk_orderdetail_order
        FOREIGN KEY (OrderID)
        REFERENCES orders(OrderID)
        ON DELETE CASCADE,
    CONSTRAINT fk_orderdetail_product
        FOREIGN KEY (ProductID) REFERENCES products(ProductID)
) CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

---

# 8. Giải thích thiết kế database

Các bảng độc lập được tạo trước (`categories`, `suppliers`, `customers`, `employees`, `shippers`), sau đó mới đến các bảng có khóa ngoại (`products`, `product_images`, `orders`, `orderdetail`).

`ProductID` là khóa kỹ thuật, còn `ProductCode` là mã nghiệp vụ như `SP001`. `Price` và `UnitPrice` dùng `DECIMAL(12,2)` vì dữ liệu tiền cần độ chính xác thập phân ổn định. `IsActive` cho phép ngừng kinh doanh sản phẩm mà không bắt buộc xóa lịch sử. `product_images` tạo quan hệ một-nhiều để một sản phẩm có nhiều ảnh; database chỉ lưu tên file, ví dụ `phone-a-1.jpg`, còn file thật dự kiến nằm trong `public/uploads/products/`. `orderdetail.UnitPrice` giữ giá tại thời điểm mua để đơn hàng cũ không bị ảnh hưởng khi `products.Price` thay đổi.

Các `CHECK` và `UNIQUE` giúp database bảo vệ một số quy tắc dữ liệu như giá không âm, tồn kho không âm, số lượng mua lớn hơn 0 và không lặp cùng một file ảnh cho cùng sản phẩm.

---

# 9. Tạo `database/seed.sql`

## Mục đích

```text
schema.sql → cấu trúc
seed.sql   → dữ liệu mẫu
```

## Lệnh thực thi

```cmd
type nul > database\seed.sql
```

## Nội dung `seed.sql`

> Lưu file bằng **UTF-8**.

```sql
SET NAMES utf8mb4;

INSERT INTO categories (CategoryName, Description)
VALUES
    ('Điện thoại', 'Các sản phẩm điện thoại di động'),
    ('Máy tính', 'Máy tính xách tay và máy tính để bàn'),
    ('Phụ kiện', 'Phụ kiện dành cho thiết bị công nghệ');

INSERT INTO suppliers
    (SupplierName, ContactName, Address, City, PostalCode, Country, Phone)
VALUES
    ('Công ty Công nghệ ABC', 'Nguyễn Văn An',
     '01 Nguyễn Huệ', 'TP. Hồ Chí Minh', '700000', 'Việt Nam', '0901234567'),
    ('Công ty Thiết bị XYZ', 'Trần Minh Bình',
     '25 Lê Lợi', 'Đà Nẵng', '550000', 'Việt Nam', '0912345678');

INSERT INTO customers
    (CustomerName, ContactName, Address, City, PostalCode, Country)
VALUES
    ('Nguyễn Minh Anh', 'Nguyễn Minh Anh',
     '12 Nguyễn Trãi', 'TP. Hồ Chí Minh', '700000', 'Việt Nam'),
    ('Trần Thanh Hà', 'Trần Thanh Hà',
     '35 Hùng Vương', 'Cần Thơ', '900000', 'Việt Nam'),
    ('Lê Quốc Bảo', 'Lê Quốc Bảo',
     '18 Lê Lợi', 'Trà Vinh', '870000', 'Việt Nam');

INSERT INTO employees
    (LastName, FirstName, BirthDate, Photo, Notes)
VALUES
    ('Nguyễn', 'Hoàng Nam', '1995-05-15', NULL, 'Nhân viên bán hàng'),
    ('Trần', 'Thúy An', '1997-09-20', NULL, 'Nhân viên chăm sóc khách hàng');

INSERT INTO shippers (ShipperName, Phone)
VALUES
    ('Giao hàng nhanh', '19001001'),
    ('Giao hàng tiết kiệm', '19001002');

INSERT INTO products
    (ProductCode, ProductName, Description, Unit, Price,
     StockQuantity, IsActive, SupplierID, CategoryID)
VALUES
    ('SP001', 'Điện thoại Smartphone A',
     'Điện thoại thông minh màn hình lớn, phù hợp nhu cầu sử dụng hằng ngày.',
     'Chiếc', 8500000.00, 20, TRUE, 1, 1),
    ('SP002', 'Laptop B',
     'Máy tính xách tay phục vụ học tập và làm việc.',
     'Chiếc', 18500000.00, 10, TRUE, 2, 2),
    ('SP003', 'Chuột không dây C',
     'Chuột không dây nhỏ gọn, kết nối ổn định.',
     'Chiếc', 450000.00, 50, TRUE, 2, 3);

INSERT INTO product_images
    (ProductID, ImageFile, AltText, IsPrimary, SortOrder)
VALUES
    (1, 'phone-a-1.jpg', 'Điện thoại Smartphone A - ảnh chính', TRUE, 1),
    (1, 'phone-a-2.jpg', 'Điện thoại Smartphone A - mặt sau', FALSE, 2),
    (2, 'laptop-b-1.jpg', 'Laptop B - ảnh chính', TRUE, 1),
    (3, 'mouse-c-1.jpg', 'Chuột không dây C - ảnh chính', TRUE, 1);

INSERT INTO orders (OrderDate, CustomerID, EmployeeID, ShipperID)
VALUES
    ('2026-08-10', 1, 1, 1),
    ('2026-08-11', 2, 2, 2),
    ('2026-08-12', 3, 1, 1);

INSERT INTO orderdetail (Quantity, UnitPrice, OrderID, ProductID)
VALUES
    (1, 8500000.00, 1, 1),
    (2, 450000.00, 1, 3),
    (1, 18500000.00, 2, 2),
    (2, 450000.00, 3, 3);
```

Dữ liệu bảng cha phải được thêm trước bảng con để thỏa mãn khóa ngoại.

---

# 10. Tự động khởi tạo database

Trong `compose.yaml`, service `db` phải có:

```yaml
volumes:
  - mysql_data:/var/lib/mysql
  - ./database/schema.sql:/docker-entrypoint-initdb.d/01-schema.sql:ro
  - ./database/seed.sql:/docker-entrypoint-initdb.d/02-seed.sql:ro
```

`01-schema.sql` chạy trước để tạo bảng; `02-seed.sql` chạy sau để thêm dữ liệu. `:ro` nghĩa là container chỉ được đọc file mount.

Các init script chỉ chạy khi MySQL khởi tạo data directory mới. Vì vậy nếu volume cũ vẫn tồn tại, restart container sẽ không tự chạy lại hai file này.

---

# 11. Khởi tạo lại database từ đầu

## Mục đích

Kiểm chứng database có thể được tạo hoàn toàn tự động từ các file trong project.

## Lệnh thực thi

> **Cảnh báo:** lệnh đầu tiên xóa toàn bộ dữ liệu trong named volume MySQL.

```cmd
docker compose down -v
docker compose up -d
docker compose ps
```

## Kết quả mong đợi

```text
web   Up
db    Up
```

Pipeline:

```text
.env
 ↓
compose.yaml
 ↓
MySQL
 ↓
ql_banhang
 ↓
01-schema.sql → 9 bảng
 ↓
02-seed.sql   → dữ liệu mẫu
```

Nếu cần kiểm tra lỗi:

```cmd
docker compose logs db
```

---

# 12. Kiểm tra database và dữ liệu

## Lệnh thực thi trên Windows CMD

```cmd
chcp 65001
docker compose exec db mysql --default-character-set=utf8mb4 -u root -p
```

Nhập mật khẩu `MYSQL_ROOT_PASSWORD` trong `.env`.

## Lệnh kiểm tra trong MySQL

```sql
USE ql_banhang;
SHOW TABLES;

SELECT * FROM categories;
SELECT * FROM products;
SELECT * FROM product_images;
SELECT * FROM orders;
SELECT * FROM orderdetail;
```

Kiểm tra cấu trúc:

```sql
DESCRIBE products;
SHOW CREATE TABLE products;
```

Kiểm tra số lượng:

```sql
SELECT COUNT(*) FROM categories;
SELECT COUNT(*) FROM suppliers;
SELECT COUNT(*) FROM products;
SELECT COUNT(*) FROM product_images;
```

---

# 13. Kiểm tra tổng tiền đơn hàng

```sql
SELECT
    o.OrderID,
    o.OrderDate,
    SUM(od.Quantity * od.UnitPrice) AS TotalAmount
FROM orders o
JOIN orderdetail od
    ON o.OrderID = od.OrderID
GROUP BY
    o.OrderID,
    o.OrderDate;
```

```text
Thành tiền = Quantity × UnitPrice
Tổng đơn hàng = tổng thành tiền các dòng của cùng OrderID
```

`TotalAmount` chưa cần lưu trong `orders` vì có thể tính từ dữ liệu chi tiết.

---

# 14. Xử lý sự cố tiếng Việt

Nếu MySQL CLI hiển thị ký tự như `?i?n tho?i`, chưa thể kết luận dữ liệu trong database bị hỏng.

Kiểm tra dữ liệu thực:

```sql
SELECT ProductID, ProductName, HEX(ProductName)
FROM products;
```

Kiểm tra charset:

```sql
SHOW VARIABLES LIKE 'character_set%';
```

Thoát MySQL và kết nối lại:

```sql
exit;
```

```cmd
chcp 65001
docker compose exec db mysql --default-character-set=utf8mb4 -u root -p
```

Sau đó:

```sql
USE ql_banhang;
SELECT ProductID, ProductName, Description, Unit
FROM products;
```

> `schema.sql` và `seed.sql` phải được lưu bằng UTF-8. `utf8mb4` được sử dụng để hỗ trợ đầy đủ dữ liệu tiếng Việt.

---

# 15. Checkpoint cuối Hands-on 02

```text
[ ] web đang Up
[ ] db đang Up
[ ] ql_banhang được tạo tự động
[ ] schema.sql tạo đủ 9 bảng
[ ] seed.sql nạp dữ liệu thành công
[ ] khóa chính/khóa ngoại hoạt động
[ ] products có ProductCode, StockQuantity và IsActive
[ ] product_images hỗ trợ nhiều ảnh/sản phẩm
[ ] orderdetail có UnitPrice
[ ] dữ liệu tiếng Việt được lưu đúng
[ ] mysql_data hoạt động
[ ] http://localhost:8080 vẫn hoạt động
```

---

# 16. Commit và push checkpoint HO02

## Kiểm tra

```cmd
git status
```

Các thay đổi chính dự kiến:

```text
modified:   .env.example
modified:   .gitignore
modified:   compose.yaml
new file:   database/schema.sql
new file:   database/seed.sql
```

`.env` không được đưa vào commit.

## Lệnh thực thi

```cmd
git add .
git status
git commit -m "Add MySQL database schema and sample data"
git push
git status
git log -1 --oneline
```

## Kết quả mong đợi

```text
nothing to commit, working tree clean
```

Checkpoint thử nghiệm khi xây dựng Hands-on 02:

```text
571115c Add MySQL database schema and sample data
```

Hash commit của người học sẽ khác.

---

# 17. Kiến thức đã sử dụng

**Docker và môi trường phát triển:** Docker Compose nhiều service; environment variables; `.env`; `.env.example`; `.gitignore`; named volume; bind mount; MySQL initialization scripts.

**Cơ sở dữ liệu:** bảng; khóa chính; khóa ngoại; quan hệ một-nhiều; `CREATE TABLE`; `INSERT`; `SELECT`; `JOIN`; `CHECK`; `UNIQUE`; `DECIMAL`; `utf8mb4`.

**Thiết kế ứng dụng:** tách schema và seed; quản lý nhiều ảnh bằng bảng riêng; chỉ lưu tên file ảnh; phân biệt `ProductID` và `ProductCode`; giữ giá lịch sử bằng `UnitPrice`; sử dụng `IsActive` thay cho việc bắt buộc xóa vật lý sản phẩm.

**Git/GitHub:** kiểm tra thay đổi; ignore file; stage; commit; push; checkpoint theo hands-on.

---

# 18. Tổng kết

Sau Hands-on 02, database có thể được tái tạo từ các file của project:

```text
Project source
     │
     ├── .env
     ├── compose.yaml
     ├── schema.sql
     └── seed.sql
             │
             ▼
       Docker Compose
             │
             ▼
           MySQL
             │
             ▼
       ql_banhang
             │
        ┌────┴─────┐
        ▼          ▼
     9 bảng    dữ liệu mẫu
```

Hands-on tiếp theo sẽ bắt đầu kết nối ứng dụng PHP với database bằng **MySQLi** và sử dụng dữ liệu thật trong giao diện Web.
