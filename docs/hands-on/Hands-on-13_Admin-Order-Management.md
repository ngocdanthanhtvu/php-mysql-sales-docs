# Hands-on 13 -- Quản lý đơn hàng trong khu vực Admin

## Mục tiêu

Sau khi hoàn thành Hands-on này, bạn có thể:

- xây dựng chức năng quản lý đơn hàng trong khu vực Admin;
- đọc dữ liệu liên quan giữa `orders` và `customers` để lập danh sách đơn hàng;
- đọc dữ liệu `orderdetail` và `products` để hiển thị chi tiết một đơn hàng;
- sử dụng Prepared Statement khi truy vấn đơn hàng theo `OrderID`;
- mô hình hóa vòng đời đơn hàng bằng các trạng thái `Pending`, `Confirmed`, `Shipping`, `Completed`, `Cancelled`;
- giới hạn các phép chuyển trạng thái theo quy tắc nghiệp vụ;
- kiểm tra lại trạng thái đơn hàng ở phía server thay vì chỉ dựa vào giao diện;
- sử dụng transaction khi hủy đơn hàng;
- hoàn lại số lượng tồn kho khi đơn hàng chuyển sang `Cancelled`;
- sử dụng `SELECT ... FOR UPDATE` để khóa đơn hàng trong quá trình hủy;
- ngăn việc hoàn tồn kho nhiều lần cho cùng một đơn hàng;
- kiểm thử tính nhất quán giữa trạng thái đơn hàng và số lượng tồn kho.

> **Phạm vi Hands-on 13:** quản trị viên xem danh sách đơn hàng, xem chi tiết và cập nhật trạng thái đơn. Khi hủy đơn, hệ thống hoàn lại tồn kho. Chưa xây dựng đăng nhập/phân quyền, phân công nhân viên, chọn người giao hàng hoặc lịch sử thay đổi trạng thái.

------------------------------------------------------------------------

## 1. Chuẩn bị

Hands-on này tiếp tục trực tiếp từ **Hands-on 12**. Ứng dụng hiện đã có quy trình:

```text
Giỏ hàng
   ↓
Checkout
   ↓
customers
   ↓
orders
   ↓
orderdetail
   ↓
Cập nhật tồn kho
```

Khi khách đặt hàng thành công, đơn mới có trạng thái mặc định:

```text
Pending
```

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

Mở khu vực Admin:

```text
http://localhost:8080/admin/
```

------------------------------------------------------------------------

## 2. Phân tích nghiệp vụ quản lý đơn hàng

Khác với danh mục hoặc sản phẩm, đơn hàng **không được tạo thủ công trong Admin**. Đơn hàng được hình thành từ quá trình Checkout ở frontend.

Vì vậy, chức năng Admin trong Hands-on này tập trung vào:

```text
Danh sách đơn hàng
        ↓
Xem chi tiết
        ↓
Cập nhật trạng thái
        ↓
Theo dõi quá trình xử lý
```

Không xây dựng `create.php` và cũng không xóa trực tiếp đơn hàng khỏi cơ sở dữ liệu. Khi không tiếp tục xử lý một đơn, hệ thống chuyển đơn sang trạng thái `Cancelled` để vẫn giữ được dữ liệu lịch sử.

### 2.1. Vòng đời đơn hàng

Sử dụng năm trạng thái:

```text
Pending
Confirmed
Shipping
Completed
Cancelled
```

Quy tắc chuyển trạng thái:

```text
Pending ─────→ Confirmed ─────→ Shipping ─────→ Completed
   │               │                │
   └───────────────┴────────────────┴──────────→ Cancelled
```

Trong đó:

- `Pending`: đơn vừa được khách tạo;
- `Confirmed`: đơn đã được xác nhận;
- `Shipping`: đơn đang được giao;
- `Completed`: đơn đã hoàn tất;
- `Cancelled`: đơn đã hủy.

`Completed` và `Cancelled` là **trạng thái kết thúc**. Khi đơn đã ở một trong hai trạng thái này, hệ thống không cho chuyển sang trạng thái khác.

### 2.2. Hủy đơn và tồn kho

Ở Hands-on 12, số lượng tồn kho được giảm ngay khi đặt hàng thành công. Vì vậy, nếu Admin hủy đơn, lượng hàng đã giữ cho đơn đó phải được hoàn lại:

```text
Đặt 2 sản phẩm
StockQuantity - 2

Hủy đơn
StockQuantity + 2
```

Hai thao tác **hoàn tồn kho** và **đổi trạng thái sang `Cancelled`** phải cùng thành công. Đây là lý do transaction tiếp tục được sử dụng trong Hands-on 13.

------------------------------------------------------------------------

## 3. Bổ sung liên kết Đơn hàng vào Navbar Admin

Mở:

```text
src/includes/admin/navbar.php
```

Tìm mục Đơn hàng đang sử dụng đường dẫn:

```php
<a class="nav-link" href="/orders/">
    Đơn hàng
</a>
```

Thay bằng:

```php
<a class="nav-link" href="/admin/orders/">
    Đơn hàng
</a>
```

Sau thay đổi, menu Admin dẫn đúng đến khu vực quản lý đơn hàng:

```text
/admin/orders/
```

------------------------------------------------------------------------

## 4. Tạo trang danh sách đơn hàng

Tạo thư mục:

```text
public/admin/orders/
```

Trong thư mục này, tạo tập tin mới:

```text
public/admin/orders/index.php
```

Vì đây là tập tin mới, nhập toàn bộ nội dung sau:

```php
<?php

$pageTitle = 'Quản lý đơn hàng';

require_once '/var/www/src/config/database.php';

$sql = "
    SELECT
        o.OrderID,
        o.OrderDate,
        o.TotalAmount,
        o.Status,
        c.CustomerName,
        c.Phone
    FROM
        orders AS o,
        customers AS c
    WHERE
        o.CustomerID = c.CustomerID
    ORDER BY
        o.OrderID DESC
";

$result = $conn->query($sql);

require_once '/var/www/src/includes/admin/header.php';
require_once '/var/www/src/includes/admin/navbar.php';

?>

<div class="container mt-4">

    <div class="d-flex justify-content-between align-items-center mb-3">

        <h2>Quản lý đơn hàng</h2>

    </div>

    <div class="table-responsive">

        <table class="table table-bordered table-striped align-middle">

            <thead class="table-dark">
                <tr>
                    <th>Mã đơn</th>
                    <th>Ngày đặt</th>
                    <th>Khách hàng</th>
                    <th>Điện thoại</th>
                    <th>Tổng tiền</th>
                    <th>Trạng thái</th>
                    <th>Thao tác</th>
                </tr>
            </thead>

            <tbody>

            <?php while ($order = $result->fetch_assoc()): ?>

                <tr>

                    <td>
                        #<?= (int) $order['OrderID'] ?>
                    </td>

                    <td>
                        <?= htmlspecialchars($order['OrderDate']) ?>
                    </td>

                    <td>
                        <?= htmlspecialchars($order['CustomerName']) ?>
                    </td>

                    <td>
                        <?= htmlspecialchars($order['Phone'] ?? '') ?>
                    </td>

                    <td class="text-end">
                        <?= number_format(
                            (float) $order['TotalAmount'],
                            0,
                            ',',
                            '.'
                        ) ?> đ
                    </td>

                    <td>

                        <?php

                        $status = $order['Status'];

                        $badgeClass = match ($status) {
                            'Pending' => 'bg-warning text-dark',
                            'Confirmed' => 'bg-primary',
                            'Shipping' => 'bg-info text-dark',
                            'Completed' => 'bg-success',
                            'Cancelled' => 'bg-secondary',
                            default => 'bg-secondary'
                        };

                        ?>

                        <span class="badge <?= $badgeClass ?>">
                            <?= htmlspecialchars($status) ?>
                        </span>

                    </td>

                    <td>
                        <a
                            href="/admin/orders/detail.php?id=<?= (int) $order['OrderID'] ?>"
                            class="btn btn-sm btn-primary"
                        >
                            Xem
                        </a>
                    </td>

                </tr>

            <?php endwhile; ?>

            </tbody>

        </table>

    </div>

</div>

<?php

require_once '/var/www/src/includes/admin/footer.php';

$conn->close();
```

### 4.1. Phân tích truy vấn danh sách

Hai bảng được liên kết qua khóa:

```sql
FROM
    orders AS o,
    customers AS c
WHERE
    o.CustomerID = c.CustomerID
```

Cách viết này tiếp tục quy ước đã sử dụng trong các Hands-on trước: quan hệ khóa chính – khóa ngoại được biểu diễn bằng điều kiện trong `WHERE`.

Danh sách được sắp xếp:

```sql
ORDER BY o.OrderID DESC
```

nhằm đưa đơn mới nhất lên đầu.

### 4.2. Hiển thị trạng thái bằng Badge

Biểu thức `match` ánh xạ trạng thái với lớp Bootstrap:

```php
$badgeClass = match ($status) {
    'Pending' => 'bg-warning text-dark',
    'Confirmed' => 'bg-primary',
    'Shipping' => 'bg-info text-dark',
    'Completed' => 'bg-success',
    'Cancelled' => 'bg-secondary',
    default => 'bg-secondary'
};
```

Dữ liệu lưu trong MySQL vẫn là các giá trị trạng thái; lớp CSS chỉ phục vụ trình bày giao diện.

------------------------------------------------------------------------

## 5. Tạo trang chi tiết và xử lý trạng thái đơn hàng

Trong:

```text
public/admin/orders/
```

tạo tập tin mới:

```text
public/admin/orders/detail.php
```

Nhập toàn bộ nội dung sau:

```php
<?php

$pageTitle = 'Chi tiết đơn hàng';

require_once '/var/www/src/config/database.php';

$orderID = isset($_GET['id'])
    ? (int) $_GET['id']
    : 0;

if ($orderID <= 0) {
    header('Location: /admin/orders/');
    exit;
}

$sqlOrder = "
    SELECT
        o.OrderID,
        o.OrderDate,
        o.TotalAmount,
        o.Status,
        c.CustomerName,
        c.ContactName,
        c.Phone,
        c.Address,
        c.City,
        c.PostalCode,
        c.Country
    FROM
        orders AS o,
        customers AS c
    WHERE
        o.CustomerID = c.CustomerID
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
    header('Location: /admin/orders/');
    exit;
}

/*
 * Các trạng thái được phép chuyển tiếp.
 */
$allowedTransitions = [
    'Pending' => [
        'Confirmed',
        'Cancelled'
    ],
    'Confirmed' => [
        'Shipping',
        'Cancelled'
    ],
    'Shipping' => [
        'Completed',
        'Cancelled'
    ],
    'Completed' => [],
    'Cancelled' => []
];

/*
 * Xử lý cập nhật trạng thái.
 */
if ($_SERVER['REQUEST_METHOD'] === 'POST'
    && isset($_POST['update_status'])) {

    $newStatus = $_POST['status'] ?? '';

    try {

        $conn->begin_transaction();

        /*
         * Đọc và khóa đơn hàng.
         * Trạng thái phải được kiểm tra lại trong transaction.
         */
        $sqlLockOrder = "
            SELECT Status
            FROM orders
            WHERE OrderID = ?
            FOR UPDATE
        ";

        $stmtLockOrder = $conn->prepare($sqlLockOrder);
        $stmtLockOrder->bind_param('i', $orderID);
        $stmtLockOrder->execute();

        $lockResult = $stmtLockOrder->get_result();
        $lockedOrder = $lockResult->fetch_assoc();

        $lockResult->free();
        $stmtLockOrder->close();

        if (!$lockedOrder) {
            throw new Exception(
                'Không tìm thấy đơn hàng.'
            );
        }

        $currentStatus = $lockedOrder['Status'];

        $allowedStatuses =
            $allowedTransitions[$currentStatus] ?? [];

        if (!in_array(
            $newStatus,
            $allowedStatuses,
            true
        )) {
            throw new Exception(
                'Không thể chuyển sang trạng thái đã chọn.'
            );
        }

        /*
         * Nếu hủy đơn hàng, hoàn lại tồn kho
         * theo số lượng đã lưu trong orderdetail.
         */
        if ($newStatus === 'Cancelled') {

            $sqlItems = "
                SELECT
                    ProductID,
                    Quantity
                FROM orderdetail
                WHERE OrderID = ?
            ";

            $stmtItems = $conn->prepare($sqlItems);
            $stmtItems->bind_param('i', $orderID);
            $stmtItems->execute();

            $itemsResult = $stmtItems->get_result();

            $orderItems = [];

            while ($item = $itemsResult->fetch_assoc()) {
                $orderItems[] = $item;
            }

            $itemsResult->free();
            $stmtItems->close();

            $sqlRestoreStock = "
                UPDATE products
                SET StockQuantity =
                    StockQuantity + ?
                WHERE ProductID = ?
            ";

            $stmtRestoreStock =
                $conn->prepare($sqlRestoreStock);

            foreach ($orderItems as $item) {

                $quantity =
                    (int) $item['Quantity'];

                $productID =
                    (int) $item['ProductID'];

                $stmtRestoreStock->bind_param(
                    'ii',
                    $quantity,
                    $productID
                );

                $stmtRestoreStock->execute();

                if ($stmtRestoreStock->affected_rows !== 1) {
                    throw new Exception(
                        'Không thể hoàn lại tồn kho.'
                    );
                }
            }

            $stmtRestoreStock->close();
        }

        /*
         * Cập nhật trạng thái đơn hàng.
         */
        $sqlUpdate = "
            UPDATE orders
            SET Status = ?
            WHERE OrderID = ?
        ";

        $stmtUpdate = $conn->prepare($sqlUpdate);
        $stmtUpdate->bind_param(
            'si',
            $newStatus,
            $orderID
        );
        $stmtUpdate->execute();

        if ($stmtUpdate->affected_rows !== 1) {
            throw new Exception(
                'Không thể cập nhật trạng thái đơn hàng.'
            );
        }

        $stmtUpdate->close();

        $conn->commit();

        header(
            'Location: /admin/orders/detail.php?id='
            . $orderID
        );
        exit;

    } catch (Throwable $e) {

        $conn->rollback();

        $errorMessage = $e->getMessage();
    }
}

/*
 * Sau POST, đọc lại thông tin đơn hàng
 * để giao diện luôn phản ánh dữ liệu hiện tại.
 */
$stmtOrder = $conn->prepare($sqlOrder);
$stmtOrder->bind_param('i', $orderID);
$stmtOrder->execute();

$orderResult = $stmtOrder->get_result();
$order = $orderResult->fetch_assoc();

$orderResult->free();
$stmtOrder->close();

/*
 * Đọc chi tiết sản phẩm trong đơn hàng.
 */
$sqlDetail = "
    SELECT
        od.Quantity,
        od.UnitPrice,
        p.ProductCode,
        p.ProductName
    FROM
        orderdetail AS od,
        products AS p
    WHERE
        od.ProductID = p.ProductID
        AND od.OrderID = ?
    ORDER BY
        od.OrderDetailID
";

$stmtDetail = $conn->prepare($sqlDetail);
$stmtDetail->bind_param('i', $orderID);
$stmtDetail->execute();

$detailResult = $stmtDetail->get_result();

require_once '/var/www/src/includes/admin/header.php';
require_once '/var/www/src/includes/admin/navbar.php';

?>

<div class="container mt-4">

    <div
        class="d-flex
               justify-content-between
               align-items-center
               mb-3"
    >
        <h2>
            Chi tiết đơn hàng #<?= (int) $order['OrderID'] ?>
        </h2>

        <a
            href="/admin/orders/"
            class="btn btn-outline-secondary"
        >
            Quay lại
        </a>
    </div>

    <?php if (!empty($errorMessage)): ?>

        <div class="alert alert-danger">
            <?= htmlspecialchars($errorMessage) ?>
        </div>

    <?php endif; ?>

    <div class="row g-4 mb-4">

        <div class="col-md-6">

            <div class="card h-100">

                <div class="card-header">
                    <strong>Thông tin đơn hàng</strong>
                </div>

                <div class="card-body">

                    <p>
                        <strong>Mã đơn:</strong>
                        #<?= (int) $order['OrderID'] ?>
                    </p>

                    <p>
                        <strong>Ngày đặt:</strong>
                        <?= htmlspecialchars($order['OrderDate']) ?>
                    </p>

                    <p>
                        <strong>Trạng thái:</strong>
                        <?= htmlspecialchars($order['Status']) ?>
                    </p>

                    <p>
                        <strong>Tổng tiền:</strong>
                        <?= number_format(
                            (float) $order['TotalAmount'],
                            0,
                            ',',
                            '.'
                        ) ?> đ
                    </p>

                    <?php

                    $nextStatuses =
                        $allowedTransitions[$order['Status']]
                        ?? [];

                    ?>

                    <?php if (!empty($nextStatuses)): ?>

                        <hr>

                        <form method="post">

                            <div class="mb-3">

                                <label
                                    for="status"
                                    class="form-label"
                                >
                                    Cập nhật trạng thái
                                </label>

                                <select
                                    name="status"
                                    id="status"
                                    class="form-select"
                                    required
                                >

                                    <option value="">
                                        -- Chọn trạng thái --
                                    </option>

                                    <?php foreach (
                                        $nextStatuses as $status
                                    ): ?>

                                        <option
                                            value="<?= htmlspecialchars(
                                                $status
                                            ) ?>"
                                        >
                                            <?= htmlspecialchars(
                                                $status
                                            ) ?>
                                        </option>

                                    <?php endforeach; ?>

                                </select>

                            </div>

                            <button
                                type="submit"
                                name="update_status"
                                class="btn btn-primary"
                            >
                                Cập nhật
                            </button>

                        </form>

                    <?php else: ?>

                        <div class="alert alert-secondary mb-0">
                            Đơn hàng đã ở trạng thái kết thúc.
                        </div>

                    <?php endif; ?>

                </div>

            </div>

        </div>

        <div class="col-md-6">

            <div class="card h-100">

                <div class="card-header">
                    <strong>Thông tin khách hàng</strong>
                </div>

                <div class="card-body">

                    <p>
                        <strong>Khách hàng:</strong>
                        <?= htmlspecialchars(
                            $order['CustomerName']
                        ) ?>
                    </p>

                    <p>
                        <strong>Người liên hệ:</strong>
                        <?= htmlspecialchars(
                            $order['ContactName'] ?? ''
                        ) ?>
                    </p>

                    <p>
                        <strong>Điện thoại:</strong>
                        <?= htmlspecialchars(
                            $order['Phone'] ?? ''
                        ) ?>
                    </p>

                    <p>
                        <strong>Địa chỉ:</strong>
                        <?= htmlspecialchars(
                            $order['Address'] ?? ''
                        ) ?>
                    </p>

                    <p>
                        <strong>Thành phố:</strong>
                        <?= htmlspecialchars(
                            $order['City'] ?? ''
                        ) ?>
                    </p>

                    <p>
                        <strong>Mã bưu chính:</strong>
                        <?= htmlspecialchars(
                            $order['PostalCode'] ?? ''
                        ) ?>
                    </p>

                    <p class="mb-0">
                        <strong>Quốc gia:</strong>
                        <?= htmlspecialchars(
                            $order['Country'] ?? ''
                        ) ?>
                    </p>

                </div>

            </div>

        </div>

    </div>

    <div class="card">

        <div class="card-header">
            <strong>Sản phẩm trong đơn hàng</strong>
        </div>

        <div class="card-body">

            <div class="table-responsive">

                <table
                    class="table
                           table-bordered
                           align-middle
                           mb-0"
                >

                    <thead class="table-light">
                        <tr>
                            <th>Mã SP</th>
                            <th>Tên sản phẩm</th>
                            <th class="text-end">
                                Đơn giá
                            </th>
                            <th class="text-end">
                                Số lượng
                            </th>
                            <th class="text-end">
                                Thành tiền
                            </th>
                        </tr>
                    </thead>

                    <tbody>

                    <?php while (
                        $item = $detailResult->fetch_assoc()
                    ): ?>

                        <?php

                        $subtotal =
                            (float) $item['UnitPrice']
                            * (int) $item['Quantity'];

                        ?>

                        <tr>

                            <td>
                                <?= htmlspecialchars(
                                    $item['ProductCode']
                                ) ?>
                            </td>

                            <td>
                                <?= htmlspecialchars(
                                    $item['ProductName']
                                ) ?>
                            </td>

                            <td class="text-end">
                                <?= number_format(
                                    (float) $item['UnitPrice'],
                                    0,
                                    ',',
                                    '.'
                                ) ?> đ
                            </td>

                            <td class="text-end">
                                <?= (int) $item['Quantity'] ?>
                            </td>

                            <td class="text-end">
                                <?= number_format(
                                    $subtotal,
                                    0,
                                    ',',
                                    '.'
                                ) ?> đ
                            </td>

                        </tr>

                    <?php endwhile; ?>

                    </tbody>

                    <tfoot>
                        <tr>

                            <th
                                colspan="4"
                                class="text-end"
                            >
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

        </div>

    </div>

</div>

<?php

$detailResult->free();
$stmtDetail->close();

require_once '/var/www/src/includes/admin/footer.php';

$conn->close();
```

------------------------------------------------------------------------

## 6. Hiểu cách đọc chi tiết đơn hàng

Trang chi tiết nhận `OrderID` từ Query String:

```php
$orderID = isset($_GET['id'])
    ? (int) $_GET['id']
    : 0;
```

Ví dụ:

```text
/admin/orders/detail.php?id=6
```

Nếu ID không hợp lệ, người dùng được chuyển về danh sách:

```php
if ($orderID <= 0) {
    header('Location: /admin/orders/');
    exit;
}
```

Thông tin đơn hàng và khách hàng được đọc bằng Prepared Statement:

```sql
FROM
    orders AS o,
    customers AS c
WHERE
    o.CustomerID = c.CustomerID
    AND o.OrderID = ?
```

Dấu `?` không được ghép trực tiếp từ URL mà được truyền bằng:

```php
$stmtOrder->bind_param('i', $orderID);
```

Chi tiết sản phẩm được đọc từ:

```text
orderdetail → products
```

với điều kiện:

```sql
od.ProductID = p.ProductID
AND od.OrderID = ?
```

`UnitPrice` được lấy từ `orderdetail`, không lấy từ `products.Price`, vì đây là giá đã được lưu tại thời điểm khách đặt hàng.

------------------------------------------------------------------------

## 7. Kiểm soát vòng đời trạng thái

Mảng `$allowedTransitions` đóng vai trò mô tả các phép chuyển hợp lệ:

```php
$allowedTransitions = [
    'Pending' => [
        'Confirmed',
        'Cancelled'
    ],
    'Confirmed' => [
        'Shipping',
        'Cancelled'
    ],
    'Shipping' => [
        'Completed',
        'Cancelled'
    ],
    'Completed' => [],
    'Cancelled' => []
];
```

Ví dụ, khi đơn đang ở `Pending`, giao diện chỉ hiển thị:

```text
Confirmed
Cancelled
```

Sau khi chuyển sang `Confirmed`, chỉ còn:

```text
Shipping
Cancelled
```

Khi `Completed` hoặc `Cancelled`, mảng trạng thái tiếp theo rỗng. Form cập nhật không còn hiển thị và được thay bằng thông báo:

```text
Đơn hàng đã ở trạng thái kết thúc.
```

### 7.1. Không chỉ kiểm tra ở giao diện

Việc giới hạn `<select>` chưa đủ vì dữ liệu POST có thể được gửi mà không đi qua giao diện.

Server tiếp tục kiểm tra:

```php
if (!in_array(
    $newStatus,
    $allowedStatuses,
    true
)) {
    throw new Exception(
        'Không thể chuyển sang trạng thái đã chọn.'
    );
}
```

Do đó, quy tắc nghiệp vụ được bảo vệ ở phía server.

------------------------------------------------------------------------

## 8. Transaction khi hủy đơn hàng

Khi cập nhật trạng thái, transaction được bắt đầu bằng:

```php
$conn->begin_transaction();
```

Sau đó hệ thống đọc lại trạng thái hiện tại:

```sql
SELECT Status
FROM orders
WHERE OrderID = ?
FOR UPDATE
```

### 8.1. Vì sao phải đọc lại trạng thái?

Thông tin `$order` đã được đọc trước khi người dùng gửi form. Trong khoảng thời gian đó, trạng thái trong cơ sở dữ liệu có thể đã thay đổi.

Vì vậy, khi bắt đầu transaction, server không dựa hoàn toàn vào dữ liệu cũ mà đọc lại trạng thái trực tiếp từ MySQL.

### 8.2. Vai trò của `FOR UPDATE`

`FOR UPDATE` khóa mẩu tin đơn hàng trong transaction hiện tại.

Điều này đặc biệt quan trọng khi hủy đơn. Nếu hai yêu cầu cùng cố hủy một đơn, không được để cả hai cùng hoàn tồn kho:

```text
Yêu cầu A ──→ hoàn kho
Yêu cầu B ──→ hoàn kho lần nữa   ← sai
```

Với khóa và việc kiểm tra lại trạng thái, yêu cầu sau chỉ được xử lý khi transaction trước hoàn tất. Khi đó trạng thái đã là `Cancelled`, nên phép chuyển tiếp không còn hợp lệ.

------------------------------------------------------------------------

## 9. Hoàn tồn kho khi hủy đơn

Logic hoàn kho chỉ chạy khi:

```php
$newStatus === 'Cancelled'
```

Hệ thống đọc các sản phẩm của đơn:

```sql
SELECT
    ProductID,
    Quantity
FROM orderdetail
WHERE OrderID = ?
```

Sau đó hoàn lại từng số lượng:

```sql
UPDATE products
SET StockQuantity = StockQuantity + ?
WHERE ProductID = ?
```

Ví dụ, một đơn có:

```text
ProductID = 2
Quantity  = 3
```

khi hủy sẽ thực hiện tương đương:

```text
StockQuantity = StockQuantity + 3
```

Nếu cập nhật tồn kho không thành công, exception được phát sinh và transaction bị rollback.

------------------------------------------------------------------------

## 10. Vì sao cập nhật trạng thái cũng nằm trong transaction?

Sau khi hoàn kho, trạng thái mới được cập nhật:

```sql
UPDATE orders
SET Status = ?
WHERE OrderID = ?
```

Chỉ khi tất cả thao tác thành công mới thực hiện:

```php
$conn->commit();
```

Nếu có lỗi:

```php
$conn->rollback();
```

Nhờ đó không xảy ra hai tình huống không nhất quán:

```text
Đơn = Cancelled
nhưng tồn kho chưa được hoàn
```

hoặc:

```text
Tồn kho đã được hoàn
nhưng đơn vẫn chưa Cancelled
```

Toàn bộ nghiệp vụ hủy đơn được xem như **một đơn vị công việc**.

------------------------------------------------------------------------

## 11. Kiểm tra cú pháp PHP

Từ thư mục dự án, chạy:

```cmd
docker compose exec web php -l /var/www/html/admin/orders/index.php
```

Kết quả mong đợi:

```text
No syntax errors detected in /var/www/html/admin/orders/index.php
```

Tiếp tục:

```cmd
docker compose exec web php -l /var/www/html/admin/orders/detail.php
```

Kết quả mong đợi:

```text
No syntax errors detected in /var/www/html/admin/orders/detail.php
```

------------------------------------------------------------------------

## 12. Kiểm thử danh sách và chi tiết đơn hàng

### 12.1. Kiểm tra danh sách

Mở:

```text
http://localhost:8080/admin/orders/
```

Kiểm tra:

- các đơn hàng được hiển thị;
- đơn mới hơn nằm phía trên;
- có ngày đặt, khách hàng, điện thoại, tổng tiền và trạng thái;
- trạng thái được hiển thị bằng Badge;
- mỗi đơn có nút **Xem**.

### 12.2. Kiểm tra chi tiết

Bấm **Xem** tại một đơn hàng.

Trang chi tiết phải hiển thị:

- mã đơn;
- ngày đặt;
- trạng thái;
- tổng tiền;
- thông tin khách hàng;
- các sản phẩm thuộc đơn;
- đơn giá tại thời điểm đặt;
- số lượng;
- thành tiền từng dòng.

Thử URL không tồn tại:

```text
http://localhost:8080/admin/orders/detail.php?id=99999
```

Ứng dụng phải chuyển về:

```text
/admin/orders/
```

------------------------------------------------------------------------

## 13. Kiểm thử vòng đời trạng thái

Chọn một đơn `Pending` và thực hiện:

```text
Pending → Confirmed
```

Kiểm tra trực tiếp trong MySQL:

```cmd
docker compose exec db mysql -u root -p
```

Sau khi đăng nhập:

```sql
USE ql_banhang;
```

Kiểm tra:

```sql
SELECT
    OrderID,
    Status,
    TotalAmount
FROM orders
WHERE OrderID = <OrderID>;
```

Trạng thái phải là:

```text
Confirmed
```

Tiếp tục trên giao diện:

```text
Confirmed → Shipping → Completed
```

Khi đã `Completed`, form cập nhật phải biến mất và trang hiển thị:

```text
Đơn hàng đã ở trạng thái kết thúc.
```

`Completed` không được quay trở lại `Pending`, `Confirmed` hoặc `Shipping`.

------------------------------------------------------------------------

## 14. Kiểm thử hủy đơn và hoàn tồn kho

Đây là phép thử quan trọng nhất của Hands-on 13.

### 14.1. Ghi nhận tồn kho ban đầu

Chọn một sản phẩm, ví dụ `ProductID = 2`:

```sql
SELECT
    ProductID,
    ProductName,
    StockQuantity
FROM products
WHERE ProductID = 2;
```

Gọi số lượng hiện tại là **A**.

### 14.2. Tạo một đơn hàng mới

Trên frontend, thêm **1 sản phẩm** vào giỏ và đặt hàng thành công.

Kiểm tra lại:

```sql
SELECT
    ProductID,
    ProductName,
    StockQuantity
FROM products
WHERE ProductID = 2;
```

Kết quả phải là:

```text
A - 1
```

### 14.3. Hủy đơn trong Admin

Vào:

```text
/admin/orders/
```

mở đơn vừa tạo và chuyển:

```text
Pending → Cancelled
```

Kiểm tra lại tồn kho:

```sql
SELECT
    ProductID,
    ProductName,
    StockQuantity
FROM products
WHERE ProductID = 2;
```

Kết quả phải trở về:

```text
A
```

Kiểm tra trạng thái đơn:

```sql
SELECT
    OrderID,
    Status
FROM orders
ORDER BY OrderID DESC
LIMIT 1;
```

Kết quả phải có:

```text
Status = Cancelled
```

Ta đã kiểm chứng chuỗi:

```text
Tồn kho A
   ↓ đặt 1 sản phẩm
A - 1
   ↓ hủy đơn
A
```

### 14.4. So sánh với xác nhận đơn

Tạo một đơn mới khác và chuyển:

```text
Pending → Confirmed
```

Tồn kho **không được hoàn lại**. Số lượng vẫn giữ mức đã giảm khi khách đặt hàng.

Điều này xác nhận hai nhánh nghiệp vụ khác nhau:

```text
Đặt hàng
   ↓
Tồn kho giảm
   ↓
┌──────────────────┬──────────────────┐
│ Cancelled        │ Confirmed        │
│                  │                  │
│ hoàn tồn kho     │ không hoàn kho   │
└──────────────────┴──────────────────┘
```

------------------------------------------------------------------------

## 15. Cấu trúc dự án sau Hands-on 13

Các thành phần liên quan trực tiếp đến Hands-on này:

```text
php-mysql-sales/
├── public/
│   └── admin/
│       └── orders/
│           ├── index.php
│           └── detail.php
│
└── src/
    └── includes/
        └── admin/
            └── navbar.php
```

Vai trò:

```text
navbar.php
   ↓
/admin/orders/
   ↓
index.php
   ↓
Danh sách orders + customers
   ↓
Xem
   ↓
detail.php?id=...
   ├── orders + customers
   ├── orderdetail + products
   └── cập nhật trạng thái
          ↓
       Cancelled?
       ├── Không → cập nhật trạng thái
       └── Có   → transaction + hoàn tồn kho
```

------------------------------------------------------------------------

## 16. Kiểm tra thay đổi trước khi lưu checkpoint

Kiểm tra trạng thái Git:

```cmd
git status
```

Các thay đổi của Hands-on này gồm:

```text
modified:
    src/includes/admin/navbar.php

new:
    public/admin/orders/index.php
    public/admin/orders/detail.php
```

Kiểm tra lỗi khoảng trắng:

```cmd
git diff --check
```

Nếu lệnh không xuất thông báo, phần thay đổi không có lỗi whitespace mà Git phát hiện.

Kiểm tra cú pháp lần cuối:

```cmd
docker compose exec web php -l /var/www/html/admin/orders/index.php
docker compose exec web php -l /var/www/html/admin/orders/detail.php
```

Chỉ lưu checkpoint sau khi:

- danh sách đơn hàng hoạt động;
- xem chi tiết hoạt động;
- chuyển trạng thái đúng quy tắc;
- `Completed` và `Cancelled` là trạng thái kết thúc;
- hủy đơn hoàn đúng tồn kho;
- xác nhận đơn không hoàn tồn kho;
- hai tập tin PHP không có lỗi cú pháp.

------------------------------------------------------------------------

## 17. Lưu checkpoint bằng Git

Stage các thay đổi:

```cmd
git add src/includes/admin/navbar.php public/admin/orders/
```

Kiểm tra:

```cmd
git status
```

Commit:

```cmd
git commit -m "Add admin order management"
```

Đẩy lên GitHub:

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

## 18. Kết quả đạt được

Sau Hands-on 13, ứng dụng đã có một quy trình quản lý đơn hàng cơ bản ở phía Admin:

```text
Khách hàng đặt hàng
        ↓
Pending
        ↓
Admin xem đơn
        ↓
┌─────────────────────────────────────┐
│                                     │
▼                                     ▼
Confirmed                         Cancelled
   ↓                                  ↓
Shipping                         Hoàn tồn kho
   ↓                                  ↓
Completed                       Kết thúc
   ↓
Kết thúc
```

Điểm quan trọng của Hands-on này không chỉ là tạo thêm hai trang Admin. Ứng dụng đã bắt đầu thể hiện **quy tắc nghiệp vụ** trong mã nguồn:

- trạng thái không được chuyển tùy ý;
- trạng thái được kiểm tra lại ở phía server;
- đơn hoàn tất không quay lại trạng thái trước;
- đơn hủy không bị xóa khỏi cơ sở dữ liệu;
- hàng của đơn hủy được trả lại tồn kho;
- transaction bảo đảm trạng thái đơn và tồn kho được cập nhật nhất quán;
- `FOR UPDATE` giúp tránh xử lý hủy lặp trên cùng một đơn hàng.

Đây là nền tảng để tiếp tục phát triển các chức năng có người dùng và quyền truy cập ở Hands-on tiếp theo.
