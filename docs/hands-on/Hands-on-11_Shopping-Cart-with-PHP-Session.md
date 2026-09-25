# Hands-on 11 -- Xây dựng giỏ hàng với PHP Session

## Mục tiêu

Sau khi hoàn thành Hands-on này, bạn có thể:

-   giải thích vai trò của PHP Session trong việc lưu trạng thái giữa nhiều request;
-   khởi tạo Session tập trung trong một tập tin cấu hình dùng chung;
-   lưu giỏ hàng theo cấu trúc `ProductID => Quantity`;
-   thêm sản phẩm từ trang chi tiết vào giỏ hàng;
-   cộng dồn số lượng khi cùng một sản phẩm được thêm nhiều lần;
-   giới hạn số lượng mua theo tồn kho hiện tại;
-   xây dựng trang giỏ hàng và lấy thông tin sản phẩm hiện thời từ MySQL;
-   tính thành tiền cho từng sản phẩm và tổng giá trị giỏ hàng;
-   cập nhật số lượng và xóa sản phẩm khỏi giỏ hàng;
-   hiển thị tổng số lượng sản phẩm trong giỏ trên thanh điều hướng Frontend;
-   kiểm thử toàn bộ luồng giỏ hàng và lưu checkpoint bằng Git.

> **Phạm vi Hands-on 11:** giỏ hàng được lưu bằng PHP Session. Chưa tạo đơn hàng, chưa lưu giỏ hàng vào cơ sở dữ liệu và chưa thực hiện thanh toán.

------------------------------------------------------------------------

## 1. Chuẩn bị

Hands-on này tiếp tục trực tiếp từ **Hands-on 10**. Ở thời điểm bắt đầu, Frontend đã có:

-   trang danh sách sản phẩm `/products.php`;
-   tìm kiếm, lọc và phân trang sản phẩm;
-   trang chi tiết `/product-detail.php?id=...`;
-   thanh điều hướng dùng chung trong `src/includes/frontend/navbar.php`.

Mở **Command Prompt** và chuyển đến thư mục dự án:

``` cmd
cd /d D:\PTUDW-ST-2026\php-mysql-sales
```

Kiểm tra trạng thái Git:

``` cmd
git status
```

Nên bắt đầu khi Git hiển thị:

``` text
nothing to commit, working tree clean
```

Kiểm tra container:

``` cmd
docker compose ps
```

Nếu các container chưa chạy:

``` cmd
docker compose up -d
```

Mở ứng dụng:

``` text
http://localhost:8080/
```

------------------------------------------------------------------------

## 2. Thiết kế giỏ hàng bằng PHP Session

HTTP không tự ghi nhớ sản phẩm mà người dùng đã chọn giữa các request. Trong bài này, ta sử dụng **PHP Session** để lưu trạng thái giỏ hàng trong phiên làm việc của người dùng.

Giỏ hàng được tổ chức theo dạng:

``` php
$_SESSION['cart'] = [
    ProductID => Quantity
];
```

Ví dụ:

``` php
$_SESSION['cart'] = [
    3 => 2,
    7 => 1
];
```

Ý nghĩa:

``` text
ProductID = 3  → số lượng 2
ProductID = 7  → số lượng 1
```

Ta **không lưu tên, giá hoặc hình ảnh sản phẩm vào Session**. Khi hiển thị giỏ hàng, ứng dụng dùng `ProductID` để lấy lại thông tin hiện thời từ MySQL.

Cách tổ chức này giúp Session gọn và tránh giữ các thông tin sản phẩm có thể đã thay đổi trong cơ sở dữ liệu.

Luồng xử lý chính:

``` text
Chi tiết sản phẩm
        │
        │ POST: ProductID + Quantity
        ▼
PHP Session
$_SESSION['cart']
        │
        ▼
cart.php
        │
        ├── lấy thông tin sản phẩm từ MySQL
        ├── tính thành tiền
        ├── cập nhật số lượng
        └── xóa sản phẩm
```

------------------------------------------------------------------------

## 3. Tạo tập tin khởi tạo Session

Tạo tập tin mới:

``` text
src/config/session.php
```

Mở bằng VS Code:

``` cmd
code src\config\session.php
```

Nhập:

``` php
<?php

if (session_status() === PHP_SESSION_NONE) {
    session_start();
}
```

### Giải thích

`session_status()` cho biết trạng thái hiện tại của Session.

Điều kiện:

``` php
session_status() === PHP_SESSION_NONE
```

đảm bảo `session_start()` chỉ được gọi khi Session chưa được khởi tạo.

Ta đặt xử lý này trong `src/config/session.php` thay vì `header.php` hoặc `navbar.php` vì:

-   `session.php` chịu trách nhiệm khởi tạo trạng thái phiên;
-   `header.php` và `navbar.php` tập trung vào giao diện;
-   Session có thể được khởi tạo **trước khi HTML được gửi về trình duyệt**.

Kiểm tra cú pháp:

``` cmd
docker compose exec web php -l /var/www/src/config/session.php
```

Kết quả mong đợi:

``` text
No syntax errors detected
```

------------------------------------------------------------------------

## 4. Khởi tạo Session trên trang chi tiết sản phẩm

Mở:

``` cmd
code public\product-detail.php
```

Ở đầu tập tin, trước kết nối cơ sở dữ liệu, thêm:

``` php
require_once '/var/www/src/config/session.php';
```

Phần đầu tập tin trở thành:

``` php
<?php

require_once '/var/www/src/config/session.php';
require_once '/var/www/src/config/database.php';
```

Session phải được khởi tạo trước phần HTML để ứng dụng có thể đọc và thay đổi `$_SESSION` an toàn.

------------------------------------------------------------------------

## 5. Xử lý thêm sản phẩm vào giỏ hàng

Trong `public/product-detail.php`, sau khi đã lấy được `$product` và kiểm tra sản phẩm tồn tại, nhưng **trước phần truy vấn hình ảnh**, thêm:

``` php
$cartMessage = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST'
    && isset($_POST['add_to_cart'])) {

    $quantity = isset($_POST['quantity'])
        ? (int) $_POST['quantity']
        : 1;

    if ($quantity < 1) {
        $quantity = 1;
    }

    $stockQuantity = (int) $product['StockQuantity'];

    if ($stockQuantity <= 0) {

        $cartMessage = 'Sản phẩm hiện đã hết hàng.';

    } else {

        $currentQuantity =
            $_SESSION['cart'][$productID] ?? 0;

        $newQuantity =
            $currentQuantity + $quantity;

        if ($newQuantity > $stockQuantity) {
            $newQuantity = $stockQuantity;
        }

        $_SESSION['cart'][$productID] =
            $newQuantity;

        $cartMessage =
            'Đã thêm sản phẩm vào giỏ hàng.';
    }
}
```

### Phân tích xử lý

Khi người dùng gửi biểu mẫu bằng `POST`, chương trình lấy số lượng:

``` php
$quantity = isset($_POST['quantity'])
    ? (int) $_POST['quantity']
    : 1;
```

Nếu dữ liệu nhỏ hơn `1`, chương trình đưa về giá trị tối thiểu:

``` php
if ($quantity < 1) {
    $quantity = 1;
}
```

Số lượng hiện có của cùng sản phẩm được lấy bằng:

``` php
$currentQuantity =
    $_SESSION['cart'][$productID] ?? 0;
```

Sau đó cộng số lượng mới:

``` php
$newQuantity =
    $currentQuantity + $quantity;
```

Nếu tổng vượt tồn kho, chỉ giữ tối đa bằng `StockQuantity`:

``` php
if ($newQuantity > $stockQuantity) {
    $newQuantity = $stockQuantity;
}
```

Cuối cùng cập nhật Session:

``` php
$_SESSION['cart'][$productID] =
    $newQuantity;
```

Như vậy, thêm lại cùng một sản phẩm **không tạo một dòng giỏ hàng mới** mà làm tăng số lượng của sản phẩm đó.

------------------------------------------------------------------------

## 6. Tạo giao diện thêm vào giỏ hàng

Trong phần thông tin sản phẩm của `public/product-detail.php`, sau danh sách thông tin (`</dl>`) và trước phần mô tả sản phẩm, thêm:

``` php
<?php if ($cartMessage !== ''): ?>

    <div class="alert alert-info">
        <?= htmlspecialchars($cartMessage) ?>
    </div>

<?php endif; ?>

<?php if ((int) $product['StockQuantity'] > 0): ?>

    <form method="post" class="mb-4">

        <div class="row g-3 align-items-end">

            <div class="col-auto">

                <label
                    for="quantity"
                    class="form-label"
                >
                    Số lượng
                </label>

                <input
                    type="number"
                    name="quantity"
                    id="quantity"
                    class="form-control"
                    value="1"
                    min="1"
                    max="<?= (int) $product['StockQuantity'] ?>"
                    style="width: 100px;"
                >

            </div>

            <div class="col-auto">

                <button
                    type="submit"
                    name="add_to_cart"
                    class="btn btn-primary"
                >
                    Thêm vào giỏ hàng
                </button>

            </div>

        </div>

    </form>

<?php else: ?>

    <div class="alert alert-warning">
        Sản phẩm hiện đã hết hàng.
    </div>

<?php endif; ?>
```

Ở đây:

-   `min="1"` không cho nhập số lượng nhỏ hơn 1 trên giao diện;
-   `max` lấy từ `StockQuantity` của sản phẩm;
-   `name="add_to_cart"` giúp PHP nhận biết thao tác thêm vào giỏ;
-   `htmlspecialchars()` được dùng khi hiển thị thông báo.

### Kiểm tra bước thêm sản phẩm

Kiểm tra cú pháp:

``` cmd
docker compose exec web php -l /var/www/html/product-detail.php
```

Mở một sản phẩm, ví dụ:

``` text
http://localhost:8080/product-detail.php?id=3
```

Kiểm thử:

1. Chọn số lượng `1` và nhấn **Thêm vào giỏ hàng**.
2. Thêm lại chính sản phẩm đó với số lượng khác.
3. Thử nhập số lượng bằng tồn kho tối đa.

Kết quả cần đạt:

-   xuất hiện thông báo đã thêm sản phẩm;
-   thêm lại cùng sản phẩm làm tăng số lượng;
-   số lượng lưu trong giỏ không vượt quá tồn kho.

------------------------------------------------------------------------

## 7. Tạo trang giỏ hàng

Tạo tập tin mới:

``` text
public/cart.php
```

Mở:

``` cmd
code public\cart.php
```

Vì đây là **tập tin mới**, nhập toàn bộ nội dung sau:

``` php
<?php

require_once '/var/www/src/config/session.php';
require_once '/var/www/src/config/database.php';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    if (isset($_POST['update_cart'])) {

        $quantities = $_POST['quantities'] ?? [];

        foreach ($quantities as $productID => $quantity) {

            $productID = (int) $productID;
            $quantity = (int) $quantity;

            if ($productID <= 0) {
                continue;
            }

            if ($quantity <= 0) {
                unset($_SESSION['cart'][$productID]);
                continue;
            }

            $sqlStock = "
                SELECT StockQuantity
                FROM products
                WHERE ProductID = ?
                  AND IsActive = 1
            ";

            $stmtStock = $conn->prepare($sqlStock);
            $stmtStock->bind_param('i', $productID);
            $stmtStock->execute();

            $stockResult = $stmtStock->get_result();
            $stockRow = $stockResult->fetch_assoc();

            $stockResult->free();
            $stmtStock->close();

            if (!$stockRow) {
                unset($_SESSION['cart'][$productID]);
                continue;
            }

            $stockQuantity =
                (int) $stockRow['StockQuantity'];

            if ($stockQuantity <= 0) {
                unset($_SESSION['cart'][$productID]);
                continue;
            }

            $_SESSION['cart'][$productID] =
                min($quantity, $stockQuantity);
        }

        header('Location: /cart.php');
        exit;
    }

    if (isset($_POST['remove_product'])) {

        $productID =
            (int) $_POST['remove_product'];

        if ($productID > 0) {
            unset($_SESSION['cart'][$productID]);
        }

        header('Location: /cart.php');
        exit;
    }
}

$cart = $_SESSION['cart'] ?? [];

$cartItems = [];
$totalAmount = 0;

if (!empty($cart)) {

    foreach ($cart as $productID => $quantity) {

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

            WHERE
                p.ProductID = ?
                AND p.IsActive = 1
        ";

        $stmt = $conn->prepare($sql);
        $stmt->bind_param('i', $productID);
        $stmt->execute();

        $result = $stmt->get_result();
        $product = $result->fetch_assoc();

        $result->free();
        $stmt->close();

        if ($product) {

            $quantity = (int) $quantity;

            $subtotal =
                (float) $product['Price'] * $quantity;

            $product['Quantity'] = $quantity;
            $product['Subtotal'] = $subtotal;

            $cartItems[] = $product;
            $totalAmount += $subtotal;
        }
    }
}

$pageTitle = 'Giỏ hàng';

require_once '/var/www/src/includes/frontend/header.php';
require_once '/var/www/src/includes/frontend/navbar.php';

?>

<main class="container py-5">

    <div class="mb-4">

        <h1>Giỏ hàng</h1>

        <p class="text-muted">
            Các sản phẩm bạn đã chọn.
        </p>

    </div>

    <?php if (empty($cartItems)): ?>

        <div class="alert alert-info">
            Giỏ hàng của bạn đang trống.
        </div>

        <a
            href="/products.php"
            class="btn btn-primary"
        >
            Tiếp tục mua hàng
        </a>

    <?php else: ?>

        <form method="post" action="/cart.php">

            <div class="table-responsive">

                <table class="table align-middle">

                    <thead>

                        <tr>
                            <th>Sản phẩm</th>
                            <th class="text-end">Đơn giá</th>
                            <th class="text-center">Số lượng</th>
                            <th class="text-end">Thành tiền</th>
                            <th class="text-center">Thao tác</th>
                        </tr>

                    </thead>

                    <tbody>

                        <?php foreach ($cartItems as $item): ?>

                            <tr>

                                <td>

                                    <div
                                        class="d-flex
                                               align-items-center
                                               gap-3"
                                    >

                                        <?php if (!empty($item['ImageFile'])): ?>

                                            <img
                                                src="/uploads/products/<?=
                                                    htmlspecialchars(
                                                        $item['ImageFile']
                                                    )
                                                ?>"
                                                alt="<?=
                                                    htmlspecialchars(
                                                        $item['ProductName']
                                                    )
                                                ?>"
                                                style="
                                                    width: 80px;
                                                    height: 80px;
                                                    object-fit: contain;
                                                "
                                            >

                                        <?php endif; ?>

                                        <div>

                                            <strong>
                                                <?=
                                                    htmlspecialchars(
                                                        $item['ProductName']
                                                    )
                                                ?>
                                            </strong>

                                            <div class="text-muted small">
                                                <?=
                                                    htmlspecialchars(
                                                        $item['ProductCode']
                                                    )
                                                ?>
                                            </div>

                                        </div>

                                    </div>

                                </td>

                                <td class="text-end">

                                    <?=
                                        number_format(
                                            (float) $item['Price'],
                                            0,
                                            ',',
                                            '.'
                                        )
                                    ?> đ

                                </td>

                                <td class="text-center">

                                    <input
                                        type="number"
                                        name="quantities[<?=
                                            (int) $item['ProductID']
                                        ?>]"
                                        value="<?=
                                            (int) $item['Quantity']
                                        ?>"
                                        min="1"
                                        max="<?=
                                            (int) $item['StockQuantity']
                                        ?>"
                                        class="form-control mx-auto"
                                        style="width: 90px;"
                                    >

                                </td>

                                <td class="text-end fw-bold">

                                    <?=
                                        number_format(
                                            (float) $item['Subtotal'],
                                            0,
                                            ',',
                                            '.'
                                        )
                                    ?> đ

                                </td>

                                <td class="text-center">

                                    <button
                                        type="submit"
                                        name="remove_product"
                                        value="<?=
                                            (int) $item['ProductID']
                                        ?>"
                                        class="btn btn-sm btn-outline-danger"
                                        formnovalidate
                                    >
                                        Xóa
                                    </button>

                                </td>

                            </tr>

                        <?php endforeach; ?>

                    </tbody>

                    <tfoot>

                        <tr>

                            <th
                                colspan="3"
                                class="text-end"
                            >
                                Tổng cộng
                            </th>

                            <th class="text-end fs-5">

                                <?=
                                    number_format(
                                        $totalAmount,
                                        0,
                                        ',',
                                        '.'
                                    )
                                ?> đ

                            </th>

                            <th></th>

                        </tr>

                    </tfoot>

                </table>

            </div>

            <div class="d-flex justify-content-end mt-3">

                <button
                    type="submit"
                    name="update_cart"
                    class="btn btn-primary"
                >
                    Cập nhật giỏ hàng
                </button>

            </div>

        </form>

        <div class="mt-4">

            <a
                href="/products.php"
                class="btn btn-outline-secondary"
            >
                Tiếp tục mua hàng
            </a>

        </div>

    <?php endif; ?>

</main>

<?php

require_once '/var/www/src/includes/frontend/footer.php';
```

Kiểm tra cú pháp:

``` cmd
docker compose exec web php -l /var/www/html/cart.php
```

------------------------------------------------------------------------

## 8. Hiểu cách trang giỏ hàng lấy dữ liệu

Dữ liệu trong Session chỉ có:

``` text
ProductID → Quantity
```

Vì vậy, với từng sản phẩm trong giỏ:

``` php
foreach ($cart as $productID => $quantity)
```

ứng dụng truy vấn lại bảng `products` bằng `ProductID`.

Điều kiện:

``` sql
WHERE
    p.ProductID = ?
    AND p.IsActive = 1
```

đảm bảo chỉ lấy sản phẩm đang hoạt động.

Ảnh chính được lấy bằng truy vấn con:

``` sql
(
    SELECT pi.ImageFile
    FROM product_images pi
    WHERE pi.ProductID = p.ProductID
      AND pi.IsPrimary = 1
    LIMIT 1
) AS ImageFile
```

Thành tiền của một sản phẩm:

``` php
$subtotal =
    (float) $product['Price'] * $quantity;
```

Tổng giỏ hàng được cộng dồn:

``` php
$totalAmount += $subtotal;
```

### Một kết quả -- một lần `get_result()`

Sau mỗi `execute()`, kết quả được lấy và giải phóng trước khi statement đóng:

``` php
$result = $stmt->get_result();
$product = $result->fetch_assoc();

$result->free();
$stmt->close();
```

Khi kiểm tra tồn kho lúc cập nhật giỏ hàng, ta cũng thực hiện cùng nguyên tắc:

``` php
$stockResult = $stmtStock->get_result();
$stockRow = $stockResult->fetch_assoc();

$stockResult->free();
$stmtStock->close();
```

------------------------------------------------------------------------

## 9. Cập nhật số lượng trong giỏ hàng

Mỗi ô số lượng có tên theo `ProductID`:

``` php
name="quantities[<?= (int) $item['ProductID'] ?>]"
```

Ví dụ trình duyệt có thể gửi:

``` text
quantities[3] = 2
quantities[7] = 4
```

PHP nhận toàn bộ bằng:

``` php
$quantities = $_POST['quantities'] ?? [];
```

và duyệt:

``` php
foreach ($quantities as $productID => $quantity)
```

Trước khi cập nhật Session, chương trình truy vấn lại `StockQuantity`. Nếu số lượng người dùng nhập lớn hơn tồn kho:

``` php
$_SESSION['cart'][$productID] =
    min($quantity, $stockQuantity);
```

Nhờ đó, không thể cập nhật giỏ vượt quá số lượng hiện có trong cơ sở dữ liệu.

Sau khi xử lý, trang chuyển hướng về chính nó:

``` php
header('Location: /cart.php');
exit;
```

Cách này giúp request hiển thị sau cập nhật trở lại phương thức `GET`.

### Kiểm thử cập nhật

1. Thêm ít nhất hai sản phẩm vào giỏ.
2. Mở:

``` text
http://localhost:8080/cart.php
```

3. Thay đổi số lượng một sản phẩm.
4. Nhấn **Cập nhật giỏ hàng**.
5. Kiểm tra lại thành tiền và tổng cộng.
6. Nhấn `F5`.

Kết quả cần đạt: số lượng sau cập nhật vẫn được giữ trong Session và tổng tiền vẫn đúng.

------------------------------------------------------------------------

## 10. Xóa sản phẩm khỏi giỏ hàng

Mỗi dòng có nút:

``` php
<button
    type="submit"
    name="remove_product"
    value="<?= (int) $item['ProductID'] ?>"
    class="btn btn-sm btn-outline-danger"
    formnovalidate
>
    Xóa
</button>
```

Khi nhấn nút, `ProductID` được gửi qua `remove_product`.

Phía PHP xử lý:

``` php
if (isset($_POST['remove_product'])) {

    $productID =
        (int) $_POST['remove_product'];

    if ($productID > 0) {
        unset($_SESSION['cart'][$productID]);
    }

    header('Location: /cart.php');
    exit;
}
```

`unset()` xóa phần tử tương ứng khỏi mảng Session.

Kiểm thử hai trường hợp:

-   xóa một sản phẩm khi giỏ còn nhiều sản phẩm;
-   xóa sản phẩm cuối cùng.

Khi giỏ không còn sản phẩm, trang phải hiển thị:

``` text
Giỏ hàng của bạn đang trống.
```

------------------------------------------------------------------------

## 11. Hiển thị số lượng giỏ hàng trên Navbar

Ta muốn người dùng nhìn thấy trạng thái giỏ hàng khi chuyển giữa các trang Frontend.

Ví dụ:

``` text
Sales Store    Trang chủ    Sản phẩm       Giỏ hàng (3)    Quản trị
```

Trong bài này, số `3` là **tổng số lượng sản phẩm**, không phải số dòng sản phẩm khác nhau.

Ví dụ:

``` text
Sản phẩm A × 1
Sản phẩm B × 2
----------------
Giỏ hàng (3)
```

### 11.1. Khởi tạo Session ở trang chủ

Mở:

``` cmd
code public\index.php
```

Ở đầu tập tin, trước `$pageTitle`, thêm:

``` php
require_once '/var/www/src/config/session.php';
```

Phần đầu trở thành:

``` php
<?php

require_once '/var/www/src/config/session.php';

$pageTitle = 'Trang chủ';
```

### 11.2. Khởi tạo Session ở trang danh sách sản phẩm

Mở:

``` cmd
code public\products.php
```

Đầu tập tin hiện có:

``` php
require_once '/var/www/src/config/database.php';
```

Thêm Session trước kết nối cơ sở dữ liệu:

``` php
require_once '/var/www/src/config/session.php';
require_once '/var/www/src/config/database.php';
```

Không thay đổi các phần tìm kiếm, lọc và phân trang đã hoàn thành ở Hands-on 10.

`product-detail.php` và `cart.php` đã nạp `session.php` ở các bước trước nên không cần thêm lần nữa.

------------------------------------------------------------------------

## 12. Bổ sung liên kết giỏ hàng vào Navbar

Mở:

``` cmd
code src\includes\frontend\navbar.php
```

Ở đầu tập tin, trước HTML, thêm:

``` php
<?php

$cartCount = array_sum(
    $_SESSION['cart'] ?? []
);

?>
```

`array_sum()` cộng toàn bộ `Quantity` đang lưu trong giỏ.

Ở phần bên phải Navbar, thay nút **Quản trị** hiện tại bằng nhóm hai nút:

``` php
<div class="d-flex gap-2">

    <a
        class="btn btn-outline-light btn-sm"
        href="/cart.php"
    >
        Giỏ hàng (<?= (int) $cartCount ?>)
    </a>

    <a
        class="btn btn-outline-light btn-sm"
        href="/admin/"
    >
        Quản trị
    </a>

</div>
```

Navbar lúc này có hai nhóm chức năng:

``` text
Điều hướng chính                   Tiện ích
Trang chủ | Sản phẩm               Giỏ hàng (n) | Quản trị
```

`navbar.php` chỉ **đọc** `$_SESSION`; việc khởi tạo Session vẫn thuộc trách nhiệm của các trang Frontend trước khi nạp Navbar.

------------------------------------------------------------------------

## 13. Kiểm tra cú pháp toàn bộ phần thay đổi

Chạy lần lượt:

``` cmd
docker compose exec web php -l /var/www/html/index.php
docker compose exec web php -l /var/www/html/products.php
docker compose exec web php -l /var/www/html/product-detail.php
docker compose exec web php -l /var/www/html/cart.php
docker compose exec web php -l /var/www/src/config/session.php
docker compose exec web php -l /var/www/src/includes/frontend/navbar.php
```

Cả sáu tập tin phải trả về:

``` text
No syntax errors detected
```

------------------------------------------------------------------------

## 14. Kiểm thử toàn bộ luồng giỏ hàng

Thực hiện theo đúng thứ tự sau.

### Trường hợp 1 -- Giỏ hàng ban đầu

Mở trang chủ và trang sản phẩm.

Khi chưa có sản phẩm trong Session, Navbar hiển thị:

``` text
Giỏ hàng (0)
```

### Trường hợp 2 -- Thêm một sản phẩm

Mở trang chi tiết sản phẩm, chọn số lượng `1`, nhấn **Thêm vào giỏ hàng**.

Kết quả:

``` text
Giỏ hàng (1)
```

### Trường hợp 3 -- Thêm lại cùng sản phẩm

Thêm tiếp cùng sản phẩm.

Kết quả cần đạt:

-   sản phẩm không xuất hiện thành hai dòng riêng;
-   số lượng của sản phẩm được cộng dồn;
-   số lượng không vượt tồn kho.

### Trường hợp 4 -- Thêm sản phẩm khác

Ví dụ:

``` text
Sản phẩm A × 1
Sản phẩm B × 2
```

Navbar phải hiển thị:

``` text
Giỏ hàng (3)
```

### Trường hợp 5 -- Kiểm tra thành tiền

Mở:

``` text
http://localhost:8080/cart.php
```

Kiểm tra:

``` text
Thành tiền = Đơn giá × Số lượng
Tổng cộng  = Tổng các thành tiền
```

### Trường hợp 6 -- Cập nhật số lượng

Thay đổi số lượng rồi nhấn:

``` text
Cập nhật giỏ hàng
```

Kiểm tra:

-   số lượng mới được giữ lại;
-   thành tiền thay đổi đúng;
-   tổng cộng thay đổi đúng;
-   số trên Navbar thay đổi tương ứng;
-   nhấn `F5` vẫn giữ kết quả.

### Trường hợp 7 -- Xóa sản phẩm

Nhấn **Xóa** ở một sản phẩm.

Kiểm tra:

-   đúng sản phẩm bị xóa;
-   tổng cộng được tính lại;
-   số trên Navbar giảm tương ứng.

### Trường hợp 8 -- Xóa sản phẩm cuối cùng

Sau khi xóa hết sản phẩm:

``` text
Giỏ hàng của bạn đang trống.
```

Navbar trở về:

``` text
Giỏ hàng (0)
```

### Trường hợp 9 -- Chuyển giữa các trang

Chuyển lần lượt:

``` text
Trang chủ
    ↓
Sản phẩm
    ↓
Chi tiết sản phẩm
    ↓
Giỏ hàng
```

Số lượng trên Navbar phải nhất quán giữa các trang trong cùng Session.

------------------------------------------------------------------------

## 15. Cấu trúc dự án sau Hands-on 11

Các tập tin liên quan trực tiếp đến chức năng mới:

``` text
php-mysql-sales/
│
├── public/
│   ├── index.php                 ← khởi tạo Session cho Frontend
│   ├── products.php              ← khởi tạo Session cho Frontend
│   ├── product-detail.php        ← thêm sản phẩm vào giỏ
│   └── cart.php                  ← xem/cập nhật/xóa giỏ hàng
│
└── src/
    ├── config/
    │   ├── database.php
    │   └── session.php           ← khởi tạo PHP Session
    │
    └── includes/
        └── frontend/
            ├── header.php
            ├── navbar.php        ← hiển thị Giỏ hàng (n)
            └── footer.php
```

------------------------------------------------------------------------

## 16. Lưu checkpoint bằng Git

Kiểm tra thay đổi:

``` cmd
git status
```

Các tập tin liên quan đến Hands-on này gồm:

``` text
public/cart.php
public/index.php
public/product-detail.php
public/products.php
src/config/session.php
src/includes/frontend/navbar.php
```

Xem nội dung thay đổi:

``` cmd
git diff
```

Đưa các tập tin vào staging area:

``` cmd
git add public/index.php public/product-detail.php public/products.php public/cart.php src/config/session.php src/includes/frontend/navbar.php
```

Kiểm tra lại:

``` cmd
git status
```

Commit:

``` cmd
git commit -m "Add session-based shopping cart"
```

Push lên GitHub:

``` cmd
git push
```

Kiểm tra lần cuối:

``` cmd
git status
```

Kết quả mong đợi:

``` text
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

------------------------------------------------------------------------

## 17. Kết quả đạt được

Sau Hands-on 11, Frontend đã có một luồng giỏ hàng hoàn chỉnh ở mức Session:

``` text
Danh sách sản phẩm
        ↓
Chi tiết sản phẩm
        ↓
Chọn số lượng
        ↓
Thêm vào giỏ hàng
        ↓
PHP Session
        ↓
Xem giỏ hàng
        ├── cập nhật số lượng
        ├── tính lại thành tiền
        ├── xóa sản phẩm
        └── hiển thị tổng số lượng trên Navbar
```

Điểm quan trọng cần ghi nhớ:

-   Session lưu **trạng thái giỏ hàng của phiên làm việc**, không thay thế cơ sở dữ liệu.
-   Giỏ hàng chỉ lưu `ProductID => Quantity`; thông tin sản phẩm được lấy lại từ MySQL khi cần hiển thị.
-   Dữ liệu từ trình duyệt vẫn phải được kiểm tra lại ở phía server.
-   Số lượng mua được giới hạn theo `StockQuantity` hiện tại.
-   `header('Location: ...')` được thực hiện trước khi xuất HTML.
-   Các kết quả MySQLi được đọc, giải phóng và statement được đóng trước khi tiếp tục các truy vấn liên quan.
-   `navbar.php` chỉ đọc trạng thái Session; việc khởi tạo Session được thực hiện trước khi nạp giao diện.

> Đến đây, ứng dụng đã cho phép người dùng chọn và quản lý sản phẩm trong giỏ hàng. Dữ liệu vẫn chỉ tồn tại trong Session; việc tạo đơn hàng và lưu dữ liệu mua hàng vào MySQL sẽ được phát triển ở bước tiếp theo.
