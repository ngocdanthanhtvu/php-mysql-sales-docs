# Hands-on 06 -- Upload hình ảnh sản phẩm với PHP

## 1. Mục tiêu

Sau Hands-on 06, người học có thể:

-   sử dụng `enctype="multipart/form-data"` và `$_FILES`;
-   kiểm tra lỗi upload, kích thước và MIME của ảnh;
-   dùng `finfo`, `random_bytes()`, `bin2hex()` và
    `move_uploaded_file()`;
-   lưu tên ảnh vào bảng `product_images`;
-   dùng `$conn->insert_id` để liên kết ảnh với sản phẩm vừa thêm;
-   dùng transaction, `commit()`, `rollback()` và `unlink()` để bảo đảm
    dữ liệu đồng bộ;
-   upload từ 1 đến 4 ảnh, quy định ảnh đầu tiên là ảnh chính và lưu
    `SortOrder`.

Kết quả cuối bài:

``` text
Product
└── 1..4 ProductImage
    ├── ảnh 1 → IsPrimary = 1 → SortOrder = 1
    ├── ảnh 2 → IsPrimary = 0 → SortOrder = 2
    ├── ảnh 3 → IsPrimary = 0 → SortOrder = 3
    └── ảnh 4 → IsPrimary = 0 → SortOrder = 4
```

------------------------------------------------------------------------

## 2. Kiểm tra môi trường upload

Chạy:

``` cmd
docker compose exec web php -i | findstr /I "file_uploads upload_max_filesize post_max_size"
```

Cần có:

``` text
file_uploads => On => On
```

Kiểm tra thư mục:

``` cmd
dir public\uploads\products
docker compose exec web ls -ld /var/www/html/uploads/products
```

Do `public` được mount vào `/var/www/html`:

``` text
public/uploads/products/
        ↕
/var/www/html/uploads/products/
```

------------------------------------------------------------------------

## 3. Tạo trang thử nghiệm `upload-test.php`

Tạo:

``` text
public/products/upload-test.php
```

Nội dung:

``` php
<?php
$pageTitle = 'Kiểm tra Upload File';
require_once '/var/www/src/includes/header.php';
require_once '/var/www/src/includes/navbar.php';
?>

<div class="container mt-4">

    <h2>Kiểm tra Upload File</h2>

    <form method="post" enctype="multipart/form-data">

        <div class="mb-3">
            <label for="productImage" class="form-label">
                Chọn ảnh
            </label>

            <input
                type="file"
                class="form-control"
                id="productImage"
                name="product_image"
                accept="image/*"
            >
        </div>

        <button type="submit" class="btn btn-primary">
            Gửi file
        </button>
    </form>

    <?php if ($_SERVER['REQUEST_METHOD'] === 'POST'): ?>

        <hr>
        <h4>Dữ liệu nhận được trong $_FILES</h4>
        <pre><?php print_r($_FILES); ?></pre>

    <?php endif; ?>

</div>

<?php
require_once '/var/www/src/includes/footer.php';
```

Mở:

``` text
http://localhost:8080/products/upload-test.php
```

Kết quả mẫu:

``` text
Array
(
    [product_image] => Array
        (
            [name] => test.png
            [full_path] => test.png
            [type] => image/png
            [tmp_name] => /tmp/phpxxxxxx
            [error] => 0
            [size] => 2851
        )
)
```

### Kiến thức cần hiểu

`multipart/form-data` là kiểu mã hóa cần dùng khi form gửi file.

Với:

``` html
name="product_image"
```

PHP nhận file qua:

``` php
$_FILES['product_image']
```

  Phần tử      Ý nghĩa
  ------------ -------------------------------
  `name`       tên file gốc
  `type`       MIME type do request cung cấp
  `tmp_name`   file tạm trên server
  `error`      mã lỗi upload
  `size`       kích thước theo byte

`error = 0` tương ứng `UPLOAD_ERR_OK`.

------------------------------------------------------------------------

## 4. Lưu file bằng `move_uploaded_file()`

Sau khi đã quan sát `$_FILES`, bổ sung vào phần POST của
`upload-test.php`:

``` php
<?php

if (
    isset($_FILES['product_image'])
    && $_FILES['product_image']['error'] === UPLOAD_ERR_OK
) {

    $file = $_FILES['product_image'];

    $originalName = basename($file['name']);

    $destination =
        '/var/www/html/uploads/products/' . $originalName;

    if (move_uploaded_file(
        $file['tmp_name'],
        $destination
    )) {
        echo '<div class="alert alert-success mt-3">';
        echo 'Upload file thành công.';
        echo '</div>';
    } else {
        echo '<div class="alert alert-danger mt-3">';
        echo 'Không thể lưu file.';
        echo '</div>';
    }
}
?>
```

Kiểm tra:

``` cmd
dir public\uploads\products
```

File chỉ được lưu lâu dài sau khi `move_uploaded_file()` thành công.

------------------------------------------------------------------------

## 5. Kiểm tra file trước khi lưu

Quy định của bài:

``` text
JPEG / PNG / WebP
Tối đa 2 MB cho mỗi ảnh
```

Sau:

``` php
$file = $_FILES['product_image'];
```

thêm:

``` php
$maxSize = 2 * 1024 * 1024;

if ($file['size'] > $maxSize) {
    die('File ảnh không được vượt quá 2 MB.');
}
```

Kiểm tra MIME ở server:

``` php
$finfo = new finfo(FILEINFO_MIME_TYPE);
$mimeType = $finfo->file($file['tmp_name']);

$allowedTypes = [
    'image/jpeg',
    'image/png',
    'image/webp'
];

if (!in_array($mimeType, $allowedTypes, true)) {
    die('Chỉ cho phép file JPG, JPEG, PNG hoặc WebP.');
}
```

Có thể tạm quan sát:

``` php
echo '<p>MIME type: '
    . htmlspecialchars($mimeType)
    . '</p>';
```

> `accept="image/*"` chỉ hỗ trợ lựa chọn file ở trình duyệt; vẫn phải
> kiểm tra file ở server.

------------------------------------------------------------------------

## 6. Tạo tên file mới

Ánh xạ MIME sang phần mở rộng:

``` php
$extensionMap = [
    'image/jpeg' => 'jpg',
    'image/png'  => 'png',
    'image/webp' => 'webp'
];

$extension = $extensionMap[$mimeType];
```

Tạo tên:

``` php
$newFileName =
    'product-'
    . bin2hex(random_bytes(8))
    . '.'
    . $extension;
```

Ví dụ:

``` text
product-a83f51c98d467102.png
```

Đổi đường dẫn đích thành:

``` php
$destination =
    '/var/www/html/uploads/products/' . $newFileName;
```

`random_bytes()` tạo dữ liệu ngẫu nhiên; `bin2hex()` chuyển thành chuỗi
hexadecimal thuận tiện cho tên file.

------------------------------------------------------------------------

## 7. Ghép upload vào chức năng Thêm sản phẩm

Đến đây `upload-test.php` đã kiểm chứng riêng toàn bộ cơ chế upload.

Ta ghép:

``` text
create.php của HO05
        +
logic đã kiểm chứng trong upload-test.php
        ↓
create.php có upload ảnh
```

Vì thay đổi logic liên quan nhiều khối phụ thuộc nhau, ở bước tích hợp
nên dùng **mã hoàn chỉnh đã kiểm chứng**, thay vì yêu cầu người học chắp
ghép quá nhiều đoạn nhỏ.

Có thể sao lưu trước:

``` cmd
copy public\products\create.php public\products\create-before-upload.php
```

Form phải có:

``` html
<form method="post" enctype="multipart/form-data">
```

Input một ảnh ở giai đoạn đầu:

``` html
<input
    type="file"
    class="form-control"
    id="productImage"
    name="product_image"
    accept="image/jpeg,image/png,image/webp"
    required
>
```

PHP nhận:

``` php
$file = $_FILES['product_image'] ?? null;
```

Sau khi thêm sản phẩm:

``` php
$productID = $conn->insert_id;
```

ID này được dùng khi thêm ảnh:

``` php
$stmtImage->bind_param(
    'iss',
    $productID,
    $newFileName,
    $altText
);
```

Luồng:

``` text
INSERT products
      ↓
$conn->insert_id
      ↓
move_uploaded_file()
      ↓
INSERT product_images
```

### Dữ liệu mẫu

``` text
Mã SP:         SP004
Tên:           Tai nghe Bluetooth D
Mô tả:         Tai nghe Bluetooth không dây, pin sử dụng lâu.
Đơn vị:        Chiếc
Giá:           890000
Tồn kho:       40
Danh mục:      Phụ kiện
Nhà cung cấp:  Công ty Thiết bị XYZ
Trạng thái:    Đang kinh doanh
Ảnh:           JPG/PNG/WebP <= 2 MB
```

Kiểm tra:

``` sql
SELECT
    p.ProductID,
    p.ProductCode,
    p.ProductName,
    pi.ImageFile,
    pi.IsPrimary
FROM
    products AS p,
    product_images AS pi
WHERE
    p.ProductID = pi.ProductID
    AND p.ProductCode = 'SP004';
```

------------------------------------------------------------------------

## 8. Bảo đảm toàn vẹn bằng transaction

Rủi ro:

``` text
INSERT products ✓
move file       ✓
INSERT image    ✗
```

Nếu không xử lý, sản phẩm và ảnh có thể không đồng bộ.

Dùng:

``` php
$conn->begin_transaction();
```

Khi thành công:

``` php
$conn->commit();
```

Khi lỗi:

``` php
$conn->rollback();
```

Cấu trúc:

``` php
try {

    $conn->begin_transaction();

    // INSERT products
    // lấy insert_id
    // move_uploaded_file()
    // INSERT product_images

    $conn->commit();

} catch (Throwable $e) {

    $conn->rollback();

    $error = $e->getMessage();
}
```

### File không thuộc transaction của MySQL

Ghi nhận:

``` php
$fileMoved = false;
```

Sau khi move thành công:

``` php
$fileMoved = true;
```

Nếu database lỗi:

``` php
if (
    $fileMoved
    && file_exists($destination)
) {
    unlink($destination);
}
```

Như vậy:

``` text
Database lỗi
   ├── rollback() → hủy dữ liệu
   └── unlink()   → xóa file đã lưu
```

------------------------------------------------------------------------

## 9. Kiểm thử rollback có chủ đích

Tạm đổi trong câu `INSERT product_images`:

``` sql
ImageFile
```

thành:

``` sql
ImageFileXYZ
```

Thử thêm `SP006`.

Kiểm tra:

``` sql
SELECT *
FROM products
WHERE ProductCode = 'SP006';
```

Kết quả mong đợi:

``` text
Empty set
```

Kiểm tra file:

``` cmd
dir public\uploads\products
```

Không được có file rác của lần thêm thất bại.

### CẢNH BÁO BẮT BUỘC

> `ImageFileXYZ` chỉ dùng để cố ý tạo lỗi kiểm thử.

Sau khi xác nhận `rollback()` và `unlink()` hoạt động, **phải sửa
ngay**:

``` sql
ImageFileXYZ
```

trở lại:

``` sql
ImageFile
```

**Không chuyển sang upload nhiều ảnh khi chưa khôi phục `ImageFile`.**

------------------------------------------------------------------------

## 10. Quan sát `$_FILES` khi upload nhiều ảnh

Đổi input thành:

``` html
<input
    type="file"
    class="form-control"
    id="productImages"
    name="product_images[]"
    accept="image/jpeg,image/png,image/webp"
    multiple
    required
>
```

Đổi label tương ứng sang:

``` html
for="productImages"
```

Hướng dẫn:

``` html
<div class="form-text">
    Chọn từ 1 đến 4 ảnh.
    Chấp nhận JPG, PNG hoặc WebP.
    Mỗi ảnh tối đa 2 MB.
    Ảnh đầu tiên là ảnh chính.
</div>
```

Tạm quan sát:

``` php
echo '<pre>';
print_r($_FILES['product_images'] ?? []);
echo '</pre>';
exit;
```

Khi chọn 4 ảnh, `name`, `tmp_name`, `error`, `size` đều trở thành các
mảng có cùng chỉ số:

``` text
              ảnh 1   ảnh 2   ảnh 3   ảnh 4
name            [0]     [1]     [2]     [3]
tmp_name        [0]     [1]     [2]     [3]
error           [0]     [1]     [2]     [3]
size            [0]     [1]     [2]     [3]
```

> Sau khi quan sát xong, xóa đoạn `print_r()` và `exit`.

------------------------------------------------------------------------

## 11. Xử lý từ 1 đến 4 ảnh

Nhận:

``` php
$files = $_FILES['product_images'] ?? null;
```

Đếm:

``` php
$fileCount = count($files['name']);
```

Kiểm tra:

``` php
if ($fileCount < 1 || $fileCount > 4) {
    $error = 'Chỉ được chọn từ 1 đến 4 ảnh.';
}
```

Kiểm tra từng ảnh:

``` php
for ($i = 0; $i < $fileCount; $i++) {
    // error
    // size
    // MIME
    // tạo tên file
}
```

Chuẩn bị:

``` php
$preparedImages[] = [
    'tmp_name'   => $files['tmp_name'][$i],
    'file_name'  => $newFileName,
    'is_primary' => ($i === 0) ? 1 : 0,
    'sort_order' => $i + 1
];
```

Nguyên tắc:

``` text
kiểm tra tất cả ảnh
        ↓
tất cả hợp lệ
        ↓
mới bắt đầu transaction
```

------------------------------------------------------------------------

## 12. `create.php` hoàn chỉnh hỗ trợ nhiều ảnh

Thay nội dung `public/products/create.php` bằng phiên bản sau sau khi đã
hoàn tất các bước quan sát ở trên:

``` php
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

    $files = $_FILES['product_images'] ?? null;

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
    } elseif (
        !$files
        || !isset($files['name'])
        || !is_array($files['name'])
    ) {
        $error = 'Vui lòng chọn ảnh sản phẩm.';
    } else {

        $fileCount = count($files['name']);

        if ($fileCount < 1 || $fileCount > 4) {

            $error = 'Chỉ được chọn từ 1 đến 4 ảnh.';

        } else {

            $maxSize = 2 * 1024 * 1024;

            $extensionMap = [
                'image/jpeg' => 'jpg',
                'image/png'  => 'png',
                'image/webp' => 'webp'
            ];

            $finfo = new finfo(FILEINFO_MIME_TYPE);
            $preparedImages = [];

            for ($i = 0; $i < $fileCount; $i++) {

                if ($files['error'][$i] !== UPLOAD_ERR_OK) {
                    $error = 'Có file ảnh upload không thành công.';
                    break;
                }

                if ($files['size'][$i] > $maxSize) {
                    $error = 'Mỗi file ảnh không được vượt quá 2 MB.';
                    break;
                }

                $mimeType = $finfo->file(
                    $files['tmp_name'][$i]
                );

                if (!isset($extensionMap[$mimeType])) {
                    $error = 'Chỉ cho phép file JPG, PNG hoặc WebP.';
                    break;
                }

                $extension = $extensionMap[$mimeType];

                $newFileName =
                    'product-'
                    . bin2hex(random_bytes(8))
                    . '.'
                    . $extension;

                $preparedImages[] = [
                    'tmp_name'   => $files['tmp_name'][$i],
                    'file_name'  => $newFileName,
                    'is_primary' => ($i === 0) ? 1 : 0,
                    'sort_order' => $i + 1
                ];
            }

            if ($error === '') {

                $movedFiles = [];

                try {

                    $conn->begin_transaction();

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

                    if (!$stmt->execute()) {
                        throw new Exception(
                            'Không thể thêm sản phẩm.'
                        );
                    }

                    $productID = $conn->insert_id;
                    $stmt->close();

                    $sqlImage = "
                        INSERT INTO product_images
                        (
                            ProductID,
                            ImageFile,
                            AltText,
                            IsPrimary,
                            SortOrder
                        )
                        VALUES
                        (?, ?, ?, ?, ?)
                    ";

                    $stmtImage = $conn->prepare($sqlImage);

                    foreach ($preparedImages as $index => $image) {

                        $destination =
                            '/var/www/html/uploads/products/'
                            . $image['file_name'];

                        if (!move_uploaded_file(
                            $image['tmp_name'],
                            $destination
                        )) {
                            throw new Exception(
                                'Không thể lưu một trong các file ảnh.'
                            );
                        }

                        $movedFiles[] = $destination;

                        if ($image['is_primary'] === 1) {
                            $altText =
                                $productName . ' - ảnh chính';
                        } else {
                            $altText =
                                $productName
                                . ' - ảnh '
                                . ($index + 1);
                        }

                        $imageFile = $image['file_name'];
                        $isPrimary = $image['is_primary'];
                        $sortOrder = $image['sort_order'];

                        $stmtImage->bind_param(
                            'issii',
                            $productID,
                            $imageFile,
                            $altText,
                            $isPrimary,
                            $sortOrder
                        );

                        if (!$stmtImage->execute()) {
                            throw new Exception(
                                'Không thể lưu thông tin ảnh.'
                            );
                        }
                    }

                    $stmtImage->close();
                    $conn->commit();

                    header('Location: /products/');
                    exit;

                } catch (Throwable $e) {

                    $conn->rollback();

                    foreach ($movedFiles as $movedFile) {
                        if (file_exists($movedFile)) {
                            unlink($movedFile);
                        }
                    }

                    $error = $e->getMessage();
                }
            }
        }
    }
}

require_once '/var/www/src/includes/header.php';
require_once '/var/www/src/includes/navbar.php';

?>

<div class="container mt-4">

    <h2 class="mb-4">Thêm sản phẩm</h2>

    <?php if ($error !== ''): ?>
        <div class="alert alert-danger">
            <?= htmlspecialchars($error) ?>
        </div>
    <?php endif; ?>

    <form method="post" enctype="multipart/form-data">

        <div class="row">

            <div class="col-md-4 mb-3">
                <label for="productCode" class="form-label">
                    Mã sản phẩm
                </label>
                <input
                    type="text"
                    class="form-control"
                    id="productCode"
                    name="product_code"
                    value="<?= htmlspecialchars(
                        $_POST['product_code'] ?? ''
                    ) ?>"
                    required
                >
            </div>

            <div class="col-md-8 mb-3">
                <label for="productName" class="form-label">
                    Tên sản phẩm
                </label>
                <input
                    type="text"
                    class="form-control"
                    id="productName"
                    name="product_name"
                    value="<?= htmlspecialchars(
                        $_POST['product_name'] ?? ''
                    ) ?>"
                    required
                >
            </div>

        </div>

        <div class="mb-3">
            <label for="description" class="form-label">
                Mô tả
            </label>
            <textarea
                class="form-control"
                id="description"
                name="description"
                rows="3"
            ><?= htmlspecialchars(
                $_POST['description'] ?? ''
            ) ?></textarea>
        </div>

        <div class="row">

            <div class="col-md-4 mb-3">
                <label for="unit" class="form-label">
                    Đơn vị tính
                </label>
                <input
                    type="text"
                    class="form-control"
                    id="unit"
                    name="unit"
                    value="<?= htmlspecialchars(
                        $_POST['unit'] ?? ''
                    ) ?>"
                >
            </div>

            <div class="col-md-4 mb-3">
                <label for="price" class="form-label">
                    Giá
                </label>
                <input
                    type="number"
                    class="form-control"
                    id="price"
                    name="price"
                    min="0"
                    step="0.01"
                    value="<?= htmlspecialchars(
                        $_POST['price'] ?? '0'
                    ) ?>"
                    required
                >
            </div>

            <div class="col-md-4 mb-3">
                <label for="stockQuantity" class="form-label">
                    Tồn kho
                </label>
                <input
                    type="number"
                    class="form-control"
                    id="stockQuantity"
                    name="stock_quantity"
                    min="0"
                    value="<?= htmlspecialchars(
                        $_POST['stock_quantity'] ?? '0'
                    ) ?>"
                    required
                >
            </div>

        </div>

        <div class="row">

            <div class="col-md-6 mb-3">
                <label for="categoryID" class="form-label">
                    Danh mục
                </label>

                <select
                    class="form-select"
                    id="categoryID"
                    name="category_id"
                    required
                >
                    <option value="">-- Chọn danh mục --</option>

                    <?php while (
                        $category = $categories->fetch_assoc()
                    ): ?>

                        <option
                            value="<?= $category['CategoryID'] ?>"
                            <?= (
                                ($_POST['category_id'] ?? '')
                                == $category['CategoryID']
                            ) ? 'selected' : '' ?>
                        >
                            <?= htmlspecialchars(
                                $category['CategoryName']
                            ) ?>
                        </option>

                    <?php endwhile; ?>
                </select>
            </div>

            <div class="col-md-6 mb-3">
                <label for="supplierID" class="form-label">
                    Nhà cung cấp
                </label>

                <select
                    class="form-select"
                    id="supplierID"
                    name="supplier_id"
                    required
                >
                    <option value="">-- Chọn nhà cung cấp --</option>

                    <?php while (
                        $supplier = $suppliers->fetch_assoc()
                    ): ?>

                        <option
                            value="<?= $supplier['SupplierID'] ?>"
                            <?= (
                                ($_POST['supplier_id'] ?? '')
                                == $supplier['SupplierID']
                            ) ? 'selected' : '' ?>
                        >
                            <?= htmlspecialchars(
                                $supplier['SupplierName']
                            ) ?>
                        </option>

                    <?php endwhile; ?>
                </select>
            </div>

        </div>

        <div class="mb-3">
            <label for="productImages" class="form-label">
                Hình ảnh sản phẩm
            </label>

            <input
                type="file"
                class="form-control"
                id="productImages"
                name="product_images[]"
                accept="image/jpeg,image/png,image/webp"
                multiple
                required
            >

            <div class="form-text">
                Chọn từ 1 đến 4 ảnh.
                Chấp nhận JPG, PNG hoặc WebP.
                Mỗi ảnh tối đa 2 MB.
                Ảnh đầu tiên là ảnh chính.
            </div>
        </div>

        <div class="form-check mb-3">
            <input
                type="checkbox"
                class="form-check-input"
                id="isActive"
                name="is_active"
                value="1"
                <?= (
                    isset($_POST['is_active'])
                    || $_SERVER['REQUEST_METHOD'] !== 'POST'
                ) ? 'checked' : '' ?>
            >

            <label class="form-check-label" for="isActive">
                Đang kinh doanh
            </label>
        </div>

        <button type="submit" class="btn btn-primary">
            Lưu
        </button>

        <a href="/products/" class="btn btn-secondary">
            Hủy
        </a>

    </form>
</div>

<?php
require_once '/var/www/src/includes/footer.php';
$conn->close();
```

------------------------------------------------------------------------

## 13. Kiểm tra kết quả cuối bài

Dữ liệu mẫu:

``` text
Mã SP:         SP006
Tên:           Loa Bluetooth F
Mô tả:         Loa Bluetooth di động, âm thanh rõ và pin sử dụng lâu.
Đơn vị:        Chiếc
Giá:           1590000
Tồn kho:       15
Danh mục:      Phụ kiện
Nhà cung cấp:  Công ty Công nghệ ABC
Trạng thái:    Đang kinh doanh
Ảnh:           chọn 4 ảnh
```

Kiểm tra:

``` sql
SELECT
    p.ProductID,
    p.ProductCode,
    p.ProductName,
    pi.ImageFile,
    pi.IsPrimary,
    pi.SortOrder
FROM
    products AS p,
    product_images AS pi
WHERE
    p.ProductID = pi.ProductID
    AND p.ProductCode = 'SP006'
ORDER BY
    pi.SortOrder;
```

Kết quả cần có 4 dòng:

``` text
ảnh 1 → IsPrimary = 1 → SortOrder = 1
ảnh 2 → IsPrimary = 0 → SortOrder = 2
ảnh 3 → IsPrimary = 0 → SortOrder = 3
ảnh 4 → IsPrimary = 0 → SortOrder = 4
```

Trang danh sách sản phẩm vẫn chỉ lấy ảnh chính nếu truy vấn có:

``` sql
AND pi.IsPrimary = 1
```

------------------------------------------------------------------------

## 14. Checklist cuối Hands-on 06

``` text
[ ] file_uploads = On
[ ] thư mục uploads/products tồn tại
[ ] hiểu multipart/form-data
[ ] hiểu $_FILES khi upload một file
[ ] hiểu tmp_name và UPLOAD_ERR_OK
[ ] sử dụng move_uploaded_file()
[ ] kiểm tra giới hạn 2 MB
[ ] kiểm tra MIME bằng finfo
[ ] chỉ chấp nhận JPEG, PNG, WebP
[ ] tạo tên file bằng random_bytes() và bin2hex()
[ ] ghép upload vào create.php
[ ] hiểu $conn->insert_id
[ ] lưu đúng ProductID vào product_images
[ ] hiểu begin_transaction(), commit(), rollback()
[ ] dùng unlink() khi transaction thất bại
[ ] đã kiểm thử rollback bằng ImageFileXYZ
[ ] BẮT BUỘC đã sửa ImageFileXYZ trở lại ImageFile
[ ] đã xóa print_r($_FILES) và exit dùng để quan sát
[ ] hiểu $_FILES khi upload nhiều file
[ ] chọn được từ 1 đến 4 ảnh
[ ] tất cả ảnh được kiểm tra trước khi INSERT
[ ] ảnh đầu tiên có IsPrimary = 1
[ ] các ảnh còn lại có IsPrimary = 0
[ ] SortOrder lần lượt 1, 2, 3, 4
[ ] SP006 có đúng 4 ảnh trong product_images
```

------------------------------------------------------------------------

## 15. Kiến thức PHP/MySQLi đã sử dụng

  Thành phần               Vai trò
  ------------------------ -------------------------------------
  `$_FILES`                nhận file upload
  `multipart/form-data`    mã hóa form có file
  `UPLOAD_ERR_OK`          xác nhận upload không lỗi
  `finfo`                  kiểm tra MIME
  `random_bytes()`         tạo dữ liệu ngẫu nhiên
  `bin2hex()`              chuyển byte thành chuỗi hexadecimal
  `move_uploaded_file()`   lưu file upload
  `$conn->insert_id`       lấy ID bản ghi vừa thêm
  `begin_transaction()`    bắt đầu transaction
  `commit()`               xác nhận transaction
  `rollback()`             hủy thay đổi database
  `throw` / `catch`        phát sinh và xử lý ngoại lệ
  `file_exists()`          kiểm tra file
  `unlink()`               xóa file
  `for` / `foreach`        duyệt nhiều ảnh
  Prepared Statement       thực thi SQL có tham số

------------------------------------------------------------------------

## 16. Kết thúc Hands-on 06

Hands-on 06 dừng tại:

``` text
Thêm sản phẩm
      +
Upload 1–4 ảnh
      +
Transaction
      +
Ảnh chính
      +
Thứ tự ảnh
```

Chưa thực hiện quản lý vòng đời ảnh trong bài này.

Hands-on tiếp theo:

``` text
HO07 – Quản lý hình ảnh sản phẩm

Hiển thị tất cả ảnh
        ↓
Đặt ảnh chính
        ↓
Thêm ảnh
        ↓
Xóa ảnh
        ↓
Đồng bộ database và file vật lý
```

> Giữ nguyên SP006 và 4 ảnh đã tạo ở cuối HO06 để tiếp tục thực hành
> HO07.
