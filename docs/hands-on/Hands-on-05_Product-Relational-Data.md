# Hands-on 05 – Quản lý sản phẩm và dữ liệu quan hệ

## 1. Mục tiêu

Sau khi hoàn thành Hands-on 05, người học có thể:

- kiểm tra các quan hệ khóa chính – khóa ngoại trong cơ sở dữ liệu;
- phân biệt vai trò của `FOREIGN KEY` và `NOT NULL`;
- chuẩn hóa các khóa ngoại bắt buộc thành `NOT NULL`;
- kiểm tra dữ liệu hiện có trước khi áp dụng ràng buộc mới;
- kết nhiều bảng bằng quan hệ khóa chính – khóa ngoại;
- viết truy vấn bằng `FROM ... WHERE ... AND ...` và đối chiếu với `INNER JOIN ... ON`;
- xây dựng CRUD Product bằng PHP/MySQLi;
- tạo `<select>` động cho Category và Supplier;
- xử lý giá, tồn kho và trạng thái kinh doanh;
- kết hợp `product_images` để hiển thị ảnh chính của sản phẩm;
- hiểu nguyên tắc: file ảnh lưu trong hệ thống tệp, database chỉ lưu tên file ảnh.

> **Phạm vi HO05:** dừng ở việc hiển thị ảnh chính đã có. Upload ảnh và quản lý nhiều ảnh sẽ chuyển sang Hands-on tiếp theo.

---

## 2. Kiểm tra trạng thái project

### Lệnh thực thi

```cmd
cd /d D:\PTUDW-ST-2026\php-mysql-sales

git status
docker compose up -d
docker compose ps
```

### Kết quả mong đợi

```text
web   Up
db    Up
```

Trang `http://localhost:8080` vẫn hoạt động.

---

# 3. Kiểm tra toàn bộ khóa ngoại

Đăng nhập MySQL:

```cmd
docker compose exec db mysql --default-character-set=utf8mb4 -u root -p
```

Chọn database:

```sql
USE ql_banhang;
```

Liệt kê toàn bộ khóa ngoại:

```sql
SELECT
    TABLE_NAME,
    COLUMN_NAME,
    REFERENCED_TABLE_NAME,
    REFERENCED_COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'ql_banhang'
  AND REFERENCED_TABLE_NAME IS NOT NULL
ORDER BY TABLE_NAME, COLUMN_NAME;
```

Project có 8 khóa ngoại:

```text
orderdetail.OrderID      → orders.OrderID
orderdetail.ProductID    → products.ProductID
orders.CustomerID        → customers.CustomerID
orders.EmployeeID        → employees.EmployeeID
orders.ShipperID         → shippers.ShipperID
product_images.ProductID → products.ProductID
products.CategoryID      → categories.CategoryID
products.SupplierID      → suppliers.SupplierID
```

---

# 4. Kiểm tra khóa ngoại nào còn cho phép NULL

```sql
SELECT
    TABLE_NAME,
    COLUMN_NAME,
    IS_NULLABLE,
    COLUMN_TYPE
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'ql_banhang'
  AND (TABLE_NAME, COLUMN_NAME) IN (
      SELECT
          TABLE_NAME,
          COLUMN_NAME
      FROM information_schema.KEY_COLUMN_USAGE
      WHERE TABLE_SCHEMA = 'ql_banhang'
        AND REFERENCED_TABLE_NAME IS NOT NULL
  )
ORDER BY TABLE_NAME, COLUMN_NAME;
```

Cần chú ý:

```text
IS_NULLABLE = YES
→ cột hiện cho phép NULL

IS_NULLABLE = NO
→ cột không cho phép NULL
```

---

# 5. Hiểu `FOREIGN KEY` và `NOT NULL`

Ví dụ:

```sql
CategoryID INT NOT NULL
```

ngăn:

```text
CategoryID = NULL
```

Còn:

```sql
FOREIGN KEY (CategoryID)
REFERENCES categories(CategoryID)
```

ngăn:

```text
CategoryID = 999
```

nếu danh mục `999` không tồn tại.

Có thể ghi nhớ:

```text
NOT NULL
→ quan hệ bắt buộc phải có

FOREIGN KEY
→ giá trị tham chiếu phải tồn tại
```

---

# 6. Chuẩn hóa các khóa ngoại bắt buộc

Mở:

```text
database/schema.sql
```

## 6.1. Trong bảng `products`

Tìm:

```sql
SupplierID INT,
CategoryID INT,
```

chỉ thay thành:

```sql
SupplierID INT NOT NULL,
CategoryID INT NOT NULL,
```

## 6.2. Trong bảng `orders`

Tìm:

```sql
CustomerID INT,
EmployeeID INT,
ShipperID INT,
```

chỉ thay thành:

```sql
CustomerID INT NOT NULL,
EmployeeID INT NOT NULL,
ShipperID INT NOT NULL,
```

> Không thay toàn bộ file `schema.sql`. Chỉ sửa đúng các dòng cần thiết.

---

# 7. Kiểm tra dữ liệu hiện có trước khi reset database

```sql
SELECT
    'products.SupplierID' AS ForeignKey,
    COUNT(*) AS NullCount
FROM products
WHERE SupplierID IS NULL

UNION ALL

SELECT
    'products.CategoryID',
    COUNT(*)
FROM products
WHERE CategoryID IS NULL

UNION ALL

SELECT
    'orders.CustomerID',
    COUNT(*)
FROM orders
WHERE CustomerID IS NULL

UNION ALL

SELECT
    'orders.EmployeeID',
    COUNT(*)
FROM orders
WHERE EmployeeID IS NULL

UNION ALL

SELECT
    'orders.ShipperID',
    COUNT(*)
FROM orders
WHERE ShipperID IS NULL;
```

Kết quả mong đợi:

```text
products.SupplierID  0
products.CategoryID  0
orders.CustomerID    0
orders.EmployeeID    0
orders.ShipperID     0
```

Chỉ khi tất cả `NullCount = 0` mới tiếp tục.

---

# 8. Chuẩn bị ảnh mẫu trước khi chạy lại database

Đây là bước bắt buộc phải hoàn thành **trước khi chạy `docker compose down -v`**, để tên file thật và tên lưu trong `seed.sql` khớp nhau.

## 8.1. Tạo thư mục ảnh

Thoát MySQL:

```sql
exit;
```

Trong CMD:

```cmd
mkdir public\uploads
mkdir public\uploads\products
```

Cấu trúc:

```text
public/
└── uploads/
    └── products/
```

## 8.2. Tự tải hoặc tự chuẩn bị 4 ảnh minh họa

Người học tự tải hoặc tự tạo:

- 2 ảnh điện thoại;
- 1 ảnh laptop;
- 1 ảnh chuột không dây.

### Tên file ảnh mẫu đề xuất

Có thể đặt phần tên như sau:

```text
phone-a-1
phone-a-2
laptop-b-1
mouse-c-1
```

Ảnh tải về có thể có phần mở rộng như `.jpg`, `.jpeg`, `.png`, `.webp`,... và có thể giữ nguyên định dạng thực tế của file.

Ví dụ:

```text
public/uploads/products/
├── phone-a-1.png
├── phone-a-2.jpg
├── laptop-b-1.webp
└── mouse-c-1.jpeg
```

Đặt tất cả ảnh tại:

```text
public/uploads/products/
```

Kiểm tra tên file và phần mở rộng thực tế:

```cmd
dir public\uploads\products
```

> **Quan trọng:** tên file và phần mở rộng ghi trong trường `ImageFile` của `seed.sql` phải khớp chính xác với file thật trong `public/uploads/products/`. Không mặc định tất cả ảnh đều có cùng phần mở rộng.

## 8.3. Kiểm tra ảnh bằng trình duyệt

Ví dụ:

```text
http://localhost:8080/uploads/products/phone-a-1.webp
```

Ảnh phải hiển thị.

---

# 9. Cập nhật tên ảnh trong `seed.sql`

Mở:

```text
database/seed.sql
```

Tìm phần:

```sql
INSERT INTO product_images
```

Cập nhật `ImageFile` theo **đúng tên và phần mở rộng của file ảnh thực tế**.

Ví dụ, nếu các file đã tải về là:

```text
phone-a-1.png
phone-a-2.jpg
laptop-b-1.webp
mouse-c-1.jpeg
```

thì dữ liệu phải tương ứng:

```sql
INSERT INTO product_images
    (ProductID, ImageFile, AltText, IsPrimary, SortOrder)
VALUES
    (1, 'phone-a-1.png',
     'Điện thoại Smartphone A - ảnh chính', TRUE, 1),
    (1, 'phone-a-2.jpg',
     'Điện thoại Smartphone A - mặt sau', FALSE, 2),
    (2, 'laptop-b-1.webp',
     'Laptop B - ảnh chính', TRUE, 1),
    (3, 'mouse-c-1.jpeg',
     'Chuột không dây C - ảnh chính', TRUE, 1);
```

Nguyên tắc cần ghi nhớ:

```text
Tên file thật
        =
giá trị ImageFile trong database
```

Nếu file thật là `laptop-b-1.png` thì `ImageFile` cũng phải là `laptop-b-1.png`. Nếu tên hoặc phần mở rộng không khớp, trình duyệt sẽ tạo đường dẫn đến một file không tồn tại và ảnh sẽ không hiển thị.

---

# 10. Tạo lại database

Chỉ thực hiện khi:

```text
[✓] schema.sql đã sửa NOT NULL
[✓] không có khóa ngoại hiện tại chứa NULL
[✓] 4 ảnh đã được đặt đúng thư mục
[✓] seed.sql đã dùng đúng tên và phần mở rộng file ảnh
```

Lệnh:

```cmd
docker compose down -v
docker compose up -d
docker compose ps
```

`-v` xóa named volume cũ để MySQL chạy lại:

```text
01-schema.sql
      ↓
tạo schema mới

02-seed.sql
      ↓
nạp dữ liệu mẫu
```

---

# 11. Kiểm tra schema sau khi tạo lại

```cmd
docker compose exec db mysql --default-character-set=utf8mb4 -u root -p
```

```sql
USE ql_banhang;
```

Kiểm tra tổng hợp:

```sql
SELECT
    k.TABLE_NAME,
    k.COLUMN_NAME,
    c.COLUMN_TYPE,
    c.IS_NULLABLE,
    k.REFERENCED_TABLE_NAME,
    k.REFERENCED_COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE AS k
JOIN information_schema.COLUMNS AS c
    ON k.TABLE_SCHEMA = c.TABLE_SCHEMA
    AND k.TABLE_NAME = c.TABLE_NAME
    AND k.COLUMN_NAME = c.COLUMN_NAME
WHERE k.TABLE_SCHEMA = 'ql_banhang'
  AND k.REFERENCED_TABLE_NAME IS NOT NULL
ORDER BY
    k.TABLE_NAME,
    k.COLUMN_NAME;
```

Tất cả 8 khóa ngoại phải có:

```text
IS_NULLABLE = NO
```

Kiểm tra nhanh PASS/FAIL:

```sql
SELECT
    CASE
        WHEN COUNT(*) = 0
        THEN 'PASS - Tat ca khoa ngoai deu NOT NULL'
        ELSE CONCAT(
            'FAIL - Con ',
            COUNT(*),
            ' khoa ngoai cho phep NULL'
        )
    END AS Result
FROM information_schema.KEY_COLUMN_USAGE AS k
JOIN information_schema.COLUMNS AS c
    ON k.TABLE_SCHEMA = c.TABLE_SCHEMA
    AND k.TABLE_NAME = c.TABLE_NAME
    AND k.COLUMN_NAME = c.COLUMN_NAME
WHERE k.TABLE_SCHEMA = 'ql_banhang'
  AND k.REFERENCED_TABLE_NAME IS NOT NULL
  AND c.IS_NULLABLE = 'YES';
```

Kiểm tra seed:

```sql
SELECT COUNT(*) AS ProductCount FROM products;
SELECT COUNT(*) AS OrderCount FROM orders;
```

Mong đợi:

```text
ProductCount = 3
OrderCount   = 3
```

---

# 12. Kết bảng theo quan hệ khóa chính – khóa ngoại

Quan hệ:

```text
products.CategoryID (FK)
        =
categories.CategoryID (PK)

products.SupplierID (FK)
        =
suppliers.SupplierID (PK)
```

## 12.1. Cách 1 – dùng `FROM ... WHERE ... AND ...`

```sql
SELECT
    p.ProductID,
    p.ProductCode,
    p.ProductName,
    p.Unit,
    p.Price,
    p.StockQuantity,
    p.IsActive,
    c.CategoryName,
    s.SupplierName
FROM
    products AS p,
    categories AS c,
    suppliers AS s
WHERE
    p.CategoryID = c.CategoryID
    AND p.SupplierID = s.SupplierID
ORDER BY
    p.ProductID;
```

## 12.2. Cách 2 – viết tương đương bằng `INNER JOIN ... ON`

```sql
SELECT
    p.ProductID,
    p.ProductCode,
    p.ProductName,
    p.Unit,
    p.Price,
    p.StockQuantity,
    p.IsActive,
    c.CategoryName,
    s.SupplierName
FROM products AS p
INNER JOIN categories AS c
    ON p.CategoryID = c.CategoryID
INNER JOIN suppliers AS s
    ON p.SupplierID = s.SupplierID
ORDER BY
    p.ProductID;
```

Hai câu phải trả cùng kết quả.

Điểm cần hiểu:

```text
quan sát quan hệ
→ xác định PK và FK
→ viết PK = FK
→ chọn cú pháp SQL phù hợp
```

---

# 13. Tạo trang danh sách sản phẩm

Thoát MySQL:

```sql
exit;
```

Tạo:

```cmd
mkdir public\products
type nul > public\products\index.php
```

Nếu thư mục đã tồn tại, chỉ tạo file còn thiếu.

Đây là file mới nên nhập đầy đủ:

```php
<?php

$pageTitle = 'Quản lý sản phẩm';

require_once '/var/www/src/config/database.php';

$sql = "
    SELECT
        p.ProductID,
        p.ProductCode,
        p.ProductName,
        p.Unit,
        p.Price,
        p.StockQuantity,
        p.IsActive,
        c.CategoryName,
        s.SupplierName
    FROM
        products AS p,
        categories AS c,
        suppliers AS s
    WHERE
        p.CategoryID = c.CategoryID
        AND p.SupplierID = s.SupplierID
    ORDER BY
        p.ProductID
";

$result = $conn->query($sql);

require_once '/var/www/src/includes/header.php';
require_once '/var/www/src/includes/navbar.php';

?>

<div class="container mt-4">

    <div class="d-flex justify-content-between align-items-center mb-3">
        <h2>Quản lý sản phẩm</h2>

        <a href="#" class="btn btn-primary">
            Thêm sản phẩm
        </a>
    </div>

    <div class="table-responsive">

        <table class="table table-bordered table-striped align-middle">

            <thead class="table-dark">
                <tr>
                    <th>Mã SP</th>
                    <th>Tên sản phẩm</th>
                    <th>Danh mục</th>
                    <th>Nhà cung cấp</th>
                    <th>Đơn vị</th>
                    <th>Giá</th>
                    <th>Tồn kho</th>
                    <th>Trạng thái</th>
                    <th>Thao tác</th>
                </tr>
            </thead>

            <tbody>

            <?php while ($product = $result->fetch_assoc()): ?>

                <tr>

                    <td><?= htmlspecialchars($product['ProductCode']) ?></td>

                    <td><?= htmlspecialchars($product['ProductName']) ?></td>

                    <td><?= htmlspecialchars($product['CategoryName']) ?></td>

                    <td><?= htmlspecialchars($product['SupplierName']) ?></td>

                    <td><?= htmlspecialchars($product['Unit'] ?? '') ?></td>

                    <td class="text-end">
                        <?= number_format(
                            (float) $product['Price'],
                            0,
                            ',',
                            '.'
                        ) ?> đ
                    </td>

                    <td class="text-end">
                        <?= (int) $product['StockQuantity'] ?>
                    </td>

                    <td>
                        <?php if ((int) $product['IsActive'] === 1): ?>

                            <span class="badge bg-success">
                                Đang bán
                            </span>

                        <?php else: ?>

                            <span class="badge bg-secondary">
                                Ngừng bán
                            </span>

                        <?php endif; ?>
                    </td>

                    <td>
                        <a href="#" class="btn btn-sm btn-warning">
                            Sửa
                        </a>

                        <a href="#" class="btn btn-sm btn-danger">
                            Xóa
                        </a>
                    </td>

                </tr>

            <?php endwhile; ?>

            </tbody>

        </table>

    </div>

</div>

<?php

require_once '/var/www/src/includes/footer.php';

$conn->close();
```

---

# 14. Cập nhật menu Sản phẩm

Trong:

```text
src/includes/navbar.php
```

tìm mục **Sản phẩm** đang có:

```text
href="#"
```

chỉ thay `#` bằng:

```text
/products/
```

Không thay toàn bộ file.

Kiểm tra:

```text
http://localhost:8080/products/
```

---

# 15. Kiến thức mới trong danh sách sản phẩm

## `number_format()`

```php
number_format(
    (float) $product['Price'],
    0,
    ',',
    '.'
)
```

Ví dụ:

```text
8500000.00 → 8.500.000
```

## Ép kiểu

```php
(float) $product['Price']
(int) $product['StockQuantity']
```

## `IsActive`

```text
1 → Đang bán
0 → Ngừng bán
```

---

# 16. Liên kết nút “Thêm sản phẩm”

Trong:

```text
public/products/index.php
```

tìm nút **Thêm sản phẩm** có:

```text
href="#"
```

chỉ thay thành:

```text
/products/create.php
```

---

# 17. Tạo chức năng thêm sản phẩm

Tạo:

```cmd
type nul > public\products\create.php
```

Đây là file mới. Nội dung chính:

```php
<?php

$pageTitle = 'Thêm sản phẩm';

require_once '/var/www/src/config/database.php';

$error = '';

$sqlCategories = "
    SELECT CategoryID, CategoryName
    FROM categories
    ORDER BY CategoryName
";

$categories = $conn->query($sqlCategories);

$sqlSuppliers = "
    SELECT SupplierID, SupplierName
    FROM suppliers
    ORDER BY SupplierName
";

$suppliers = $conn->query($sqlSuppliers);

if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    $productCode = trim($_POST['product_code'] ?? '');
    $productName = trim($_POST['product_name'] ?? '');
    $description = trim($_POST['description'] ?? '');
    $unit = trim($_POST['unit'] ?? '');

    $price = (float) ($_POST['price'] ?? 0);
    $stockQuantity = (int) ($_POST['stock_quantity'] ?? 0);

    $categoryID = (int) ($_POST['category_id'] ?? 0);
    $supplierID = (int) ($_POST['supplier_id'] ?? 0);

    $isActive = isset($_POST['is_active']) ? 1 : 0;

    if ($productCode === '') {
        $error = 'Mã sản phẩm không được để trống.';

    } elseif ($productName === '') {
        $error = 'Tên sản phẩm không được để trống.';

    } elseif ($price < 0) {
        $error = 'Giá sản phẩm không hợp lệ.';

    } elseif ($stockQuantity < 0) {
        $error = 'Số lượng tồn kho không hợp lệ.';

    } elseif ($categoryID <= 0) {
        $error = 'Vui lòng chọn danh mục.';

    } elseif ($supplierID <= 0) {
        $error = 'Vui lòng chọn nhà cung cấp.';

    } else {

        $sql = "
            INSERT INTO products
            (
                ProductCode,
                ProductName,
                Description,
                Unit,
                Price,
                StockQuantity,
                IsActive,
                SupplierID,
                CategoryID
            )
            VALUES
            (?, ?, ?, ?, ?, ?, ?, ?, ?)
        ";

        $stmt = $conn->prepare($sql);

        $stmt->bind_param(
            'ssssdiiii',
            $productCode,
            $productName,
            $description,
            $unit,
            $price,
            $stockQuantity,
            $isActive,
            $supplierID,
            $categoryID
        );

        if ($stmt->execute()) {
            header('Location: /products/');
            exit;
        }

        $error = 'Không thể thêm sản phẩm.';
        $stmt->close();
    }
}

require_once '/var/www/src/includes/header.php';
require_once '/var/www/src/includes/navbar.php';
?>
```

Phần form cần có các trường:

```text
ProductCode
ProductName
Description
Unit
Price
StockQuantity
Category
Supplier
IsActive
```

Category và Supplier phải dùng `<select>` động.

Ví dụ:

```php
<select name="category_id" class="form-select" required>

    <option value="">
        -- Chọn danh mục --
    </option>

    <?php while ($category = $categories->fetch_assoc()): ?>

        <option
            value="<?= $category['CategoryID'] ?>"
        >
            <?= htmlspecialchars($category['CategoryName']) ?>
        </option>

    <?php endwhile; ?>

</select>
```

Supplier làm tương tự.

> Khi đóng gói bài thực hành chính thức, người học cần tự viết form dựa trên cấu trúc Category CRUD đã học; không nên chỉ copy một khối HTML quá lớn mà không hiểu từng control.

---

# 18. Dữ liệu mẫu để thử Create Product

| Trường | Giá trị |
|---|---|
| Mã sản phẩm | `SP004` |
| Tên sản phẩm | `Bàn phím cơ D` |
| Mô tả | `Bàn phím cơ có dây, phù hợp học tập và làm việc.` |
| Đơn vị | `Chiếc` |
| Giá | `1250000` |
| Tồn kho | `25` |
| Danh mục | `Phụ kiện` |
| Nhà cung cấp | `Công ty Thiết bị XYZ` |
| Đang kinh doanh | Có |

Kết quả mong đợi:

```text
SP004
Bàn phím cơ D
Phụ kiện
Công ty Thiết bị XYZ
1.250.000 đ
25
Đang bán
```

---

# 19. Cập nhật nút Sửa

Trong:

```text
public/products/index.php
```

tìm nút **Sửa** đang có:

```text
href="#"
```

chỉ thay bằng:

```php
href="/products/edit.php?id=<?= $product['ProductID'] ?>"
```

---

# 20. Tạo chức năng sửa sản phẩm

Tạo:

```cmd
type nul > public\products\edit.php
```

Logic chính:

1. lấy `ProductID` từ `$_GET['id']`;
2. `SELECT` sản phẩm hiện tại;
3. lấy danh sách Category;
4. lấy danh sách Supplier;
5. hiển thị dữ liệu hiện tại vào form;
6. khi POST thì `UPDATE`;
7. redirect về `/products/`.

Prepared statement của UPDATE:

```php
$sql = "
    UPDATE products
    SET
        ProductCode = ?,
        ProductName = ?,
        Description = ?,
        Unit = ?,
        Price = ?,
        StockQuantity = ?,
        IsActive = ?,
        SupplierID = ?,
        CategoryID = ?
    WHERE ProductID = ?
";

$stmt = $conn->prepare($sql);

$stmt->bind_param(
    'ssssdiiiii',
    $productCode,
    $productName,
    $description,
    $unit,
    $price,
    $stockQuantity,
    $isActive,
    $supplierID,
    $categoryID,
    $productID
);
```

Điểm mới quan trọng của form Edit:

```php
$selectedCategoryID =
    $_POST['category_id']
    ?? $product['CategoryID'];
```

và:

```php
$selectedSupplierID =
    $_POST['supplier_id']
    ?? $product['SupplierID'];
```

để `<select>` tự chọn đúng khóa ngoại hiện tại.

---

# 21. Dữ liệu mẫu để thử Update

Chọn:

```text
SP004 – Bàn phím cơ D
```

sửa:

| Trường | Giá trị mới |
|---|---|
| Tên sản phẩm | `Bàn phím cơ D Pro` |
| Giá | `1450000` |
| Tồn kho | `30` |
| Danh mục | `Phụ kiện` |
| Nhà cung cấp | `Công ty Công nghệ ABC` |
| Đang kinh doanh | Có |

Kết quả:

```text
SP004
Bàn phím cơ D Pro
Phụ kiện
Công ty Công nghệ ABC
1.450.000 đ
30
Đang bán
```

---

# 22. Chức năng Delete Product

Trong `public/products/index.php`, tìm riêng nút Xóa:

```php
<a href="#" class="btn btn-sm btn-danger">
    Xóa
</a>
```

thay riêng thẻ đó bằng form POST:

```php
<form
    action="/products/delete.php"
    method="post"
    class="d-inline"
    onsubmit="return confirm('Bạn có chắc muốn xóa sản phẩm này?');"
>
    <input
        type="hidden"
        name="id"
        value="<?= $product['ProductID'] ?>"
    >

    <button
        type="submit"
        class="btn btn-sm btn-danger"
    >
        Xóa
    </button>
</form>
```

Tạo:

```cmd
type nul > public\products\delete.php
```

Nội dung:

```php
<?php

require_once '/var/www/src/config/database.php';

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    header('Location: /products/');
    exit;
}

$productID = isset($_POST['id'])
    ? (int) $_POST['id']
    : 0;

if ($productID <= 0) {
    header('Location: /products/');
    exit;
}

$sql = "
    DELETE FROM products
    WHERE ProductID = ?
";

$stmt = $conn->prepare($sql);
$stmt->bind_param('i', $productID);

if ($stmt->execute()) {

    $stmt->close();
    $conn->close();

    header('Location: /products/');
    exit;
}

$stmt->close();
$conn->close();

die('Không thể xóa sản phẩm.');
```

Nên thử xóa SP004 vì sản phẩm này chưa có trong `orderdetail`.

---

# 23. Vì sao có sản phẩm không thể xóa?

`product_images.ProductID` có:

```sql
ON DELETE CASCADE
```

nên ảnh dữ liệu liên quan có thể tự xóa khi sản phẩm bị xóa.

Nhưng `orderdetail.ProductID` không dùng `ON DELETE CASCADE`.

Điều này bảo vệ lịch sử đơn hàng.

Với sản phẩm đã phát sinh giao dịch:

```text
không nên xóa vật lý
→ có thể dùng IsActive = 0
→ Ngừng kinh doanh
```

---

# 24. Kiểm tra dữ liệu ảnh

Đăng nhập MySQL và chạy:

```sql
USE ql_banhang;

SELECT
    ProductImageID,
    ProductID,
    ImageFile,
    AltText,
    IsPrimary,
    SortOrder
FROM product_images
ORDER BY ProductID, SortOrder;
```

Mong đợi:

```text
Product 1 → phone-a-1.webp  → IsPrimary = 1
Product 1 → phone-a-2.webp  → IsPrimary = 0
Product 2 → laptop-b-1.webp → IsPrimary = 1
Product 3 → mouse-c-1.webp  → IsPrimary = 1
```

---

# 25. Mở rộng truy vấn sản phẩm để lấy ảnh chính

Trong:

```text
public/products/index.php
```

không thay toàn bộ file.

Trong phần `SELECT`, bổ sung:

```sql
pi.ImageFile,
pi.AltText
```

Trong `FROM`, bổ sung:

```sql
product_images AS pi
```

Trong `WHERE`, bổ sung:

```sql
AND p.ProductID = pi.ProductID
AND pi.IsPrimary = 1
```

Câu truy vấn sau thay đổi:

```php
$sql = "
    SELECT
        p.ProductID,
        p.ProductCode,
        p.ProductName,
        p.Unit,
        p.Price,
        p.StockQuantity,
        p.IsActive,
        c.CategoryName,
        s.SupplierName,
        pi.ImageFile,
        pi.AltText
    FROM
        products AS p,
        categories AS c,
        suppliers AS s,
        product_images AS pi
    WHERE
        p.CategoryID = c.CategoryID
        AND p.SupplierID = s.SupplierID
        AND p.ProductID = pi.ProductID
        AND pi.IsPrimary = 1
    ORDER BY
        p.ProductID
";
```

---

# 26. Thêm cột Hình ảnh

Trong `<thead>`, chèn trước cột **Mã SP**:

```html
<th>Hình ảnh</th>
```

Trong `<tbody>`, ngay đầu `<tr>`, thêm:

```php
<?php
$imageFile = $product['ImageFile'] ?? '';
$altText = $product['AltText'] ?? $product['ProductName'];
?>

<td>
    <img
        src="/uploads/products/<?= htmlspecialchars($imageFile) ?>"
        alt="<?= htmlspecialchars($altText) ?>"
        width="80"
        class="img-thumbnail"
    >
</td>
```

Luồng:

```text
product_images.ImageFile
        ↓
$product['ImageFile']
        ↓
/uploads/products/<tên-file>
        ↓
public/uploads/products/<tên-file>
        ↓
Browser
```

---

# 27. Xử lý sự cố thường gặp

## 27.1. `ImageFile` có trong database nhưng PHP không có giá trị

`fetch_assoc()` chỉ chứa các cột có trong `SELECT`.

Kiểm tra câu query đã có:

```sql
pi.ImageFile,
pi.AltText
```

hay chưa.

Có thể debug tạm:

```php
echo '<pre>';
print_r($product);
echo '</pre>';
```

Sau khi kiểm tra phải xóa đoạn debug.

## 27.2. Biểu tượng ảnh bị vỡ

Kiểm tra trực tiếp:

```text
http://localhost:8080/uploads/products/phone-a-1.webp
```

và:

```cmd
dir public\uploads\products
```

Tên file vật lý và tên trong database phải giống hoàn toàn.

---

# 28. Checkpoint Hands-on 05

```text
[ ] Xác định đủ 8 khóa ngoại
[ ] Tất cả khóa ngoại đều NOT NULL
[ ] Hiểu FOREIGN KEY và NOT NULL
[ ] Kiểm tra dữ liệu NULL trước khi reset

[ ] Tự chuẩn bị 4 ảnh sản phẩm
[ ] Đặt ảnh tại public/uploads/products/
[ ] Tên và phần mở rộng file ảnh khớp ImageFile trong seed.sql
[ ] Database tạo lại thành công

[ ] Chạy được FROM ... WHERE ... AND ...
[ ] Hiểu INNER JOIN ... ON tương đương
[ ] Menu Sản phẩm hoạt động
[ ] Danh sách Product hoạt động

[ ] Create Product hoạt động
[ ] Category/Supplier dùng select động
[ ] Update Product hoạt động
[ ] Delete Product hoạt động

[ ] product_images được kết với products
[ ] Lấy đúng IsPrimary = 1
[ ] Ảnh chính hiển thị đúng
[ ] Tiếng Việt hiển thị đúng
```

---

# 29. Commit và push checkpoint HO05

Kiểm tra:

```cmd
git status
git diff
```

Các thay đổi chính có thể gồm:

```text
database/schema.sql
database/seed.sql

src/includes/navbar.php

public/products/index.php
public/products/create.php
public/products/edit.php
public/products/delete.php

public/uploads/products/
```

Stage:

```cmd
git add .
git status
```

Commit:

```cmd
git commit -m "Add product CRUD and relational data handling"
```

Push:

```cmd
git push
```

Kiểm tra:

```cmd
git status
git log -1 --oneline
```

Mong đợi:

```text
nothing to commit, working tree clean
```

---

# 30. Kiến thức cần ghi nhớ

## Khóa ngoại bắt buộc

```text
FOREIGN KEY + NOT NULL
```

## Kết bảng

```text
FROM nhiều bảng
WHERE PK = FK
AND PK = FK
```

tương đương trong trường hợp hiện tại với:

```text
INNER JOIN
ON PK = FK
```

## Select động

```text
database
→ SELECT
→ <option value="ID">Tên</option>
```

## CRUD Product

```text
Create → INSERT
Read   → SELECT + kết bảng
Update → UPDATE
Delete → DELETE
```

## Ảnh sản phẩm

```text
Database
→ chỉ lưu ImageFile

File system
→ public/uploads/products/
```

---

# 31. Chuẩn bị cho Hands-on tiếp theo

HO05 mới dùng ảnh **đã chuẩn bị sẵn**.

Hands-on tiếp theo sẽ học:

```text
multipart/form-data
$_FILES
upload file
kiểm tra định dạng
move_uploaded_file()
INSERT product_images
nhiều ảnh cho một sản phẩm
ảnh chính
```

Đó là bước chuyển từ **hiển thị ảnh có sẵn** sang **quản lý ảnh do người dùng upload**.
