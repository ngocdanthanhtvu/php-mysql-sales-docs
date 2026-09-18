# Hands-on 07 -- Quản lý hình ảnh sản phẩm khi cập nhật

## 1. Mục tiêu

Trong Hands-on 06, chúng ta đã xây dựng chức năng thêm sản phẩm và upload từ 1 đến 4 ảnh. Thông tin ảnh được lưu trong bảng `product_images`; một ảnh được đánh dấu là ảnh chính bằng `IsPrimary`, còn `SortOrder` lưu thứ tự hiển thị.

Hands-on 07 tiếp tục hoàn thiện trang `edit.php`. Mục tiêu không chỉ là làm cho chức năng hoạt động, mà còn giúp người học hiểu cách quản lý **vòng đời của tập tin ảnh** và giữ dữ liệu MySQL đồng bộ với tập tin vật lý.

Sau Hands-on 07, người học có thể:

- đọc và hiển thị nhiều ảnh của một sản phẩm;
- phân biệt ảnh chính và ảnh phụ;
- thay đổi ảnh chính;
- thêm nhiều ảnh cho sản phẩm đã tồn tại;
- xóa ảnh và đồng bộ dữ liệu với tập tin vật lý;
- tự chọn ảnh chính mới khi ảnh chính bị xóa;
- chuẩn hóa lại `SortOrder` sau khi xóa;
- sử dụng transaction cho các thao tác thay đổi nhiều mẩu tin;
- xử lý trường hợp sản phẩm không còn ảnh.

Kết quả cuối bài:

```text
Sửa sản phẩm
│
├── Cập nhật thông tin sản phẩm
└── Quản lý hình ảnh
    ├── Hiển thị tất cả ảnh
    ├── Đặt ảnh chính
    ├── Thêm ảnh
    └── Xóa ảnh
         ├── cập nhật product_images
         ├── chọn ảnh chính mới nếu cần
         ├── chuẩn hóa SortOrder
         └── xóa tập tin vật lý
```

---

## 2. Chuẩn bị dữ liệu thực hành

Hands-on này tiếp tục trực tiếp từ kết quả của Hands-on 06. Sản phẩm `SP006` đã có 4 ảnh và được dùng làm dữ liệu thực hành.

Khởi động:

```cmd
docker compose up -d
docker compose ps
```

Mở `http://localhost:8080/products/`.

Vào MySQL:

```cmd
docker compose exec db mysql -u root -p
```

```sql
USE ql_banhang;

SELECT
    p.ProductID, p.ProductCode, p.ProductName,
    pi.ProductImageID, pi.ImageFile, pi.IsPrimary, pi.SortOrder
FROM products AS p, product_images AS pi
WHERE p.ProductID = pi.ProductID
  AND p.ProductCode = 'SP006'
ORDER BY pi.SortOrder;
```

Cần có 4 ảnh, `SortOrder` lần lượt 1–4 và chỉ một ảnh có `IsPrimary = 1`.

> Tên tập tin có thể khác vì Hands-on 06 tạo tên bằng `random_bytes()`. Không sửa tên trong cơ sở dữ liệu chỉ để giống ví dụ.

---

## 3. Đọc danh sách ảnh của sản phẩm

Mở `public/products/edit.php`. Sau khi đã xác định `$productID`, bổ sung:

```php
$sqlImages = "
    SELECT
        ProductImageID,
        ImageFile,
        AltText,
        IsPrimary,
        SortOrder
    FROM product_images
    WHERE ProductID = ?
    ORDER BY SortOrder, ProductImageID
";

$stmtImages = $conn->prepare($sqlImages);
$stmtImages->bind_param('i', $productID);
$stmtImages->execute();
$productImages = $stmtImages->get_result();
```

### Kiến thức cần hiểu

Quan hệ là `Product 1 -- N ProductImage`. Vì vậy kết quả `$productImages` chứa nhiều mẩu tin và phải được duyệt bằng vòng lặp. Prepared Statement tiếp tục được sử dụng vì `ProductID` là tham số đưa vào SQL.

---

## 4. Hiển thị tất cả ảnh trên trang sửa sản phẩm

Trong form `edit.php`, thêm khu vực hình ảnh:

```php
<hr class="my-4">
<h4 class="mb-3">Hình ảnh sản phẩm</h4>

<div class="row">
    <?php while ($image = $productImages->fetch_assoc()): ?>
        <div class="col-md-3 mb-3">
            <div class="card h-100">
                <img
                    src="/uploads/products/<?= htmlspecialchars($image['ImageFile']) ?>"
                    class="card-img-top"
                    alt="<?= htmlspecialchars($image['AltText'] ?? '') ?>"
                >

                <div class="card-body">
                    <small class="text-muted">
                        Thứ tự: <?= $image['SortOrder'] ?>
                    </small>

                    <?php if ((int) $image['IsPrimary'] === 1): ?>
                        <div class="mt-2">
                            <span class="badge bg-success">Ảnh chính</span>
                        </div>
                    <?php endif; ?>
                </div>
            </div>
        </div>
    <?php endwhile; ?>
</div>
```

### Kiểm tra

Mở trang sửa `SP006`. Cần thấy đủ 4 ảnh, đúng thứ tự và một ảnh có nhãn **Ảnh chính**.

### Kiến thức mới

Cần phân biệt:

```text
CSDL:       product-xxxxxxxx.webp
URL:        /uploads/products/product-xxxxxxxx.webp
Server:     /var/www/html/uploads/products/product-xxxxxxxx.webp
```

Tên tập tin, URL và đường dẫn vật lý phục vụ ba mục đích khác nhau.

---

## 5. Đặt một ảnh khác làm ảnh chính

Mỗi sản phẩm chỉ nên có **một ảnh chính**:

```text
ảnh được chọn   → IsPrimary = 1
ảnh còn lại     → IsPrimary = 0
```

### 5.1. Thêm nút trên ảnh phụ

Trong card của ảnh không phải ảnh chính, thêm:

```php
<button
    type="submit"
    class="btn btn-outline-primary btn-sm"
    name="set_primary_image"
    value="<?= $image['ProductImageID'] ?>"
    formaction="/products/edit.php?id=<?= $productID ?>"
    formmethod="post"
>
    Đặt làm ảnh chính
</button>
```

### 5.2. Xử lý POST

Nhánh xử lý đổi ảnh chính phải được đặt trước nhánh cập nhật thông tin sản phẩm thông thường:

```php
if (isset($_POST['set_primary_image'])) {

    $imageID = (int) $_POST['set_primary_image'];

    try {
        $conn->begin_transaction();

        $sqlResetPrimary = "
            UPDATE product_images
            SET IsPrimary = 0
            WHERE ProductID = ?
        ";

        $stmtResetPrimary = $conn->prepare($sqlResetPrimary);
        $stmtResetPrimary->bind_param('i', $productID);
        $stmtResetPrimary->execute();
        $stmtResetPrimary->close();

        $sqlSetPrimary = "
            UPDATE product_images
            SET IsPrimary = 1
            WHERE ProductImageID = ?
              AND ProductID = ?
        ";

        $stmtSetPrimary = $conn->prepare($sqlSetPrimary);
        $stmtSetPrimary->bind_param('ii', $imageID, $productID);
        $stmtSetPrimary->execute();

        if ($stmtSetPrimary->affected_rows !== 1) {
            throw new Exception('Không thể đặt ảnh chính.');
        }

        $stmtSetPrimary->close();
        $conn->commit();

        header(
            'Location: /products/edit.php?id='
            . $productID
            . '&primary_updated=1'
        );
        exit;

    } catch (Throwable $e) {
        $conn->rollback();
        $error = $e->getMessage();
    }
}
```

### Vì sao cần transaction?

Thao tác gồm hai `UPDATE`. Nếu chỉ câu đầu thành công, sản phẩm có thể không còn ảnh chính. Transaction giúp hai thay đổi được xem như một đơn vị công việc.

### 5.3. Feedback

```php
<?php if (
    isset($_GET['primary_updated'])
    && $_GET['primary_updated'] === '1'
): ?>
    <div class="alert alert-success">
        Đã cập nhật ảnh chính.
    </div>
<?php endif; ?>
```

### Kiểm tra

Đổi một ảnh của `SP006` thành ảnh chính rồi kiểm tra:

```sql
SELECT ProductImageID, IsPrimary, SortOrder
FROM product_images
WHERE ProductID = <ProductID-cua-SP006>
ORDER BY SortOrder;

SELECT COUNT(*) AS PrimaryImageCount
FROM product_images
WHERE ProductID = <ProductID-cua-SP006>
  AND IsPrimary = 1;
```

`PrimaryImageCount` phải bằng `1`.

---

## 6. Thêm ảnh cho sản phẩm đã tồn tại

Khác với `create.php`, ở trang sửa sản phẩm chúng ta đã có `$productID`.
Nhiệm vụ của bước này là nhận các ảnh mới, kiểm tra chúng, lưu tập tin và
thêm các mẩu tin tương ứng vào `product_images`.

### 6.1. Bổ sung input nhiều ảnh

Trước hết, kiểm tra thẻ `<form>` bao ngoài. Form phải có:

```html
enctype="multipart/form-data"
```

Nếu chưa có, bổ sung thuộc tính này. Sau khu vực hiển thị các ảnh hiện có,
thêm:

```html
<div class="mb-3">
    <label for="productImages" class="form-label">
        Thêm hình ảnh
    </label>

    <input
        type="file"
        class="form-control"
        id="productImages"
        name="product_images[]"
        accept="image/jpeg,image/png,image/webp"
        multiple
    >

    <div class="form-text">
        Chấp nhận JPG, PNG hoặc WebP.
        Mỗi ảnh tối đa 2 MB.
    </div>
</div>

<button
    type="submit"
    class="btn btn-outline-success"
    name="add_images"
    value="1"
>
    Thêm ảnh
</button>
```

Ở đây `product_images[]` cho phép PHP nhận nhiều tập tin trong cùng một lần
gửi form.

### 6.2. Tạo nhánh xử lý `add_images`

Trong phần xử lý `POST`, đặt nhánh này **trước nhánh cập nhật thông tin sản
phẩm thông thường**:

```php
if (isset($_POST['add_images'])) {
    $files = $_FILES['product_images'] ?? null;

    if (
        !$files
        || !isset($files['name'])
        || !is_array($files['name'])
    ) {
        $error = 'Vui lòng chọn ít nhất một ảnh.';
    } else {
        $maxSize = 2 * 1024 * 1024;

        $extensionMap = [
            'image/jpeg' => 'jpg',
            'image/png'  => 'png',
            'image/webp' => 'webp'
        ];

        $validImages = [];
        $fileCount = count($files['name']);

        $finfo = new finfo(FILEINFO_MIME_TYPE);

        for ($i = 0; $i < $fileCount; $i++) {
            if ($files['error'][$i] === UPLOAD_ERR_NO_FILE) {
                continue;
            }

            if ($files['error'][$i] !== UPLOAD_ERR_OK) {
                $error = 'Có lỗi xảy ra khi upload ảnh.';
                break;
            }

            if ($files['size'][$i] > $maxSize) {
                $error = 'Mỗi ảnh chỉ được có kích thước tối đa 2 MB.';
                break;
            }

            $mimeType = $finfo->file($files['tmp_name'][$i]);

            if (!isset($extensionMap[$mimeType])) {
                $error = 'Chỉ chấp nhận ảnh JPG, PNG hoặc WebP.';
                break;
            }

            $extension = $extensionMap[$mimeType];

            $fileName =
                'product-'
                . bin2hex(random_bytes(8))
                . '.'
                . $extension;

            $validImages[] = [
                'tmp_name'  => $files['tmp_name'][$i],
                'file_name' => $fileName
            ];
        }

        if (!$error && count($validImages) === 0) {
            $error = 'Vui lòng chọn ít nhất một ảnh.';
        }
    }
}
```

### Kiến thức cần hiểu

Không nên tin hoàn toàn vào phần mở rộng của tên tập tin do người dùng gửi
lên. `finfo` đọc MIME từ nội dung thực tế của tập tin. Tên mới được tạo bằng
`random_bytes()` để tránh phụ thuộc vào tên gốc và giảm khả năng trùng tên.

### 6.3. Xác định `SortOrder` và trạng thái ảnh chính

Nếu tất cả ảnh hợp lệ, tiếp tục **bên trong nhánh `add_images`**, sau phần
kiểm tra ở trên:

```php
if (!$error) {
    $sqlImageState = "
        SELECT
            COUNT(*) AS ImageCount,
            COALESCE(MAX(SortOrder), 0) AS MaxSortOrder
        FROM product_images
        WHERE ProductID = ?
    ";

    $stmtImageState = $conn->prepare($sqlImageState);
    $stmtImageState->bind_param('i', $productID);
    $stmtImageState->execute();

    $imageState =
        $stmtImageState->get_result()->fetch_assoc();

    $stmtImageState->close();

    $imageCount = (int) $imageState['ImageCount'];
    $nextSortOrder =
        (int) $imageState['MaxSortOrder'] + 1;
}
```

Quy tắc:

```text
Sản phẩm đã có ảnh  → ảnh mới IsPrimary = 0
Sản phẩm chưa có ảnh → ảnh mới đầu tiên IsPrimary = 1
```

Điều này đặc biệt quan trọng với sản phẩm đã bị xóa hết ảnh ở một lần chỉnh
sửa trước đó.

### 6.4. Lưu ảnh bằng transaction

Tiếp tục trong `if (!$error)`:

```php
if (!$error) {
    $movedFiles = [];

    try {
        $conn->begin_transaction();

        $sqlInsertImage = "
            INSERT INTO product_images
            (
                ProductID,
                ImageFile,
                AltText,
                IsPrimary,
                SortOrder
            )
            VALUES (?, ?, ?, ?, ?)
        ";

        $stmtInsertImage =
            $conn->prepare($sqlInsertImage);

        foreach ($validImages as $index => $image) {
            $destination =
                '/var/www/html/uploads/products/'
                . $image['file_name'];

            if (!move_uploaded_file(
                $image['tmp_name'],
                $destination
            )) {
                throw new Exception(
                    'Không thể lưu một trong các ảnh.'
                );
            }

            $movedFiles[] = $destination;

            $isPrimary =
                ($imageCount === 0 && $index === 0)
                ? 1
                : 0;

            $sortOrder = $nextSortOrder + $index;

            $altText =
                $product['ProductName']
                . (
                    $isPrimary === 1
                    ? ' - ảnh chính'
                    : ' - ảnh ' . $sortOrder
                );

            $stmtInsertImage->bind_param(
                'issii',
                $productID,
                $image['file_name'],
                $altText,
                $isPrimary,
                $sortOrder
            );

            if (!$stmtInsertImage->execute()) {
                throw new Exception(
                    'Không thể lưu thông tin ảnh.'
                );
            }
        }

        $stmtInsertImage->close();
        $conn->commit();

        header(
            'Location: /products/edit.php?id='
            . $productID
            . '&images_added=1'
        );
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
```

> Nếu trong `edit.php` của bạn biến chứa thông tin sản phẩm có tên khác
> `$product`, hãy dùng đúng biến hiện có khi tạo `$altText`.

### Vì sao phải dọn tập tin khi rollback?

Transaction chỉ quản lý dữ liệu MySQL:

```text
rollback() → hoàn tác INSERT trong product_images
unlink()   → xóa các tập tin đã move trước khi lỗi xảy ra
```

Nếu chỉ `rollback()` mà không `unlink()`, thư mục upload có thể còn các tập
tin không còn mẩu tin tương ứng trong cơ sở dữ liệu.

### 6.5. Thêm feedback

Cùng khu vực đang hiển thị các thông báo thành công, bổ sung:

```php
<?php if (
    isset($_GET['images_added'])
    && $_GET['images_added'] === '1'
): ?>
    <div class="alert alert-success">
        Đã thêm hình ảnh sản phẩm.
    </div>
<?php endif; ?>
```

### 6.6. Kiểm tra

Thêm 2 ảnh vào `SP006`, sau đó kiểm tra:

```sql
SELECT
    ProductImageID,
    ProductID,
    ImageFile,
    IsPrimary,
    SortOrder
FROM product_images
WHERE ProductID = <ProductID-cua-SP006>
ORDER BY SortOrder;
```

Cần thỏa:

```text
hai ảnh mới đã được thêm
SortOrder nối tiếp các ảnh cũ
chỉ có một ảnh có IsPrimary = 1
```

Kiểm tra số ảnh chính:

```sql
SELECT COUNT(*) AS PrimaryImageCount
FROM product_images
WHERE ProductID = <ProductID-cua-SP006>
  AND IsPrimary = 1;
```

`PrimaryImageCount` phải bằng `1`.

---

## 7. Xóa một ảnh

Một ảnh của sản phẩm tồn tại đồng thời ở hai nơi:

```text
MySQL                     File system
product_images            uploads/products/
     │                           │
     └──── ImageFile ────────────┘
```

Vì vậy, xóa ảnh phải xử lý cả mẩu tin trong `product_images` và tập tin vật
lý.

### 7.1. Thêm nút Xóa ảnh

Nút xóa phải xuất hiện ở cả ảnh chính và ảnh phụ:

```php
<button
    type="submit"
    class="btn btn-outline-danger btn-sm ms-2"
    name="delete_image"
    value="<?= $image['ProductImageID'] ?>"
    formaction="/products/edit.php?id=<?= $productID ?>"
    formmethod="post"
    onclick="return confirm('Bạn có chắc muốn xóa ảnh này?');"
>
    Xóa ảnh
</button>
```

Ảnh chính vẫn giữ badge **Ảnh chính**, nhưng bên dưới vẫn có nút **Xóa ảnh**.

### 7.2. Những dữ liệu cần biết trước khi xóa

Trước khi `DELETE`, chương trình cần đọc:

```text
ImageFile   → biết tập tin vật lý cần xóa
IsPrimary   → biết có phải chọn ảnh chính mới hay không
SortOrder   → biết vị trí cần chuẩn hóa lại
```

Ba giá trị này phải được lấy **trước khi xóa mẩu tin**.

---

## 8. Xử lý khi xóa ảnh chính

Nếu ảnh bị xóa là ảnh chính, ảnh còn lại có thứ tự nhỏ nhất sẽ trở thành ảnh
chính mới:

```text
xóa ảnh chính
      ↓
còn ảnh?
 ┌────┴────┐
có        không
│           │
↓           ↓
ảnh đầu    sản phẩm
tiên mới   không còn ảnh
→ primary
```

Câu SQL dùng để chọn ảnh còn lại đầu tiên:

```sql
UPDATE product_images
SET IsPrimary = 1
WHERE ProductImageID = (
    SELECT ProductImageID
    FROM (
        SELECT ProductImageID
        FROM product_images
        WHERE ProductID = ?
        ORDER BY SortOrder, ProductImageID
        LIMIT 1
    ) AS remaining_images
);
```

Nếu không còn ảnh, câu `UPDATE` không tác động mẩu tin nào. Đây là trạng thái
hợp lệ.

### Kiến thức cần hiểu

Lớp truy vấn:

```sql
FROM (
    SELECT ...
) AS remaining_images
```

tạo một tập kết quả trung gian trước khi cập nhật chính bảng
`product_images`.

---

## 9. Chuẩn hóa `SortOrder` sau khi xóa

Giả sử thứ tự hiện tại là:

```text
1  2  3  4  5  6
```

Nếu xóa ảnh thứ 4, dữ liệu tạm thời thành:

```text
1  2  3  5  6
```

Ta muốn đưa về:

```text
1  2  3  4  5
```

Do đó các ảnh đứng sau ảnh vừa xóa cần giảm `SortOrder` đi 1:

```sql
UPDATE product_images
SET SortOrder = SortOrder - 1
WHERE ProductID = ?
  AND SortOrder > ?;
```

Cần phân biệt:

```text
ProductImageID → khóa chính, không cần liên tục
SortOrder      → vị trí hiển thị, nên được giữ liên tục
```

Ví dụ `ProductImageID` là `7, 8, 11, 13` vẫn hoàn toàn hợp lệ.

---

## 10. Ghép hoàn chỉnh nhánh xử lý xóa ảnh

Sau khi đã hiểu ba vấn đề ở trên, ta ghép chúng thành một nhánh xử lý hoàn
chỉnh. Trong phần xử lý `POST`, đặt nhánh `delete_image` **trước nhánh cập
nhật thông tin sản phẩm thông thường**:

```php
if (isset($_POST['delete_image'])) {
    $imageID = (int) $_POST['delete_image'];

    try {
        $conn->begin_transaction();

        $sqlImage = "
            SELECT
                ProductImageID,
                ImageFile,
                IsPrimary,
                SortOrder
            FROM product_images
            WHERE ProductImageID = ?
              AND ProductID = ?
        ";

        $stmtImage = $conn->prepare($sqlImage);
        $stmtImage->bind_param(
            'ii',
            $imageID,
            $productID
        );
        $stmtImage->execute();

        $imageToDelete =
            $stmtImage->get_result()->fetch_assoc();

        $stmtImage->close();

        if (!$imageToDelete) {
            throw new Exception(
                'Không tìm thấy ảnh cần xóa.'
            );
        }

        $sqlDelete = "
            DELETE FROM product_images
            WHERE ProductImageID = ?
              AND ProductID = ?
        ";

        $stmtDelete = $conn->prepare($sqlDelete);
        $stmtDelete->bind_param(
            'ii',
            $imageID,
            $productID
        );
        $stmtDelete->execute();

        if ($stmtDelete->affected_rows !== 1) {
            throw new Exception(
                'Không thể xóa ảnh.'
            );
        }

        $stmtDelete->close();

        if ((int) $imageToDelete['IsPrimary'] === 1) {
            $sqlNewPrimary = "
                UPDATE product_images
                SET IsPrimary = 1
                WHERE ProductImageID = (
                    SELECT ProductImageID
                    FROM (
                        SELECT ProductImageID
                        FROM product_images
                        WHERE ProductID = ?
                        ORDER BY
                            SortOrder,
                            ProductImageID
                        LIMIT 1
                    ) AS remaining_images
                )
            ";

            $stmtNewPrimary =
                $conn->prepare($sqlNewPrimary);

            $stmtNewPrimary->bind_param(
                'i',
                $productID
            );

            $stmtNewPrimary->execute();
            $stmtNewPrimary->close();
        }

        $deletedSortOrder =
            (int) $imageToDelete['SortOrder'];

        $sqlReorder = "
            UPDATE product_images
            SET SortOrder = SortOrder - 1
            WHERE ProductID = ?
              AND SortOrder > ?
        ";

        $stmtReorder = $conn->prepare($sqlReorder);
        $stmtReorder->bind_param(
            'ii',
            $productID,
            $deletedSortOrder
        );
        $stmtReorder->execute();
        $stmtReorder->close();

        $conn->commit();

        $filePath =
            '/var/www/html/uploads/products/'
            . $imageToDelete['ImageFile'];

        if (file_exists($filePath)) {
            unlink($filePath);
        }

        header(
            'Location: /products/edit.php?id='
            . $productID
            . '&image_deleted=1'
        );
        exit;

    } catch (Throwable $e) {
        $conn->rollback();
        $error = $e->getMessage();
    }
}
```

### 10.1. Vì sao `commit()` trước `unlink()`?

Nếu xóa tập tin trước nhưng transaction MySQL thất bại:

```text
tập tin đã mất
database vẫn còn ImageFile
```

Trong phạm vi Hands-on này, ta dùng:

```text
thay đổi database
      ↓
commit thành công
      ↓
xóa tập tin vật lý
```

`unlink()` không thuộc transaction của MySQL. Trong hệ thống thực tế có yêu
cầu cao hơn, cần thêm cơ chế xử lý khi việc xóa tập tin vật lý thất bại sau
khi database đã commit.

### 10.2. Thêm feedback

Bổ sung:

```php
<?php if (
    isset($_GET['image_deleted'])
    && $_GET['image_deleted'] === '1'
): ?>
    <div class="alert alert-success">
        Đã xóa hình ảnh sản phẩm.
    </div>
<?php endif; ?>
```

---

### 10.3. Thứ tự các nhánh `POST`

Đến đây, phần xử lý `POST` của `edit.php` đã có nhiều chức năng. Cần bảo đảm
các nút chuyên biệt được nhận diện trước nhánh cập nhật sản phẩm:

```text
POST
├── delete_image
├── add_images
├── set_primary_image
└── cập nhật thông tin sản phẩm
```

Mỗi nhánh chuyên biệt phải redirect hoặc kết thúc xử lý sau khi hoàn thành để
không rơi tiếp xuống nhánh cập nhật sản phẩm.

---

## 11. Kiểm thử xóa ảnh phụ

Trước khi xóa, ghi lại `ImageFile`. Chọn một ảnh phụ và xóa.

Kiểm tra CSDL:

```sql
SELECT
    ProductImageID,
    ImageFile,
    IsPrimary,
    SortOrder
FROM product_images
WHERE ProductID = <ProductID-cua-SP006>
ORDER BY SortOrder;
```

Cần thỏa:

```text
ảnh đã xóa không còn trong product_images
SortOrder liên tục từ 1
vẫn chỉ có 1 ảnh chính
```

Kiểm tra tập tin:

```cmd
dir public\uploads\products\<ten-file-vua-xoa>
```

Mong đợi:

```text
File Not Found
```

---

## 12. Kiểm thử xóa ảnh chính

Xóa ảnh đang có `IsPrimary = 1`.

Sau đó:

```sql
SELECT
    ProductImageID,
    ImageFile,
    IsPrimary,
    SortOrder
FROM product_images
WHERE ProductID = <ProductID-cua-SP006>
ORDER BY SortOrder;

SELECT COUNT(*) AS PrimaryImageCount
FROM product_images
WHERE ProductID = <ProductID-cua-SP006>
  AND IsPrimary = 1;
```

Nếu vẫn còn ảnh:

```text
PrimaryImageCount = 1
```

Ảnh còn lại có thứ tự nhỏ nhất trở thành ảnh chính. Trên giao diện, badge **Ảnh chính** phải chuyển sang ảnh mới.

---

## 13. Trường hợp biên: từ 1 ảnh về 0 ảnh

Tạo một sản phẩm thử chỉ có một ảnh, ví dụ `SP007`.

Trước khi xóa:

```sql
SELECT
    p.ProductID,
    p.ProductCode,
    p.ProductName,
    pi.ProductImageID,
    pi.ImageFile,
    pi.IsPrimary,
    pi.SortOrder
FROM products AS p
LEFT JOIN product_images AS pi
    ON pi.ProductID = p.ProductID
WHERE p.ProductCode = 'SP007';
```

Ghi lại `ProductID` và `ImageFile`, sau đó xóa ảnh duy nhất trên trang sửa.

Kiểm tra:

```sql
SELECT
    ProductImageID,
    ProductID,
    ImageFile,
    IsPrimary,
    SortOrder
FROM product_images
WHERE ProductID = <ProductID-cua-SP007>;
```

Mong đợi:

```text
Empty set
```

Nhưng sản phẩm vẫn tồn tại:

```sql
SELECT ProductID, ProductCode, ProductName
FROM products
WHERE ProductID = <ProductID-cua-SP007>;
```

Kiểm tra tập tin:

```cmd
dir public\uploads\products\<ImageFile-cua-SP007>
```

Mong đợi:

```text
File Not Found
```

### Ý nghĩa

Quan hệ thực tế là:

```text
Product 1 ─── 0..N ProductImage
```

Sản phẩm có thể tạm thời không có ảnh và sau đó được bổ sung ảnh trở lại.

---

## 14. Rà soát giao diện quản lý ảnh

Card của mỗi ảnh cần thể hiện:

```text
Ảnh chính
├── badge "Ảnh chính"
└── nút "Xóa ảnh"

Ảnh phụ
├── nút "Đặt làm ảnh chính"
└── nút "Xóa ảnh"
```

Không đặt nút **Đặt làm ảnh chính** trên ảnh vốn đã là ảnh chính.

Sau danh sách ảnh là khu vực:

```text
Thêm hình ảnh
[ Chọn file... ]
[ Thêm ảnh ]
```

Như vậy một trang `edit.php` thực hiện hai nhóm nghiệp vụ:

```text
Thông tin sản phẩm
+
Quản lý hình ảnh sản phẩm
```

---

## 15. Kiểm tra cú pháp và dữ liệu cuối bài

Kiểm tra PHP:

```cmd
docker compose exec web php -l /var/www/html/products/edit.php
```

Cần có:

```text
No syntax errors detected in /var/www/html/products/edit.php
```

Kiểm tra ảnh của `SP006`:

```sql
SELECT
    p.ProductCode,
    pi.ProductImageID,
    pi.ImageFile,
    pi.IsPrimary,
    pi.SortOrder
FROM products AS p
LEFT JOIN product_images AS pi
    ON pi.ProductID = p.ProductID
WHERE p.ProductCode = 'SP006'
ORDER BY pi.SortOrder;
```

Kiểm tra số ảnh chính:

```sql
SELECT COUNT(*) AS PrimaryImageCount
FROM product_images AS pi
JOIN products AS p
    ON p.ProductID = pi.ProductID
WHERE p.ProductCode = 'SP006'
  AND pi.IsPrimary = 1;
```

Nếu `SP006` còn ít nhất một ảnh, `PrimaryImageCount` phải bằng `1`.

---

## 16. Dọn tập tin thử nghiệm và kiểm tra Git

`upload-test.php` của Hands-on 06 chỉ dùng để quan sát cơ chế upload. Khi đã tích hợp thành công, không cần giữ trong mã nguồn chính.

Ảnh upload khi ứng dụng chạy có tên dạng:

```text
product-xxxxxxxxxxxxxxxx.jpg
product-xxxxxxxxxxxxxxxx.png
product-xxxxxxxxxxxxxxxx.webp
```

Đây là dữ liệu runtime, không nên đưa vào Git. Trong `.gitignore`:

```gitignore
# Runtime product uploads
public/uploads/products/product-*
```

Các ảnh mẫu được seed sử dụng, ví dụ:

```text
phone-a-1.webp
phone-a-2.webp
laptop-b-1.webp
mouse-c-1.webp
```

vẫn được Git quản lý vì không bắt đầu bằng `product-`.

Kiểm tra:

```cmd
git status
git diff --check
```

---

## 17. Checklist cuối Hands-on 07

```text
[ ] SP006 có dữ liệu ảnh từ HO06
[ ] edit.php đọc được tất cả ảnh theo ProductID
[ ] ảnh được hiển thị theo SortOrder
[ ] nhận biết được ảnh chính bằng IsPrimary
[ ] đổi được ảnh chính
[ ] sau khi đổi vẫn chỉ có một IsPrimary = 1
[ ] form edit.php có multipart/form-data
[ ] thêm được nhiều ảnh
[ ] kiểm tra MIME và kích thước trước khi lưu
[ ] ảnh mới có SortOrder tiếp theo
[ ] xóa được ảnh phụ
[ ] xóa được ảnh chính
[ ] khi xóa ảnh chính, ảnh còn lại đầu tiên trở thành ảnh chính
[ ] SortOrder được chuẩn hóa sau khi xóa
[ ] tập tin vật lý được xóa bằng unlink()
[ ] database và uploads/products đồng bộ
[ ] sản phẩm vẫn tồn tại khi xóa ảnh cuối cùng
[ ] sản phẩm 0 ảnh có thể thêm ảnh trở lại
[ ] php -l không báo lỗi
[ ] git diff --check không báo lỗi
```

---

## 18. Kiến thức PHP/MySQLi đã sử dụng

| Thành phần | Vai trò |
|---|---|
| `$_FILES` | nhận các tập tin upload |
| `multipart/form-data` | cho phép form gửi tập tin |
| `finfo` | kiểm tra MIME thực tế |
| `random_bytes()` | tạo dữ liệu ngẫu nhiên cho tên tập tin |
| `move_uploaded_file()` | lưu tập tin upload |
| `file_exists()` | kiểm tra tập tin vật lý |
| `unlink()` | xóa tập tin vật lý |
| Prepared Statement | thực thi SQL có tham số |
| `begin_transaction()` | bắt đầu transaction |
| `commit()` | xác nhận thay đổi CSDL |
| `rollback()` | hủy thay đổi khi có lỗi |
| `affected_rows` | kiểm tra số mẩu tin bị tác động |
| `LEFT JOIN` | vẫn lấy sản phẩm khi không còn ảnh |
| `COALESCE()` | xử lý `NULL` khi tìm `MAX(SortOrder)` |
| `ORDER BY` | xác định thứ tự ảnh |
| `LIMIT 1` | chọn ảnh đầu tiên còn lại |

---

## 19. Tổng kết Hands-on 07

Qua Hands-on 06 và Hands-on 07, hình ảnh sản phẩm không còn được xem đơn giản là một chuỗi tên tập tin.

Người học đã xây dựng một vòng đời:

```text
Upload
   ↓
Kiểm tra
   ↓
Đặt tên
   ↓
Lưu tập tin
   ↓
Lưu product_images
   ↓
Hiển thị
   ↓
Đổi ảnh chính
   ↓
Thêm ảnh
   ↓
Xóa ảnh
   ↓
Chuẩn hóa dữ liệu
```

Điểm quan trọng nhất là duy trì sự nhất quán giữa:

```text
products
     ↕
product_images
     ↕
uploads/products/
```

Đây là bước chuyển từ CRUD trên một bảng đơn lẻ sang xử lý một chức năng web có **dữ liệu quan hệ, upload tập tin, transaction và quản lý trạng thái**.