# Hands-on 09 -- Xây dựng trang danh sách và chi tiết sản phẩm

## Mục tiêu

Sau khi hoàn thành Hands-on này, bạn có thể:

-   Xây dựng trang danh sách sản phẩm dành cho người dùng.
-   Truy vấn dữ liệu từ nhiều bảng có quan hệ khóa ngoại bằng điều kiện
    `PK = FK`.
-   Chỉ hiển thị các sản phẩm đang hoạt động.
-   Lấy ảnh chính của sản phẩm để hiển thị trên danh sách.
-   Xây dựng trang chi tiết sản phẩm theo tham số `id` trên URL.
-   Hiển thị thông tin danh mục, nhà cung cấp và nhiều hình ảnh của một
    sản phẩm.
-   Tổ chức hình ảnh sản phẩm responsive bằng Bootstrap và CSS.
-   Bổ sung liên kết Sản phẩm vào thanh điều hướng Frontend.

------------------------------------------------------------------------

## 1. Chuẩn bị

Hands-on này tiếp tục trực tiếp từ **Hands-on 08**.

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

Kiểm tra ứng dụng:

``` cmd
docker compose ps
```

Nếu các container chưa chạy:

``` cmd
docker compose up -d
```

Sau Hands-on 08, phần chính của dự án có dạng:

``` text
public/
├── index.php
├── admin/
│   ├── index.php
│   ├── categories/
│   ├── products/
│   └── shippers/
└── uploads/
    └── products/

src/includes/
├── admin/
│   ├── header.php
│   ├── navbar.php
│   └── footer.php
└── frontend/
    ├── header.php
    ├── navbar.php
    └── footer.php
```

Trong Hands-on này, ta sẽ bổ sung hai trang Frontend:

``` text
public/
├── products.php
└── product-detail.php
```

------------------------------------------------------------------------

## 2. Xác định dữ liệu cần hiển thị

Trang quản trị và trang người dùng có mục đích khác nhau.

Khu vực Admin tập trung vào thao tác quản lý dữ liệu:

``` text
/admin/products/
```

Trong khi đó, Frontend chỉ hiển thị những thông tin phù hợp cho người
dùng.

Trang danh sách sản phẩm cần:

-   mã sản phẩm;
-   tên sản phẩm;
-   giá;
-   danh mục;
-   ảnh chính.

Trang chi tiết cần thêm:

-   mô tả;
-   đơn vị tính;
-   số lượng tồn;
-   nhà cung cấp;
-   các hình ảnh của sản phẩm.

### 2.1. Quan hệ dữ liệu

Bảng `products` có khóa ngoại:

``` text
products.CategoryID
    → categories.CategoryID

products.SupplierID
    → suppliers.SupplierID
```

Trong bài này, ta kết các bảng bằng cách liệt kê bảng trong `FROM` và
đặt điều kiện liên kết khóa ngoại trong `WHERE`.

Ví dụ:

``` sql
FROM
    products p,
    categories c
WHERE
    p.CategoryID = c.CategoryID
```

!!! note "Quy ước của bài thực hành" Hands-on này sử dụng điều kiện liên
kết `PK = FK` để người học quan sát rõ quan hệ giữa khóa chính và khóa
ngoại.

------------------------------------------------------------------------

## 3. Bổ sung liên kết Sản phẩm vào Frontend

Trang sản phẩm sắp được tạo tại:

``` text
/products.php
```

Mở navbar Frontend:

``` cmd
code src\includes\frontend\navbar.php
```

Tìm danh sách:

``` html
<ul class="navbar-nav">
```

Bên dưới mục **Trang chủ**, thêm:

``` html
<li class="nav-item">
    <a class="nav-link" href="/products.php">
        Sản phẩm
    </a>
</li>
```

Nhấn **Ctrl + S** để lưu.

!!! note "Chỉ sửa phần cần thiết" `navbar.php` đã được tạo ở Hands-on 08
nên không cần thay toàn bộ tập tin. Chỉ bổ sung liên kết mới.

------------------------------------------------------------------------

## 4. Tạo trang danh sách sản phẩm

Tạo tập tin:

``` cmd
type nul > public\products.php
```

Mở:

``` cmd
code public\products.php
```

Ở Hands-on này, trang danh sách trước hết được xây dựng ở phiên bản cơ
bản. Chức năng tìm kiếm, lọc và phân trang sẽ được bổ sung ở Hands-on
tiếp theo.

------------------------------------------------------------------------

## 5. Truy vấn danh sách sản phẩm

Nhập phần PHP đầu tập tin:

``` php
<?php

require_once '/var/www/src/config/database.php';

$sql = "
    SELECT
        p.ProductID,
        p.ProductCode,
        p.ProductName,
        p.Price,
        c.CategoryName,

        (
            SELECT pi.ImageFile
            FROM product_images pi
            WHERE pi.ProductID = p.ProductID
              AND pi.IsPrimary = 1
            LIMIT 1
        ) AS ImageFile

    FROM
        products p,
        categories c

    WHERE
        p.CategoryID = c.CategoryID
        AND p.IsActive = 1

    ORDER BY
        p.ProductID DESC
";

$result = $conn->query($sql);

$pageTitle = 'Sản phẩm';

require_once '/var/www/src/includes/frontend/header.php';
require_once '/var/www/src/includes/frontend/navbar.php';
?>
```

### 5.1. Ý nghĩa của truy vấn

Điều kiện:

``` sql
p.CategoryID = c.CategoryID
```

kết bảng `products` với `categories` thông qua khóa ngoại.

Điều kiện:

``` sql
p.IsActive = 1
```

chỉ đưa các sản phẩm đang hoạt động ra Frontend.

Truy vấn con:

``` sql
(
    SELECT pi.ImageFile
    FROM product_images pi
    WHERE pi.ProductID = p.ProductID
      AND pi.IsPrimary = 1
    LIMIT 1
) AS ImageFile
```

lấy **ảnh chính** của từng sản phẩm.

Nếu một sản phẩm chưa có ảnh chính, `ImageFile` nhận giá trị `NULL`. Sản
phẩm vẫn xuất hiện trong danh sách.

------------------------------------------------------------------------

## 6. Hiển thị danh sách bằng Bootstrap Card

Tiếp tục bên dưới phần PHP vừa nhập:

``` php
<main class="container py-5">

    <div class="mb-4">

        <h1>Sản phẩm</h1>

        <p class="text-muted">
            Khám phá các sản phẩm hiện có tại cửa hàng.
        </p>

    </div>

    <?php if ($result->num_rows > 0): ?>

        <div class="row g-4">

            <?php while ($product = $result->fetch_assoc()): ?>

                <div class="col-md-6 col-lg-4">

                    <div class="card h-100">

                        <?php if (!empty($product['ImageFile'])): ?>

                            <div
                                class="bg-light d-flex
                                       align-items-center
                                       justify-content-center
                                       p-3"
                                style="height: 260px;"
                            >

                                <img
                                    src="/uploads/products/<?=
                                        htmlspecialchars(
                                            $product['ImageFile']
                                        )
                                    ?>"
                                    alt="<?=
                                        htmlspecialchars(
                                            $product['ProductName']
                                        )
                                    ?>"
                                    style="
                                        width: 100%;
                                        height: 100%;
                                        object-fit: contain;
                                    "
                                >

                            </div>

                        <?php else: ?>

                            <div
                                class="bg-light d-flex
                                       align-items-center
                                       justify-content-center
                                       text-muted"
                                style="height: 260px;"
                            >
                                Chưa có hình ảnh
                            </div>

                        <?php endif; ?>

                        <div class="card-body d-flex flex-column">

                            <p class="text-muted small mb-1">
                                <?=
                                    htmlspecialchars(
                                        $product['CategoryName']
                                    )
                                ?>
                            </p>

                            <h5 class="card-title">
                                <?=
                                    htmlspecialchars(
                                        $product['ProductName']
                                    )
                                ?>
                            </h5>

                            <p class="text-muted small">
                                Mã sản phẩm:
                                <?=
                                    htmlspecialchars(
                                        $product['ProductCode']
                                    )
                                ?>
                            </p>

                            <p class="fw-bold fs-5 mb-3">
                                <?=
                                    number_format(
                                        (float) $product['Price'],
                                        0,
                                        ',',
                                        '.'
                                    )
                                ?> đ
                            </p>

                            <a
                                href="/product-detail.php?id=<?=
                                    (int) $product['ProductID']
                                ?>"
                                class="btn btn-outline-primary mt-auto"
                            >
                                Xem chi tiết
                            </a>

                        </div>

                    </div>

                </div>

            <?php endwhile; ?>

        </div>

    <?php else: ?>

        <div class="alert alert-info">
            Hiện chưa có sản phẩm nào để hiển thị.
        </div>

    <?php endif; ?>

</main>

<?php

$result->free();

require_once '/var/www/src/includes/frontend/footer.php';
```

Nhấn **Ctrl + S** để lưu.

### 6.1. Vì sao dùng `object-fit: contain`?

Khung ảnh có chiều cao cố định:

``` css
height: 260px;
```

nhưng ảnh thực tế có thể ngang, dọc hoặc vuông.

Thuộc tính:

``` css
object-fit: contain;
```

giúp ảnh:

-   giữ đúng tỷ lệ;
-   không bị kéo méo;
-   không bị cắt nội dung;
-   nằm gọn trong vùng ảnh của Card.

Kết hợp với:

``` html
<div class="card h-100">
```

các Card trong cùng hàng có chiều cao cân đối hơn.

------------------------------------------------------------------------

## 7. Kiểm tra trang danh sách

Kiểm tra cú pháp:

``` cmd
docker compose exec web php -l /var/www/html/products.php
```

Kết quả đúng:

``` text
No syntax errors detected in /var/www/html/products.php
```

Mở:

``` text
http://localhost:8080/products.php
```

Kiểm tra:

-   chỉ sản phẩm có `IsActive = 1` xuất hiện;
-   tên danh mục hiển thị đúng;
-   giá lấy từ trường `Price`;
-   ảnh chính hiển thị;
-   sản phẩm chưa có ảnh vẫn xuất hiện;
-   ảnh không bị kéo méo hoặc cắt;
-   các Card hiển thị tốt ở nhiều kích thước màn hình;
-   nút **Xem chi tiết** chứa đúng `id` sản phẩm.

!!! success "Checkpoint" Chỉ tiếp tục khi trang `/products.php` hiển thị
đúng dữ liệu và hình ảnh.

------------------------------------------------------------------------

## 8. Tạo trang chi tiết sản phẩm

Tạo tập tin:

``` cmd
type nul > public\product-detail.php
```

Mở:

``` cmd
code public\product-detail.php
```

Trang này nhận ProductID từ URL.

Ví dụ:

``` text
/product-detail.php?id=3
```

Trong đó:

``` text
id=3
```

cho biết người dùng muốn xem sản phẩm có `ProductID = 3`.

------------------------------------------------------------------------

## 9. Nhận và kiểm tra ProductID

Nhập:

``` php
<?php

require_once '/var/www/src/config/database.php';

$productID = isset($_GET['id'])
    ? (int) $_GET['id']
    : 0;

if ($productID <= 0) {
    header('Location: /products.php');
    exit;
}
```

Việc ép kiểu:

``` php
(int) $_GET['id']
```

đảm bảo biến `$productID` được xử lý dưới dạng số nguyên.

Nếu `id` không hợp lệ, người dùng được đưa về:

``` text
/products.php
```

------------------------------------------------------------------------

## 10. Truy vấn thông tin chi tiết

Tiếp tục bên dưới:

``` php
$sql = "
    SELECT
        p.ProductID,
        p.ProductCode,
        p.ProductName,
        p.Description,
        p.Unit,
        p.Price,
        p.StockQuantity,
        c.CategoryName,
        s.SupplierName

    FROM
        products p,
        categories c,
        suppliers s

    WHERE
        p.CategoryID = c.CategoryID
        AND p.SupplierID = s.SupplierID
        AND p.ProductID = ?
        AND p.IsActive = 1
";

$stmt = $conn->prepare($sql);

$stmt->bind_param(
    'i',
    $productID
);

$stmt->execute();

$result = $stmt->get_result();

$product = $result->fetch_assoc();

$result->free();
$stmt->close();

if (!$product) {
    header('Location: /products.php');
    exit;
}
```

### 10.1. Quan hệ giữa ba bảng

Hai điều kiện:

``` sql
p.CategoryID = c.CategoryID
```

và:

``` sql
p.SupplierID = s.SupplierID
```

kết `products` với `categories` và `suppliers` thông qua các khóa ngoại.

Điều kiện:

``` sql
p.ProductID = ?
```

sử dụng placeholder `?`.

Giá trị `$productID` được truyền vào bằng:

``` php
$stmt->bind_param(
    'i',
    $productID
);
```

Ký tự:

``` text
i
```

cho biết tham số là số nguyên.

------------------------------------------------------------------------

## 11. Lấy danh sách hình ảnh của sản phẩm

Sau khi đã lấy thông tin sản phẩm, tiếp tục:

``` php
$sqlImages = "
    SELECT
        ImageID,
        ImageFile,
        IsPrimary

    FROM
        product_images

    WHERE
        ProductID = ?

    ORDER BY
        IsPrimary DESC,
        ImageID ASC
";

$stmtImages = $conn->prepare($sqlImages);

$stmtImages->bind_param(
    'i',
    $productID
);

$stmtImages->execute();

$imageResult = $stmtImages->get_result();

$images = [];

while ($image = $imageResult->fetch_assoc()) {
    $images[] = $image;
}

$imageResult->free();
$stmtImages->close();
```

Ảnh chính được sắp trước nhờ:

``` sql
ORDER BY
    IsPrimary DESC,
    ImageID ASC
```

Sau đó đặt tiêu đề và nạp layout Frontend:

``` php
$pageTitle = $product['ProductName'];

require_once '/var/www/src/includes/frontend/header.php';
require_once '/var/www/src/includes/frontend/navbar.php';
?>
```

------------------------------------------------------------------------

## 12. Hiển thị chi tiết sản phẩm

Tiếp tục:

``` php
<main class="container py-5">

    <div class="mb-4">

        <a
            href="/products.php"
            class="text-decoration-none"
        >
            &larr; Quay lại danh sách sản phẩm
        </a>

    </div>

    <div class="row g-5">

        <div class="col-lg-6">

            <?php if (!empty($images)): ?>

                <div
                    class="bg-light d-flex
                           align-items-center
                           justify-content-center
                           p-3 mb-3"
                    style="height: 420px;"
                >

                    <img
                        src="/uploads/products/<?=
                            htmlspecialchars(
                                $images[0]['ImageFile']
                            )
                        ?>"
                        alt="<?=
                            htmlspecialchars(
                                $product['ProductName']
                            )
                        ?>"
                        style="
                            width: 100%;
                            height: 100%;
                            object-fit: contain;
                        "
                    >

                </div>

                <?php if (count($images) > 1): ?>

                    <div class="row g-2">

                        <?php foreach ($images as $image): ?>

                            <div class="col-4">

                                <div
                                    class="border rounded
                                           bg-light
                                           d-flex
                                           align-items-center
                                           justify-content-center
                                           p-2"
                                    style="height: 120px;"
                                >

                                    <img
                                        src="/uploads/products/<?=
                                            htmlspecialchars(
                                                $image['ImageFile']
                                            )
                                        ?>"
                                        alt="<?=
                                            htmlspecialchars(
                                                $product['ProductName']
                                            )
                                        ?>"
                                        style="
                                            width: 100%;
                                            height: 100%;
                                            object-fit: contain;
                                        "
                                    >

                                </div>

                            </div>

                        <?php endforeach; ?>

                    </div>

                <?php endif; ?>

            <?php else: ?>

                <div
                    class="bg-light d-flex
                           align-items-center
                           justify-content-center
                           text-muted"
                    style="height: 420px;"
                >
                    Chưa có hình ảnh
                </div>

            <?php endif; ?>

        </div>

        <div class="col-lg-6">

            <p class="text-muted mb-2">
                <?=
                    htmlspecialchars(
                        $product['CategoryName']
                    )
                ?>
            </p>

            <h1 class="mb-3">
                <?=
                    htmlspecialchars(
                        $product['ProductName']
                    )
                ?>
            </h1>

            <p class="text-muted">
                Mã sản phẩm:
                <?=
                    htmlspecialchars(
                        $product['ProductCode']
                    )
                ?>
            </p>

            <p class="fs-3 fw-bold">
                <?=
                    number_format(
                        (float) $product['Price'],
                        0,
                        ',',
                        '.'
                    )
                ?> đ
            </p>

            <hr>

            <dl class="row">

                <dt class="col-sm-4">
                    Danh mục
                </dt>

                <dd class="col-sm-8">
                    <?=
                        htmlspecialchars(
                            $product['CategoryName']
                        )
                    ?>
                </dd>

                <dt class="col-sm-4">
                    Nhà cung cấp
                </dt>

                <dd class="col-sm-8">
                    <?=
                        htmlspecialchars(
                            $product['SupplierName']
                        )
                    ?>
                </dd>

                <dt class="col-sm-4">
                    Đơn vị tính
                </dt>

                <dd class="col-sm-8">
                    <?=
                        htmlspecialchars(
                            $product['Unit'] ?? ''
                        )
                    ?>
                </dd>

                <dt class="col-sm-4">
                    Tồn kho
                </dt>

                <dd class="col-sm-8">
                    <?= (int) $product['StockQuantity'] ?>
                </dd>

            </dl>

            <?php if (!empty($product['Description'])): ?>

                <hr>

                <h5>Mô tả sản phẩm</h5>

                <p>
                    <?=
                        nl2br(
                            htmlspecialchars(
                                $product['Description']
                            )
                        )
                    ?>
                </p>

            <?php endif; ?>

        </div>

    </div>

</main>

<?php
require_once '/var/www/src/includes/frontend/footer.php';
```

Nhấn **Ctrl + S** để lưu.

------------------------------------------------------------------------

## 13. Kiểm tra trang chi tiết

Kiểm tra cú pháp:

``` cmd
docker compose exec web php -l /var/www/html/product-detail.php
```

Mở trang danh sách:

``` text
http://localhost:8080/products.php
```

Nhấn **Xem chi tiết** trên một sản phẩm.

Kiểm tra:

-   đúng sản phẩm được mở;
-   tên, mã, giá và mô tả hiển thị đúng;
-   danh mục hiển thị đúng;
-   nhà cung cấp hiển thị đúng;
-   đơn vị tính và tồn kho hiển thị đúng;
-   ảnh chính hiển thị ở vùng ảnh lớn;
-   các ảnh còn lại xuất hiện bên dưới;
-   sản phẩm không có ảnh vẫn hiển thị được;
-   nút quay lại danh sách hoạt động.

Thử URL không hợp lệ:

``` text
http://localhost:8080/product-detail.php?id=0
```

và một `id` không tồn tại.

Ứng dụng phải quay về:

``` text
/products.php
```

------------------------------------------------------------------------

## 14. Kiểm tra responsive

Mở trang:

``` text
/products.php
```

và:

``` text
/product-detail.php?id=...
```

Trong trình duyệt Chrome hoặc Edge:

1.  Nhấn **F12**.
2.  Nhấn **Ctrl + Shift + M** để bật chế độ mô phỏng thiết bị.
3.  Thử một số kích thước màn hình.

Ở trang danh sách:

-   màn hình lớn hiển thị tối đa 3 Card trên một hàng;
-   màn hình trung bình hiển thị 2 Card;
-   màn hình nhỏ các Card tự xếp theo chiều dọc;
-   ảnh giữ nguyên tỷ lệ.

Ở trang chi tiết:

-   màn hình lớn chia thành vùng ảnh và vùng thông tin;
-   màn hình nhỏ hai vùng tự xếp theo chiều dọc;
-   ảnh không tràn khỏi khung.

------------------------------------------------------------------------

## 15. Kiểm tra toàn bộ Hands-on 09

Đánh dấu sau khi kiểm tra:

  Nội dung                Kết quả mong đợi
  ----------------------- -------------------------------------------------------
  Navbar Frontend         Có liên kết Sản phẩm
  `/products.php`         Hiển thị danh sách sản phẩm đang hoạt động
  Category                Tên danh mục hiển thị đúng
  Ảnh chính               Hiển thị đúng từ `/uploads/products/`
  Sản phẩm chưa có ảnh    Vẫn xuất hiện
  Card                    Responsive và chiều cao cân đối
  Xem chi tiết            Truyền đúng ProductID qua `id`
  `/product-detail.php`   Hiển thị đúng sản phẩm
  Supplier                Tên nhà cung cấp hiển thị đúng
  Nhiều ảnh               Hiển thị được danh sách ảnh
  `id` không hợp lệ       Quay về `/products.php`
  Responsive              Danh sách và chi tiết hoạt động tốt trên màn hình nhỏ

!!! success "Hoàn thành" Khi tất cả nội dung trên hoạt động đúng, bạn đã
hoàn thành phần Frontend cơ bản cho sản phẩm.

------------------------------------------------------------------------

## 16. Cấu trúc sau Hands-on 09

Chạy:

``` cmd
tree public /F
```

Phần Frontend đã có thêm:

``` text
public/
├── index.php
├── products.php
├── product-detail.php
├── admin/
│   ├── index.php
│   ├── categories/
│   ├── products/
│   └── shippers/
└── uploads/
    └── products/
```

Hai khu vực hiện có vai trò rõ ràng:

``` text
Frontend
├── /
├── /products.php
└── /product-detail.php?id=...

Admin
├── /admin/
├── /admin/categories/
├── /admin/products/
└── /admin/shippers/
```

------------------------------------------------------------------------

## 17. Lưu phiên bản bằng Git

Kiểm tra:

``` cmd
git status
```

Kiểm tra lỗi khoảng trắng trước khi commit:

``` cmd
git diff --check
```

Nếu kết quả ổn, thêm các thay đổi:

``` cmd
git add .
```

Kiểm tra:

``` cmd
git diff --cached --check
```

Xem tóm tắt:

``` cmd
git diff --cached --stat
```

Commit:

``` cmd
git commit -m "feat: add product storefront and detail page"
```

Đẩy lên GitHub:

``` cmd
git push
```

------------------------------------------------------------------------

## 18. Kết quả đạt được

Sau Hands-on 09, ứng dụng không còn chỉ có các trang quản trị.

Người dùng đã có thể:

``` text
Trang chủ
    ↓
Danh sách sản phẩm
    ↓
Xem chi tiết sản phẩm
```

Dữ liệu hiển thị trên Frontend vẫn lấy trực tiếp từ cơ sở dữ liệu mà khu
vực Admin đang quản lý.

Đây là nền tảng để tiếp tục bổ sung:

-   tìm kiếm sản phẩm;
-   lọc theo danh mục;
-   phân trang.

Các chức năng này sẽ được thực hiện ở Hands-on tiếp theo.

------------------------------------------------------------------------

## Câu hỏi củng cố

1.  Vì sao Frontend chỉ hiển thị sản phẩm có `IsActive = 1`?
2.  Điều kiện `p.CategoryID = c.CategoryID` thể hiện quan hệ nào giữa
    hai bảng?
3.  Vì sao trang danh sách chỉ lấy ảnh có `IsPrimary = 1`?
4.  Vì sao sử dụng `object-fit: contain` khi hiển thị hình ảnh sản phẩm?
5.  Tham số `id` trong `/product-detail.php?id=3` được dùng để làm gì?
6.  Vì sao truy vấn chi tiết sản phẩm sử dụng prepared statement?
7.  Vì sao danh sách hình ảnh được sắp xếp theo `IsPrimary DESC`?
8.  Admin Products và Frontend Products khác nhau về mục đích sử dụng
    như thế nào?

## Bài tập củng cố

Thực hiện các kiểm tra sau với dữ liệu của bạn:

1.  Chọn một sản phẩm đang hoạt động và kiểm tra toàn bộ thông tin giữa
    Admin và Frontend.
2.  Chọn một sản phẩm có nhiều ảnh và xác nhận ảnh chính được hiển thị
    trước.
3.  Chọn hoặc tạo một sản phẩm chưa có ảnh và kiểm tra cách Frontend
    hiển thị.
4.  Thay đổi trạng thái `IsActive` của một sản phẩm trong khu vực Admin
    và quan sát sự thay đổi trên Frontend.
