# Hands-on 15 -- REST API và Fetch API

## 1. Mục tiêu

Sau khi hoàn thành bài thực hành, sinh viên có thể:

-   Giải thích vai trò cơ bản của REST API trong ứng dụng web.
-   Phân biệt PHP tạo trực tiếp HTML với frontend lấy dữ liệu từ API.
-   Xây dựng điểm truy cập API (endpoint) bằng PHP và MySQLi.
-   Trả dữ liệu JSON từ server.
-   Sử dụng `GET` và tham số truy vấn để tìm kiếm, lọc dữ liệu.
-   Sử dụng `fetch()` để gọi API và cập nhật giao diện mà không tải lại
    toàn bộ trang.
-   Quan sát request/response bằng Developer Tools.

## 2. Bối cảnh

Ở các Hands-on trước, luồng xử lý là:

``` text
Trình duyệt → PHP → MySQL → PHP tạo HTML → Trình duyệt
```

Trong bài này:

``` text
Frontend → fetch() → REST API → PHP + MySQLi → MySQL
                         ↓
                        JSON
                         ↓
              JavaScript cập nhật giao diện
```

**REST API** là cách tổ chức giao tiếp giữa các ứng dụng qua HTTP, trong
đó dữ liệu thường được biểu diễn bằng JSON. **Điểm truy cập API
(endpoint)** là địa chỉ cụ thể mà client gửi yêu cầu đến.

Bài này xây dựng:

``` text
/api/categories.php
/api/products.php
```

## 3. Kết quả cần đạt

Dự án có thêm:

``` text
public/
├── api/
│   ├── categories.php
│   └── products.php
└── products-api.php
```

`categories.php` cung cấp danh mục; `products.php` cung cấp sản phẩm và
hỗ trợ tìm kiếm/lọc; `products-api.php` dùng JavaScript và `fetch()` để
lấy dữ liệu. Các trang sản phẩm đã xây dựng trước đó vẫn được giữ
nguyên.

## 4. Khởi động môi trường

``` cmd
cd /d D:\PTUDW-ST-2026\php-mysql-sales
docker compose up -d
docker compose ps
```

Mở `http://localhost:8080/` và xác nhận ứng dụng hoạt động.

## 5. Xây dựng API danh mục

Tạo thư mục `public/api`, sau đó tạo tập tin mới:

``` text
public/api/categories.php
```

Vì đây là tập tin mới, nhập toàn bộ mã nguồn:

``` php
<?php

require_once '/var/www/src/config/database.php';

header('Content-Type: application/json; charset=utf-8');

$sql = "
    SELECT
        CategoryID,
        CategoryName
    FROM categories
    ORDER BY CategoryName
";

$result = $conn->query($sql);

$categories = [];

while ($row = $result->fetch_assoc()) {

    $categories[] = [
        'id' => (int) $row['CategoryID'],
        'name' => $row['CategoryName']
    ];
}

$result->free();
$conn->close();

echo json_encode(
    [
        'success' => true,
        'data' => $categories
    ],
    JSON_UNESCAPED_UNICODE
);
```

`Content-Type: application/json` cho client biết response là JSON.
`json_encode()` chuyển mảng PHP thành JSON; `JSON_UNESCAPED_UNICODE` giữ
tiếng Việt ở dạng dễ đọc.

Kiểm tra:

``` cmd
docker compose exec web php -l /var/www/html/api/categories.php
```

Sau đó mở:

``` text
http://localhost:8080/api/categories.php
```

Kết quả có dạng:

``` json
{
    "success": true,
    "data": [
        {"id": 1, "name": "Điện thoại"},
        {"id": 2, "name": "Máy tính"},
        {"id": 3, "name": "Phụ kiện"}
    ]
}
```

## 6. Xây dựng API sản phẩm

Tạo tập tin mới:

``` text
public/api/products.php
```

API hỗ trợ:

``` text
/api/products.php
/api/products.php?keyword=Laptop
/api/products.php?category=1
/api/products.php?keyword=Bluetooth&category=3
```

Nhập mã nguồn:

``` php
<?php

require_once '/var/www/src/config/database.php';

header('Content-Type: application/json; charset=utf-8');

$keyword =
    trim($_GET['keyword'] ?? '');

$categoryID =
    (int) ($_GET['category'] ?? 0);

$sql = "
    SELECT
        p.ProductID,
        p.ProductCode,
        p.ProductName,
        p.Price,
        p.StockQuantity,
        c.CategoryName,
        (
            SELECT pi.ImageFile
            FROM product_images pi
            WHERE pi.ProductID = p.ProductID
              AND pi.IsPrimary = 1
            LIMIT 1
        ) AS ImageFile
    FROM products p, categories c
    WHERE p.CategoryID = c.CategoryID
      AND p.IsActive = 1
";

if ($keyword !== '') {
    $sql .= "
        AND (
            p.ProductName LIKE ?
            OR p.ProductCode LIKE ?
        )
    ";
}

if ($categoryID > 0) {
    $sql .= "
        AND p.CategoryID = ?
    ";
}

$sql .= "
    ORDER BY p.ProductID DESC
";

$stmt = $conn->prepare($sql);

if ($keyword !== '' && $categoryID > 0) {

    $searchKeyword =
        '%' . $keyword . '%';

    $stmt->bind_param(
        'ssi',
        $searchKeyword,
        $searchKeyword,
        $categoryID
    );

} elseif ($keyword !== '') {

    $searchKeyword =
        '%' . $keyword . '%';

    $stmt->bind_param(
        'ss',
        $searchKeyword,
        $searchKeyword
    );

} elseif ($categoryID > 0) {

    $stmt->bind_param(
        'i',
        $categoryID
    );
}

$stmt->execute();

$result =
    $stmt->get_result();

$products = [];

while ($row = $result->fetch_assoc()) {

    $products[] = [
        'id' => (int) $row['ProductID'],
        'code' => $row['ProductCode'],
        'name' => $row['ProductName'],
        'price' => (float) $row['Price'],
        'stock' => (int) $row['StockQuantity'],
        'category' => $row['CategoryName'],
        'image' => $row['ImageFile']
    ];
}

$result->free();
$stmt->close();
$conn->close();

echo json_encode(
    [
        'success' => true,
        'data' => $products
    ],
    JSON_UNESCAPED_UNICODE
);
```

API vẫn dùng Prepared Statement khi nhận dữ liệu từ URL. Quan hệ giữa
`products` và `categories` tiếp tục dùng điều kiện khóa:

``` sql
FROM products p, categories c
WHERE p.CategoryID = c.CategoryID
```

Kiểm tra:

``` cmd
docker compose exec web php -l /var/www/html/api/products.php
```

Lần lượt mở bốn URL kiểm thử ở đầu mục này và xác nhận trường `data`
thay đổi đúng theo tham số.

## 7. Xây dựng frontend sử dụng Fetch API

Tạo tập tin mới:

``` text
public/products-api.php
```

Trang này không trực tiếp truy vấn bảng `products` hoặc `categories`. Dữ
liệu được lấy qua API.

Nhập mã nguồn:

``` php
<?php

require_once '/var/www/src/config/session.php';

$pageTitle = 'Sản phẩm từ REST API';

require_once '/var/www/src/includes/frontend/header.php';
require_once '/var/www/src/includes/frontend/navbar.php';

?>

<div class="container py-4">

    <h1 class="h3 mb-4">
        Sản phẩm từ REST API
    </h1>

    <form id="filter-form" class="row g-3 mb-4">

        <div class="col-md-6">
            <label for="keyword" class="form-label">
                Từ khóa
            </label>

            <input
                type="text"
                id="keyword"
                class="form-control"
                placeholder="Tên hoặc mã sản phẩm"
            >
        </div>

        <div class="col-md-4">
            <label for="category" class="form-label">
                Danh mục
            </label>

            <select id="category" class="form-select">
                <option value="">
                    Tất cả danh mục
                </option>
            </select>
        </div>

        <div class="col-md-2 d-flex align-items-end">
            <button type="submit" class="btn btn-primary w-100">
                Tìm kiếm
            </button>
        </div>

    </form>

    <div id="loading" class="alert alert-info d-none">
        Đang tải dữ liệu sản phẩm...
    </div>

    <div
        id="error-message"
        class="alert alert-danger d-none"
    ></div>

    <div id="product-list" class="row g-4"></div>

</div>

<script>

const filterForm =
    document.getElementById('filter-form');

const keywordElement =
    document.getElementById('keyword');

const categoryElement =
    document.getElementById('category');

const loadingElement =
    document.getElementById('loading');

const errorElement =
    document.getElementById('error-message');

const productListElement =
    document.getElementById('product-list');


function formatPrice(price) {

    return Number(price).toLocaleString('vi-VN')
        + ' đ';
}


function escapeHtml(value) {

    const element =
        document.createElement('div');

    element.textContent =
        String(value ?? '');

    return element.innerHTML;
}


function createProductCard(product) {

    const productName =
        escapeHtml(product.name);

    const productCode =
        escapeHtml(product.code);

    const categoryName =
        escapeHtml(product.category);

    const imageUrl = product.image
        ? '/uploads/products/'
            + encodeURIComponent(product.image)
        : '';

    const imageHtml = imageUrl
        ? `
            <img
                src="${imageUrl}"
                class="card-img-top"
                alt="${productName}"
                style="
                    height: 220px;
                    object-fit: contain;
                "
            >
        `
        : `
            <div
                class="
                    d-flex
                    align-items-center
                    justify-content-center
                    bg-light
                    text-muted
                "
                style="height: 220px;"
            >
                Chưa có ảnh
            </div>
        `;

    return `
        <div class="col-md-6 col-lg-4">
            <div class="card h-100">

                ${imageHtml}

                <div class="card-body">

                    <div class="text-muted small mb-1">
                        ${categoryName}
                    </div>

                    <h2 class="h5">
                        ${productName}
                    </h2>

                    <div class="mb-2">
                        Mã: ${productCode}
                    </div>

                    <div class="fw-bold mb-2">
                        ${formatPrice(product.price)}
                    </div>

                    <div class="mb-3">
                        Tồn kho: ${product.stock}
                    </div>

                    <a
                        href="/product-detail.php?id=${product.id}"
                        class="btn btn-primary"
                    >
                        Xem chi tiết
                    </a>

                </div>
            </div>
        </div>
    `;
}


async function loadCategories() {

    try {

        const response =
            await fetch('/api/categories.php');

        if (!response.ok) {
            throw new Error(
                'Không thể tải danh mục.'
            );
        }

        const result =
            await response.json();

        if (!result.success) {
            throw new Error(
                'Dữ liệu danh mục không hợp lệ.'
            );
        }

        result.data.forEach(category => {

            const option =
                document.createElement('option');

            option.value =
                category.id;

            option.textContent =
                category.name;

            categoryElement.appendChild(option);
        });

    } catch (error) {

        errorElement.textContent =
            error.message;

        errorElement.classList.remove('d-none');
    }
}


async function loadProducts() {

    loadingElement.classList.remove('d-none');
    errorElement.classList.add('d-none');
    productListElement.innerHTML = '';

    const keyword =
        keywordElement.value.trim();

    const category =
        categoryElement.value;

    const params =
        new URLSearchParams();

    if (keyword !== '') {
        params.set('keyword', keyword);
    }

    if (category !== '') {
        params.set('category', category);
    }

    let apiUrl =
        '/api/products.php';

    if (params.toString() !== '') {
        apiUrl +=
            '?' + params.toString();
    }

    try {

        const response =
            await fetch(apiUrl);

        if (!response.ok) {
            throw new Error(
                'Không thể tải dữ liệu sản phẩm.'
            );
        }

        const result =
            await response.json();

        if (!result.success) {
            throw new Error(
                'API trả về kết quả không hợp lệ.'
            );
        }

        if (result.data.length === 0) {

            productListElement.innerHTML = `
                <div class="col-12">
                    <div class="alert alert-warning">
                        Không tìm thấy sản phẩm phù hợp.
                    </div>
                </div>
            `;

            return;
        }

        productListElement.innerHTML =
            result.data
                .map(createProductCard)
                .join('');

    } catch (error) {

        errorElement.textContent =
            error.message;

        errorElement.classList.remove('d-none');

    } finally {

        loadingElement.classList.add('d-none');
    }
}


filterForm.addEventListener(
    'submit',
    function (event) {

        event.preventDefault();

        loadProducts();
    }
);


async function initializePage() {

    await loadCategories();

    await loadProducts();
}


initializePage();

</script>

<?php

require_once '/var/www/src/includes/frontend/footer.php';

?>
```

## 8. Phân tích hoạt động

`fetch('/api/categories.php')` gửi HTTP request để lấy danh mục.
`response.json()` chuyển JSON nhận được thành đối tượng JavaScript.

`URLSearchParams` tạo query string. Ví dụ:

``` text
keyword=Bluetooth&category=3
```

sẽ tạo request:

``` text
/api/products.php?keyword=Bluetooth&category=3
```

Trong sự kiện submit:

``` javascript
event.preventDefault();
```

ngăn form tải lại trang. JavaScript gọi `loadProducts()` và chỉ cập nhật
vùng `#product-list`.

## 9. Kiểm thử giao diện

Kiểm tra cú pháp:

``` cmd
docker compose exec web php -l /var/www/html/products-api.php
```

Mở:

``` text
http://localhost:8080/products-api.php
```

Kiểm thử:

  -----------------------------------------------------------------------
  Trường hợp                          Kết quả mong đợi
  ----------------------------------- -----------------------------------
  Không nhập từ khóa, không chọn danh Tất cả sản phẩm đang hoạt động
  mục                                 

  Nhập `Laptop`                       Sản phẩm có tên hoặc mã phù hợp

  Chọn `Phụ kiện`                     Sản phẩm thuộc danh mục Phụ kiện

  `Bluetooth` + `Phụ kiện`            Sản phẩm thỏa cả hai điều kiện

  Từ khóa không tồn tại               Thông báo không tìm thấy sản phẩm
  -----------------------------------------------------------------------

Trong quá trình tìm kiếm, toàn bộ trang không được tải lại.

## 10. Quan sát request bằng Developer Tools

Mở `products-api.php`, nhấn `F12`, chọn:

``` text
Network → Fetch/XHR
```

Tải lại trang. Khi khởi tạo, có thể quan sát:

``` text
categories.php
products.php
```

Chọn `products.php` và kiểm tra:

``` text
Request Method: GET
Status Code: 200
```

Nhập `Bluetooth`, chọn `Phụ kiện` và bấm **Tìm kiếm**. Request mới có
URL tương tự:

``` text
/api/products.php?keyword=Bluetooth&category=3
```

Trong **Response**, dữ liệu được trả về dưới dạng JSON.

Điểm cần chú ý: trình duyệt không gửi lại request đến `products-api.php`
để dựng lại toàn bộ trang; JavaScript chỉ gọi API và cập nhật danh sách
sản phẩm.

## 11. So sánh với cách truyền thống

Cách PHP render truyền thống:

``` text
Tìm kiếm → Submit → PHP → MySQL → tạo lại HTML → tải lại trang
```

Cách REST API + Fetch API:

``` text
Tìm kiếm → JavaScript → fetch() → API → MySQL
                                      ↓
                                     JSON
                                      ↓
                         cập nhật danh sách sản phẩm
```

REST API không có nghĩa PHP render truyền thống không còn phù hợp. PHP
render trực tiếp đơn giản và phù hợp với nhiều website. API đặc biệt hữu
ích khi cần giao diện tương tác động, nhiều loại client cùng sử dụng dữ
liệu, tích hợp hệ thống hoặc tách frontend và backend.

## 12. Khái niệm cần ghi nhớ

-   **API (Application Programming Interface):** giao diện cho phép các
    chương trình giao tiếp với nhau.
-   **REST (Representational State Transfer):** kiểu kiến trúc thường
    dùng để thiết kế API trên nền HTTP.
-   **Endpoint:** điểm truy cập cụ thể của API, ví dụ
    `/api/products.php`.
-   **JSON:** định dạng dữ liệu thường dùng để trao đổi dữ liệu giữa API
    và client.
-   **Fetch API:** giao diện JavaScript cho phép trình duyệt gửi HTTP
    request và xử lý response.
-   **GET:** phương thức HTTP dùng để đọc dữ liệu trong bài này.

Trong REST API còn thường gặp `POST` để tạo dữ liệu, `PUT` để cập nhật
và `DELETE` để xóa. Các phương thức này có thể được phát triển thêm khi
xây dựng API CRUD hoàn chỉnh.

## 13. Kiểm tra và lưu phiên bản

Kiểm tra cú pháp:

``` cmd
docker compose exec web php -l /var/www/html/api/categories.php
docker compose exec web php -l /var/www/html/api/products.php
docker compose exec web php -l /var/www/html/products-api.php
```

Thêm các tập tin:

``` cmd
git add public/api/categories.php
git add public/api/products.php
git add public/products-api.php
```

Kiểm tra:

``` cmd
git status
git diff --cached --check
git diff --cached --stat
```

Commit và đẩy lên repository:

``` cmd
git commit -m "Add REST product API and fetch frontend"
git push
git status
```

Kết quả cuối mong đợi:

``` text
nothing to commit, working tree clean
```

## 14. Tổng kết

Bài thực hành đã mở rộng ứng dụng từ cách PHP tạo trực tiếp giao diện
sang mô hình:

``` text
MySQL → PHP REST API → JSON → fetch() → JavaScript → Giao diện
```

Sinh viên xây dựng hai endpoint cho danh mục và sản phẩm; API sản phẩm
nhận tham số tìm kiếm, lọc; frontend sử dụng `fetch()` để cập nhật danh
sách mà không tải lại toàn bộ trang.

Đây là bước chuyển từ ứng dụng PHP/MySQL truyền thống sang ứng dụng có
lớp API, tạo nền tảng để dữ liệu có thể tiếp tục được sử dụng bởi giao
diện web, ứng dụng di động hoặc các hệ thống tích hợp khác.
