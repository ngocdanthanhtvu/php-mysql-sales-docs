# Hands-on 12 -- Xử lý đặt hàng và lưu đơn hàng vào MySQL

## Mục tiêu

Sau khi hoàn thành Hands-on này, bạn có thể:

- mô tả luồng dữ liệu từ giỏ hàng đến đơn hàng trong cơ sở dữ liệu;
- mở rộng cấu trúc `customers` và `orders` để phù hợp với nghiệp vụ đặt hàng;
- xây dựng trang Checkout nhận thông tin khách hàng;
- đọc lại giá và tồn kho từ MySQL trước khi tạo đơn hàng;
- sử dụng transaction để xử lý nhiều thao tác cơ sở dữ liệu như một đơn vị công việc;
- lưu dữ liệu vào `customers`, `orders` và `orderdetail`;
- cập nhật tồn kho sau khi đặt hàng thành công;
- sử dụng `FOR UPDATE` để khóa mẩu tin sản phẩm trong quá trình kiểm tra tồn kho;
- rollback toàn bộ giao dịch khi một sản phẩm không còn đủ tồn kho;
- chỉ xóa giỏ hàng trong Session sau khi transaction được commit thành công;
- xây dựng trang xác nhận và hiển thị thông tin đơn hàng;
- cấu hình múi giờ cho môi trường Docker để thời điểm đặt hàng được ghi nhận đúng;
- kiểm thử luồng đặt hàng thành công và trường hợp transaction bị rollback.

> **Phạm vi Hands-on 12:** xây dựng quy trình đặt hàng cho khách chưa đăng nhập. Mỗi lần đặt hàng, thông tin khách được lưu vào `customers`, sau đó tạo `orders` và `orderdetail`. Chưa xây dựng tài khoản khách hàng, thanh toán trực tuyến hoặc chức năng xử lý đơn hàng cho quản trị viên.

------------------------------------------------------------------------

## 1. Chuẩn bị

Hands-on này tiếp tục trực tiếp từ **Hands-on 11**. Ở thời điểm bắt đầu, ứng dụng đã có:

- trang danh sách và chi tiết sản phẩm;
- giỏ hàng lưu bằng PHP Session;
- trang `/cart.php` cho phép cập nhật và xóa sản phẩm;
- số lượng sản phẩm trong giỏ được hiển thị trên Navbar;
- bảng `customers`, `orders` và `orderdetail` đã có trong cơ sở dữ liệu.

Mở **Command Prompt** và chuyển đến thư mục dự án:

```cmd
cd /d D:\PTUDW-ST-2026\php-mysql-sales
```

Kiểm tra Git:

```cmd
git status
```

Nên bắt đầu khi kết quả có:

```text
nothing to commit, working tree clean
```

Kiểm tra Docker:

```cmd
docker compose ps
```

Nếu các container chưa chạy:

```cmd
docker compose up -d
```

Mở ứng dụng:

```text
http://localhost:8080/
```

------------------------------------------------------------------------

## 2. Phân tích luồng đặt hàng

Ở Hands-on 11, Session chỉ lưu:

```php
$_SESSION['cart'] = [
    ProductID => Quantity
];
```

Khi khách xác nhận đặt hàng, dữ liệu cần được chuyển từ trạng thái tạm thời trong Session sang dữ liệu lâu dài trong MySQL.

Luồng xử lý được thiết kế như sau:

```text
Giỏ hàng
   │
   ▼
Checkout
   │
   ├── nhập thông tin khách hàng
   │
   └── kiểm tra lại sản phẩm
          │
          ▼
      Transaction
          │
          ├── INSERT customers
          ├── INSERT orders
          ├── INSERT orderdetail
          └── UPDATE products.StockQuantity
          │
          ▼
        COMMIT
          │
          ├── xóa Session cart
          └── chuyển đến trang xác nhận đơn hàng
```

Nếu một thao tác quan trọng thất bại, transaction phải được **rollback**:

```text
Không đủ tồn kho
      │
      ▼
   ROLLBACK
      │
      ├── không tạo Customer
      ├── không tạo Order
      ├── không tạo OrderDetail
      ├── không trừ tồn kho
      └── giữ nguyên giỏ hàng
```

Đây là điểm khác biệt quan trọng so với việc thực hiện từng câu lệnh SQL độc lập: một đơn hàng chỉ được xem là thành công khi **toàn bộ** các thao tác liên quan đều thành công.

------------------------------------------------------------------------

## 3. Mở rộng cấu trúc dữ liệu cho nghiệp vụ đặt hàng

### 3.1. Bổ sung số điện thoại cho khách hàng

Mở:

```text
database/schema.sql
```

Tìm phần khai báo bảng `customers`. Sau dòng:

```sql
Country VARCHAR(100)
```

thêm trường `Phone` và đồng thời thêm dấu phẩy cho dòng trước:

```sql
Country VARCHAR(100),
Phone VARCHAR(20)
```

Với cơ sở dữ liệu đang tồn tại từ các Hands-on trước, thay đổi cấu trúc thật bằng:

```sql
ALTER TABLE customers
ADD COLUMN Phone VARCHAR(20) AFTER Country;
```

### 3.2. Mở rộng bảng `orders`

Trong `database/schema.sql`, tìm phần khai báo `orders` và điều chỉnh các trường liên quan để có cấu trúc:

```sql
CREATE TABLE orders (
    OrderID INT AUTO_INCREMENT PRIMARY KEY,

    OrderDate DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    TotalAmount DECIMAL(12,2) NOT NULL DEFAULT 0,
    Status VARCHAR(30) NOT NULL DEFAULT 'Pending',

    CustomerID INT NOT NULL,
    EmployeeID INT,
    ShipperID INT,

    CONSTRAINT chk_orders_total
        CHECK (TotalAmount >= 0),

    CONSTRAINT fk_orders_customer
        FOREIGN KEY (CustomerID)
        REFERENCES customers(CustomerID),

    CONSTRAINT fk_orders_employee
        FOREIGN KEY (EmployeeID)
        REFERENCES employees(EmployeeID),

    CONSTRAINT fk_orders_shipper
        FOREIGN KEY (ShipperID)
        REFERENCES shippers(ShipperID)
) CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

Các thay đổi chính gồm:

- `OrderDate` chuyển từ `DATE` sang `DATETIME` và tự nhận `CURRENT_TIMESTAMP`;
- `TotalAmount` lưu tổng giá trị đơn hàng;
- `Status` lưu trạng thái xử lý, mặc định là `Pending`;
- `EmployeeID` và `ShipperID` cho phép `NULL` vì đơn vừa được khách tạo chưa được nhân viên xử lý hoặc giao cho đơn vị vận chuyển;
- `CHECK` bảo đảm tổng tiền không âm.

Không thay đổi bảng `orderdetail`. Trường `UnitPrice` tiếp tục được dùng để lưu **đơn giá tại thời điểm đặt hàng**, thay vì phụ thuộc vào giá sản phẩm có thể thay đổi sau này.

### 3.3. Cập nhật cơ sở dữ liệu đang sử dụng

Trong MySQL, chạy:

```sql
ALTER TABLE orders
    MODIFY COLUMN OrderDate DATETIME
        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    ADD COLUMN TotalAmount DECIMAL(12,2)
        NOT NULL DEFAULT 0
        AFTER OrderDate,
    ADD COLUMN Status VARCHAR(30)
        NOT NULL DEFAULT 'Pending'
        AFTER TotalAmount,
    MODIFY COLUMN EmployeeID INT NULL,
    MODIFY COLUMN ShipperID INT NULL,
    ADD CONSTRAINT chk_orders_total
        CHECK (TotalAmount >= 0);
```

Kiểm tra:

```sql
DESCRIBE customers;
```

và:

```sql
SHOW CREATE TABLE orders;
```

Kết quả cần xác nhận được `Phone`, `TotalAmount`, `Status`, `OrderDate` kiểu `DATETIME`, đồng thời `EmployeeID` và `ShipperID` cho phép `NULL`.

------------------------------------------------------------------------

## 4. Cấu hình múi giờ cho môi trường Docker

`OrderDate` sử dụng `CURRENT_TIMESTAMP`, vì vậy thời gian hệ thống của container cần phù hợp với môi trường triển khai.

Mở:

```text
compose.yaml
```

Trong `environment` của service `web`, thêm:

```yaml
TZ: Asia/Ho_Chi_Minh
```

Ví dụ:

```yaml
environment:
  TZ: Asia/Ho_Chi_Minh
  MYSQL_DATABASE: ${MYSQL_DATABASE}
  MYSQL_USER: ${MYSQL_USER}
  MYSQL_PASSWORD: ${MYSQL_PASSWORD}
```

Trong `environment` của service `db`, cũng thêm:

```yaml
TZ: Asia/Ho_Chi_Minh
```

Sau khi lưu, tạo lại container để cấu hình mới có hiệu lực:

```cmd
docker compose up -d --force-recreate
```

Kiểm tra:

```cmd
docker compose exec web date
docker compose exec db date
```

Tiếp tục vào MySQL và chạy:

```sql
SELECT
    NOW() AS mysql_now,
    UTC_TIMESTAMP() AS utc_now,
    @@global.time_zone AS global_time_zone,
    @@session.time_zone AS session_time_zone;
```

`NOW()` phải thể hiện giờ Việt Nam và chênh `UTC_TIMESTAMP()` 7 giờ.

> Khi `global_time_zone` và `session_time_zone` hiển thị `SYSTEM`, MySQL đang sử dụng múi giờ của hệ thống container. Điều này phù hợp khi container đã được cấu hình `Asia/Ho_Chi_Minh`.

------------------------------------------------------------------------

## 5. Bổ sung nút chuyển từ giỏ hàng đến Checkout

Mở:

```text
public/cart.php
```

Ở cuối phần hiển thị giỏ hàng, tìm khối chứa nút:

```text
Tiếp tục mua hàng
```

Thay phần bao quanh nút này bằng:

```php
<div
    class="d-flex
           justify-content-between
           align-items-center
           mt-4"
>
    <a
        href="/products.php"
        class="btn btn-outline-secondary"
    >
        Tiếp tục mua hàng
    </a>

    <a
        href="/checkout.php"
        class="btn btn-success"
    >
        Tiến hành đặt hàng
    </a>
</div>
```

Kiểm tra tại:

```text
http://localhost:8080/cart.php
```

Khi giỏ có sản phẩm, phía dưới phải xuất hiện hai hướng xử lý: **Tiếp tục mua hàng** và **Tiến hành đặt hàng**.

------------------------------------------------------------------------

## 6. Tạo trang Checkout

Tạo tập tin mới:

```text
public/checkout.php
```

Nhập toàn bộ nội dung sau:

```php
<?php

require_once '/var/www/src/config/session.php';
require_once '/var/www/src/config/database.php';

$pageTitle = 'Đặt hàng';

$cart = $_SESSION['cart'] ?? [];

if (empty($cart)) {
    header('Location: /cart.php');
    exit;
}

function getCartItems($conn, $cart)
{
    $items = [];
    $total = 0;

    $sql = "
        SELECT
            p.ProductID,
            p.ProductCode,
            p.ProductName,
            p.Price,
            p.StockQuantity,
            (
                SELECT pi.ImageFile
                FROM product_images pi
                WHERE pi.ProductID = p.ProductID
                  AND pi.IsPrimary = 1
                LIMIT 1
            ) AS ImageFile
        FROM products p
        WHERE p.ProductID = ?
          AND p.IsActive = 1
    ";

    $stmt = $conn->prepare($sql);

    foreach ($cart as $productID => $quantity) {
        $productID = (int) $productID;
        $quantity = (int) $quantity;

        if ($productID <= 0 || $quantity <= 0) {
            continue;
        }

        $stmt->bind_param('i', $productID);
        $stmt->execute();
        $result = $stmt->get_result();
        $product = $result->fetch_assoc();
        $result->free();

        if (!$product) {
            continue;
        }

        $product['Quantity'] = $quantity;
        $product['Subtotal'] =
            (float) $product['Price'] * $quantity;

        $total += $product['Subtotal'];
        $items[] = $product;
    }

    $stmt->close();

    return [
        'items' => $items,
        'total' => $total
    ];
}

$cartData = getCartItems($conn, $cart);
$cartItems = $cartData['items'];
$total = $cartData['total'];

if (empty($cartItems)) {
    header('Location: /cart.php');
    exit;
}

$errorMessage = '';
$customerName = '';
$phone = '';
$address = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST'
    && isset($_POST['place_order'])) {

    $customerName = trim($_POST['customer_name'] ?? '');
    $phone = trim($_POST['phone'] ?? '');
    $address = trim($_POST['address'] ?? '');

    if ($customerName === ''
        || $phone === ''
        || $address === '') {

        $errorMessage =
            'Vui lòng nhập đầy đủ thông tin khách hàng.';

    } else {

        try {
            $conn->begin_transaction();

            $orderItems = [];
            $orderTotal = 0;

            $sqlProduct = "
                SELECT
                    ProductID,
                    ProductName,
                    Price,
                    StockQuantity
                FROM products
                WHERE ProductID = ?
                  AND IsActive = 1
                FOR UPDATE
            ";

            $stmtProduct = $conn->prepare($sqlProduct);

            foreach ($cart as $productID => $quantity) {
                $productID = (int) $productID;
                $quantity = (int) $quantity;

                if ($productID <= 0 || $quantity <= 0) {
                    throw new Exception(
                        'Dữ liệu giỏ hàng không hợp lệ.'
                    );
                }

                $stmtProduct->bind_param('i', $productID);
                $stmtProduct->execute();
                $productResult = $stmtProduct->get_result();
                $product = $productResult->fetch_assoc();
                $productResult->free();

                if (!$product) {
                    throw new Exception(
                        'Có sản phẩm không còn khả dụng.'
                    );
                }

                if ($quantity > (int) $product['StockQuantity']) {
                    throw new Exception(
                        'Sản phẩm "'
                        . $product['ProductName']
                        . '" không đủ số lượng tồn kho.'
                    );
                }

                $unitPrice = (float) $product['Price'];
                $subtotal = $unitPrice * $quantity;

                $orderTotal += $subtotal;

                $orderItems[] = [
                    'ProductID' => $productID,
                    'Quantity' => $quantity,
                    'UnitPrice' => $unitPrice
                ];
            }

            $stmtProduct->close();

            $sqlCustomer = "
                INSERT INTO customers
                    (CustomerName, Address, Phone)
                VALUES (?, ?, ?)
            ";

            $stmtCustomer = $conn->prepare($sqlCustomer);
            $stmtCustomer->bind_param(
                'sss',
                $customerName,
                $address,
                $phone
            );
            $stmtCustomer->execute();

            $customerID = $conn->insert_id;
            $stmtCustomer->close();

            $status = 'Pending';

            $sqlOrder = "
                INSERT INTO orders
                    (TotalAmount, Status, CustomerID)
                VALUES (?, ?, ?)
            ";

            $stmtOrder = $conn->prepare($sqlOrder);
            $stmtOrder->bind_param(
                'dsi',
                $orderTotal,
                $status,
                $customerID
            );
            $stmtOrder->execute();

            $orderID = $conn->insert_id;
            $stmtOrder->close();

            $sqlDetail = "
                INSERT INTO orderdetail
                    (Quantity, UnitPrice, OrderID, ProductID)
                VALUES (?, ?, ?, ?)
            ";

            $stmtDetail = $conn->prepare($sqlDetail);

            $sqlStock = "
                UPDATE products
                SET StockQuantity = StockQuantity - ?
                WHERE ProductID = ?
            ";

            $stmtStock = $conn->prepare($sqlStock);

            foreach ($orderItems as $item) {
                $quantity = $item['Quantity'];
                $unitPrice = $item['UnitPrice'];
                $productID = $item['ProductID'];

                $stmtDetail->bind_param(
                    'idii',
                    $quantity,
                    $unitPrice,
                    $orderID,
                    $productID
                );
                $stmtDetail->execute();

                $stmtStock->bind_param(
                    'ii',
                    $quantity,
                    $productID
                );
                $stmtStock->execute();
            }

            $stmtDetail->close();
            $stmtStock->close();

            $conn->commit();

            $_SESSION['cart'] = [];

            header(
                'Location: /order-success.php?id=' . $orderID
            );
            exit;

        } catch (Throwable $e) {
            $conn->rollback();
            $errorMessage = $e->getMessage();
        }
    }
}

require_once '/var/www/src/includes/frontend/header.php';
require_once '/var/www/src/includes/frontend/navbar.php';
?>

<div class="container py-4">

    <h1 class="h3 mb-4">Đặt hàng</h1>

    <?php if ($errorMessage !== ''): ?>
        <div class="alert alert-danger">
            <?= htmlspecialchars($errorMessage) ?>
        </div>
    <?php endif; ?>

    <div class="row g-4">

        <div class="col-lg-7">
            <div class="card shadow-sm">
                <div class="card-body">

                    <h2 class="h5 mb-3">
                        Thông tin khách hàng
                    </h2>

                    <form method="post">

                        <div class="mb-3">
                            <label
                                for="customer_name"
                                class="form-label"
                            >
                                Họ tên
                            </label>
                            <input
                                type="text"
                                class="form-control"
                                id="customer_name"
                                name="customer_name"
                                value="<?= htmlspecialchars($customerName) ?>"
                                required
                            >
                        </div>

                        <div class="mb-3">
                            <label
                                for="phone"
                                class="form-label"
                            >
                                Số điện thoại
                            </label>
                            <input
                                type="text"
                                class="form-control"
                                id="phone"
                                name="phone"
                                value="<?= htmlspecialchars($phone) ?>"
                                required
                            >
                        </div>

                        <div class="mb-3">
                            <label
                                for="address"
                                class="form-label"
                            >
                                Địa chỉ
                            </label>
                            <textarea
                                class="form-control"
                                id="address"
                                name="address"
                                rows="3"
                                required
                            ><?= htmlspecialchars($address) ?></textarea>
                        </div>

                        <button
                            type="submit"
                            name="place_order"
                            class="btn btn-success"
                        >
                            Xác nhận đặt hàng
                        </button>

                    </form>

                </div>
            </div>
        </div>

        <div class="col-lg-5">
            <div class="card shadow-sm">
                <div class="card-body">

                    <h2 class="h5 mb-3">
                        Đơn hàng của bạn
                    </h2>

                    <?php foreach ($cartItems as $item): ?>
                        <div
                            class="d-flex
                                   justify-content-between
                                   border-bottom
                                   py-2"
                        >
                            <div>
                                <strong>
                                    <?= htmlspecialchars($item['ProductName']) ?>
                                </strong>
                                <div class="small text-muted">
                                    <?= (int) $item['Quantity'] ?>
                                    ×
                                    <?= number_format(
                                        (float) $item['Price'],
                                        0,
                                        ',',
                                        '.'
                                    ) ?> đ
                                </div>
                            </div>

                            <div>
                                <?= number_format(
                                    (float) $item['Subtotal'],
                                    0,
                                    ',',
                                    '.'
                                ) ?> đ
                            </div>
                        </div>
                    <?php endforeach; ?>

                    <div
                        class="d-flex
                               justify-content-between
                               fw-bold
                               fs-5
                               pt-3"
                    >
                        <span>Tổng cộng</span>
                        <span>
                            <?= number_format(
                                (float) $total,
                                0,
                                ',',
                                '.'
                            ) ?> đ
                        </span>
                    </div>

                    <a
                        href="/cart.php"
                        class="btn btn-outline-secondary mt-3"
                    >
                        Quay lại giỏ hàng
                    </a>

                </div>
            </div>
        </div>

    </div>

</div>

<?php
require_once '/var/www/src/includes/frontend/footer.php';
```

------------------------------------------------------------------------

## 7. Hiểu cách Checkout xử lý dữ liệu

Trang `checkout.php` thực hiện hai giai đoạn khác nhau.

**Giai đoạn hiển thị:** ứng dụng đọc `ProductID` và `Quantity` từ Session, sau đó truy vấn lại MySQL để lấy tên, giá, tồn kho và hình ảnh hiện thời. Giá trị hiển thị không được lấy từ dữ liệu do trình duyệt tự gửi lên.

**Giai đoạn xác nhận đặt hàng:** ứng dụng mở transaction và truy vấn lại từng sản phẩm một lần nữa bằng:

```sql
SELECT
    ProductID,
    ProductName,
    Price,
    StockQuantity
FROM products
WHERE ProductID = ?
  AND IsActive = 1
FOR UPDATE
```

Việc kiểm tra lại là cần thiết vì dữ liệu có thể thay đổi trong khoảng thời gian từ lúc trang Checkout được mở đến lúc khách nhấn **Xác nhận đặt hàng**.

### 7.1. Vai trò của `FOR UPDATE`

`FOR UPDATE` yêu cầu MySQL khóa các mẩu tin sản phẩm được đọc trong transaction. Trong thời gian transaction chưa `COMMIT` hoặc `ROLLBACK`, các transaction khác không thể đồng thời sửa các mẩu tin đã bị khóa theo cách gây xung đột.

Trong bài này, nó giúp gắn thao tác **kiểm tra tồn kho** và **trừ tồn kho** vào cùng một transaction.

### 7.2. Không tin giá gửi từ trình duyệt

Khi tạo `orderdetail`, `UnitPrice` được lấy từ:

```php
$unitPrice = (float) $product['Price'];
```

Tức là giá được đọc lại từ MySQL trên server. Không gửi `Price` bằng hidden input rồi dùng trực tiếp để tính đơn hàng, vì dữ liệu phía trình duyệt có thể bị thay đổi.

### 7.3. Vì sao lưu `UnitPrice` trong `orderdetail`?

`products.Price` là giá hiện tại của sản phẩm. Giá này có thể được quản trị viên thay đổi sau khi khách đặt hàng.

`orderdetail.UnitPrice` lưu giá tại **thời điểm đơn hàng được tạo**. Nhờ đó, lịch sử đơn hàng vẫn giữ đúng giá trị giao dịch ban đầu.

------------------------------------------------------------------------

## 8. Transaction và tính toàn vẹn của đơn hàng

Transaction bắt đầu bằng:

```php
$conn->begin_transaction();
```

Sau đó chương trình lần lượt:

```text
1. Kiểm tra lại sản phẩm và tồn kho
2. INSERT customers
3. INSERT orders
4. INSERT orderdetail
5. UPDATE products.StockQuantity
```

Chỉ khi tất cả thành công mới chạy:

```php
$conn->commit();
```

Nếu xảy ra lỗi, khối `catch` thực hiện:

```php
$conn->rollback();
```

Vì vậy không xuất hiện tình trạng đã tạo đơn hàng nhưng chưa có chi tiết, hoặc đã trừ tồn kho nhưng đơn hàng lại không được tạo đầy đủ.

Một chi tiết quan trọng khác là:

```php
$_SESSION['cart'] = [];
```

chỉ được thực hiện **sau `commit()`**. Nếu transaction thất bại, giỏ hàng vẫn còn để khách có thể kiểm tra hoặc điều chỉnh.

------------------------------------------------------------------------

## 9. Tạo trang xác nhận đơn hàng

Tạo tập tin mới:

```text
public/order-success.php
```

Nhập toàn bộ nội dung:

```php
<?php

require_once '/var/www/src/config/session.php';
require_once '/var/www/src/config/database.php';

$pageTitle = 'Đặt hàng thành công';

$orderID = isset($_GET['id'])
    ? (int) $_GET['id']
    : 0;

if ($orderID <= 0) {
    header('Location: /');
    exit;
}

$sqlOrder = "
    SELECT
        o.OrderID,
        o.OrderDate,
        o.TotalAmount,
        o.Status,
        c.CustomerName,
        c.Phone,
        c.Address
    FROM orders o, customers c
    WHERE o.CustomerID = c.CustomerID
      AND o.OrderID = ?
";

$stmtOrder = $conn->prepare($sqlOrder);
$stmtOrder->bind_param('i', $orderID);
$stmtOrder->execute();
$orderResult = $stmtOrder->get_result();
$order = $orderResult->fetch_assoc();
$orderResult->free();
$stmtOrder->close();

if (!$order) {
    header('Location: /');
    exit;
}

$sqlDetail = "
    SELECT
        od.Quantity,
        od.UnitPrice,
        p.ProductCode,
        p.ProductName
    FROM orderdetail od, products p
    WHERE od.ProductID = p.ProductID
      AND od.OrderID = ?
    ORDER BY od.OrderDetailID
";

$stmtDetail = $conn->prepare($sqlDetail);
$stmtDetail->bind_param('i', $orderID);
$stmtDetail->execute();
$detailResult = $stmtDetail->get_result();

$orderItems = [];

while ($row = $detailResult->fetch_assoc()) {
    $row['Subtotal'] =
        (float) $row['UnitPrice']
        * (int) $row['Quantity'];

    $orderItems[] = $row;
}

$detailResult->free();
$stmtDetail->close();

require_once '/var/www/src/includes/frontend/header.php';
require_once '/var/www/src/includes/frontend/navbar.php';
?>

<div class="container py-4">

    <div class="alert alert-success">
        <h1 class="h4 mb-2">
            Đặt hàng thành công
        </h1>
        <p class="mb-0">
            Mã đơn hàng của bạn là
            <strong>#<?= (int) $order['OrderID'] ?></strong>.
        </p>
    </div>

    <div class="card shadow-sm mb-4">
        <div class="card-body">
            <h2 class="h5 mb-3">Thông tin đơn hàng</h2>

            <p>
                <strong>Khách hàng:</strong>
                <?= htmlspecialchars($order['CustomerName']) ?>
            </p>

            <p>
                <strong>Số điện thoại:</strong>
                <?= htmlspecialchars($order['Phone']) ?>
            </p>

            <p>
                <strong>Địa chỉ:</strong>
                <?= htmlspecialchars($order['Address']) ?>
            </p>

            <p>
                <strong>Ngày đặt:</strong>
                <?= htmlspecialchars($order['OrderDate']) ?>
            </p>

            <p class="mb-0">
                <strong>Trạng thái:</strong>
                <?= htmlspecialchars($order['Status']) ?>
            </p>
        </div>
    </div>

    <div class="card shadow-sm">
        <div class="card-body">

            <h2 class="h5 mb-3">Chi tiết sản phẩm</h2>

            <div class="table-responsive">
                <table class="table align-middle">
                    <thead>
                        <tr>
                            <th>Mã SP</th>
                            <th>Sản phẩm</th>
                            <th class="text-end">Đơn giá</th>
                            <th class="text-center">Số lượng</th>
                            <th class="text-end">Thành tiền</th>
                        </tr>
                    </thead>
                    <tbody>
                        <?php foreach ($orderItems as $item): ?>
                            <tr>
                                <td>
                                    <?= htmlspecialchars($item['ProductCode']) ?>
                                </td>
                                <td>
                                    <?= htmlspecialchars($item['ProductName']) ?>
                                </td>
                                <td class="text-end">
                                    <?= number_format(
                                        (float) $item['UnitPrice'],
                                        0,
                                        ',',
                                        '.'
                                    ) ?> đ
                                </td>
                                <td class="text-center">
                                    <?= (int) $item['Quantity'] ?>
                                </td>
                                <td class="text-end">
                                    <?= number_format(
                                        (float) $item['Subtotal'],
                                        0,
                                        ',',
                                        '.'
                                    ) ?> đ
                                </td>
                            </tr>
                        <?php endforeach; ?>
                    </tbody>
                    <tfoot>
                        <tr>
                            <th colspan="4" class="text-end">
                                Tổng cộng
                            </th>
                            <th class="text-end">
                                <?= number_format(
                                    (float) $order['TotalAmount'],
                                    0,
                                    ',',
                                    '.'
                                ) ?> đ
                            </th>
                        </tr>
                    </tfoot>
                </table>
            </div>

            <a
                href="/products.php"
                class="btn btn-primary"
            >
                Tiếp tục mua hàng
            </a>

        </div>
    </div>

</div>

<?php
require_once '/var/www/src/includes/frontend/footer.php';
```

Trang này lấy `OrderID` từ URL, sau đó truy vấn `orders` kết hợp `customers` và `orderdetail` kết hợp `products` để hiển thị lại dữ liệu đã được lưu trong cơ sở dữ liệu.

------------------------------------------------------------------------

## 10. Kiểm tra cú pháp PHP

Tại Command Prompt, chạy:

```cmd
docker compose exec web php -l /var/www/html/cart.php
docker compose exec web php -l /var/www/html/checkout.php
docker compose exec web php -l /var/www/html/order-success.php
```

Cả ba tập tin phải trả về:

```text
No syntax errors detected
```

------------------------------------------------------------------------

## 11. Kiểm thử luồng đặt hàng thành công

### 11.1. Chuẩn bị giỏ hàng

Mở trang sản phẩm, chọn một sản phẩm còn tồn kho và thêm vào giỏ.

Tại:

```text
http://localhost:8080/cart.php
```

kiểm tra sản phẩm, số lượng, đơn giá và tổng tiền.

Nhấn:

```text
Tiến hành đặt hàng
```

### 11.2. Kiểm tra Checkout

Trang `/checkout.php` phải hiển thị:

- họ tên;
- số điện thoại;
- địa chỉ;
- các sản phẩm trong giỏ;
- số lượng và đơn giá;
- tổng giá trị đơn hàng.

Nhập đầy đủ thông tin và nhấn **Xác nhận đặt hàng**.

Sau khi thành công, trình duyệt phải chuyển đến URL dạng:

```text
/order-success.php?id=...
```

Trang kết quả phải hiển thị mã đơn hàng, thông tin khách, trạng thái `Pending`, các sản phẩm và tổng tiền.

### 11.3. Kiểm tra dữ liệu MySQL

Kiểm tra đơn vừa tạo:

```sql
SELECT
    OrderID,
    OrderDate,
    TotalAmount,
    Status,
    CustomerID,
    EmployeeID,
    ShipperID
FROM orders
ORDER BY OrderID DESC
LIMIT 1;
```

Đơn mới cần có:

```text
Status      = Pending
EmployeeID  = NULL
ShipperID   = NULL
```

`OrderDate` phải là thời gian hiện tại theo múi giờ Việt Nam.

Dùng `OrderID` vừa nhận được để kiểm tra chi tiết:

```sql
SELECT *
FROM orderdetail
WHERE OrderID = <OrderID>;
```

Kiểm tra tồn kho của sản phẩm:

```sql
SELECT
    ProductID,
    ProductName,
    StockQuantity
FROM products
WHERE ProductID = <ProductID>;
```

Tồn kho phải giảm đúng bằng số lượng đã mua.

Cuối cùng quay lại website và kiểm tra Navbar. Sau khi đặt hàng thành công, giỏ hàng phải trở về:

```text
Giỏ hàng (0)
```

------------------------------------------------------------------------

## 12. Kiểm thử rollback khi tồn kho thay đổi

Đây là kiểm thử quan trọng để quan sát tác dụng của transaction.

### 12.1. Mở Checkout với sản phẩm còn hàng

Thêm một sản phẩm vào giỏ với số lượng hợp lệ và mở:

```text
http://localhost:8080/checkout.php
```

**Chưa nhấn Xác nhận đặt hàng.**

### 12.2. Giả lập tồn kho thay đổi

Trong MySQL, ghi nhận các ID hiện tại:

```sql
SELECT MAX(CustomerID) AS LastCustomerID
FROM customers;

SELECT MAX(OrderID) AS LastOrderID
FROM orders;

SELECT MAX(OrderDetailID) AS LastDetailID
FROM orderdetail;
```

Sau đó tạm thời đặt tồn kho của sản phẩm đang có trong Checkout về `0`:

```sql
UPDATE products
SET StockQuantity = 0
WHERE ProductID = <ProductID>;
```

Quay lại Checkout và nhấn **Xác nhận đặt hàng**.

Ứng dụng phải thông báo sản phẩm không đủ số lượng tồn kho. Giỏ hàng vẫn còn nguyên.

### 12.3. Xác nhận rollback

Chạy lại ba câu lệnh `MAX(...)` ở trên.

Nếu transaction hoạt động đúng:

```text
LastCustomerID    không tăng
LastOrderID       không tăng
LastDetailID      không tăng
```

Điều đó chứng minh các thao tác `INSERT` không bị lưu dở dang khi transaction thất bại.

Sau kiểm thử, khôi phục `StockQuantity` về giá trị đúng trước khi tiếp tục sử dụng ứng dụng.

> Trong tình huống thực tế, tồn kho có thể thay đổi vì một khách khác vừa mua sản phẩm. Kiểm thử trên chủ động tạo tình huống tương tự để xác nhận server luôn kiểm tra lại dữ liệu ngay trước khi tạo đơn hàng.

------------------------------------------------------------------------

## 13. Cấu trúc dự án sau Hands-on 12

Các thành phần chính liên quan đến Frontend và đặt hàng lúc này gồm:

```text
php-mysql-sales/
│
├── compose.yaml
│
├── database/
│   ├── schema.sql
│   └── seed.sql
│
├── public/
│   ├── index.php
│   ├── products.php
│   ├── product-detail.php
│   ├── cart.php
│   ├── checkout.php
│   └── order-success.php
│
└── src/
    ├── config/
    │   ├── database.php
    │   └── session.php
    │
    └── includes/
        └── frontend/
            ├── header.php
            ├── navbar.php
            └── footer.php
```

`cart.php` vẫn quản lý trạng thái mua hàng tạm thời trong Session. `checkout.php` là nơi chuyển dữ liệu từ giỏ hàng sang dữ liệu nghiệp vụ trong MySQL. `order-success.php` đọc lại đơn hàng đã lưu để xác nhận kết quả cho khách.

------------------------------------------------------------------------

## 14. Kiểm tra thay đổi trước khi lưu checkpoint

Kiểm tra trạng thái Git:

```cmd
git status
```

Các thay đổi của Hands-on này gồm:

```text
modified:   compose.yaml
modified:   database/schema.sql
modified:   public/cart.php
new:        public/checkout.php
new:        public/order-success.php
```

Kiểm tra các thay đổi trên tập tin đã tồn tại:

```cmd
git diff -- compose.yaml database/schema.sql public/cart.php
```

Kiểm tra lỗi khoảng trắng:

```cmd
git diff --check
```

Nếu `git diff --check` không hiển thị gì, không phát hiện lỗi khoảng trắng cần xử lý.

------------------------------------------------------------------------

## 15. Lưu checkpoint bằng Git

Stage đúng các tập tin của Hands-on 12:

```cmd
git add compose.yaml database/schema.sql public/cart.php public/checkout.php public/order-success.php
```

Kiểm tra:

```cmd
git status
```

Commit:

```cmd
git commit -m "Add guest checkout and order processing"
```

Push lên GitHub:

```cmd
git push origin main
```

Kiểm tra lần cuối:

```cmd
git status
```

Kết quả mong đợi:

```text
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

------------------------------------------------------------------------

## 16. Kết quả đạt được

Sau Hands-on 12, ứng dụng đã có một luồng mua hàng hoàn chỉnh ở mức cơ bản:

```text
Danh sách sản phẩm
        ↓
Chi tiết sản phẩm
        ↓
Giỏ hàng bằng Session
        ↓
Checkout
        ↓
Transaction MySQL
        ↓
Customer + Order + OrderDetail
        ↓
Cập nhật tồn kho
        ↓
Xác nhận đơn hàng
```

Điểm quan trọng nhất không chỉ là tạo thêm hai trang giao diện. Ứng dụng đã bắt đầu xử lý một **nghiệp vụ gồm nhiều bảng và nhiều thao tác phụ thuộc lẫn nhau**. Transaction bảo đảm đơn hàng không bị lưu ở trạng thái dở dang; `FOR UPDATE` hỗ trợ kiểm soát việc kiểm tra và cập nhật tồn kho; còn `UnitPrice` trong `orderdetail` giúp bảo toàn giá tại thời điểm giao dịch.

Ở Hands-on tiếp theo, dữ liệu đơn hàng đã lưu sẽ được đưa vào khu vực **Admin** để quản trị viên xem và xử lý trạng thái đơn hàng.
