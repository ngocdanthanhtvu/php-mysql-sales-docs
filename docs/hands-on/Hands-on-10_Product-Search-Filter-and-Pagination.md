# Hands-on 10 -- Tìm kiếm, lọc và phân trang sản phẩm

## Mục tiêu

Sau khi hoàn thành Hands-on này, bạn có thể:

-   Nhận dữ liệu tìm kiếm và lọc từ URL bằng phương thức `GET`.
-   Xây dựng bộ lọc sản phẩm theo từ khóa và danh mục.
-   Tạo câu truy vấn SQL có điều kiện tùy theo dữ liệu người dùng nhập.
-   Sử dụng prepared statement khi truy vấn có tham số.
-   Đếm tổng số sản phẩm phù hợp bằng `COUNT(*)`.
-   Phân trang dữ liệu bằng `LIMIT` và `OFFSET`.
-   Giữ nguyên điều kiện tìm kiếm và lọc khi chuyển trang.
-   Xử lý các trường hợp số trang không hợp lệ.
-   Kiểm thử kết hợp tìm kiếm, lọc và phân trang trên Frontend.

------------------------------------------------------------------------

## 1. Chuẩn bị

Hands-on này tiếp tục trực tiếp từ **Hands-on 09**.

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

Mở trang sản phẩm đã hoàn thành ở Hands-on 09:

``` text
http://localhost:8080/products.php
```

Ở thời điểm này, trang đã có thể:

-   hiển thị sản phẩm đang hoạt động;
-   hiển thị tên danh mục;
-   hiển thị ảnh chính;
-   mở trang chi tiết sản phẩm.

Trong Hands-on 10, ta **mở rộng chính tập tin `public/products.php`**,
không tạo một trang danh sách mới.

------------------------------------------------------------------------

## 2. Xác định chức năng cần bổ sung

Khi số lượng sản phẩm tăng, việc hiển thị toàn bộ sản phẩm trên một
trang không còn thuận tiện.

Ta bổ sung ba chức năng:

``` text
Tìm kiếm
    +
Lọc theo danh mục
    +
Phân trang
```

Ví dụ URL sau:

``` text
/products.php?keyword=phone&category=2&page=3
```

chứa ba tham số:

  Tham số           Ý nghĩa
  ----------------- -----------------------------------------------------
  `keyword=phone`   Tìm theo từ khóa `phone`
  `category=2`      Chỉ lấy sản phẩm thuộc danh mục có `CategoryID = 2`
  `page=3`          Hiển thị trang số 3

Các tham số này được gửi bằng phương thức `GET`, vì vậy chúng xuất hiện
trên URL và có thể được giữ lại khi người dùng chuyển trang.

------------------------------------------------------------------------

## 3. Lấy danh sách danh mục cho bộ lọc

Mở:

``` cmd
code public\products.php
```

Ở đầu tập tin hiện có:

``` php
require_once '/var/www/src/config/database.php';
```

Ngay bên dưới dòng này, thêm:

``` php
$sqlCategories = "
    SELECT
        CategoryID,
        CategoryName
    FROM categories
    ORDER BY CategoryName
";

$categoryResult = $conn->query($sqlCategories);

$categories = [];

while ($category = $categoryResult->fetch_assoc()) {
    $categories[] = $category;
}

$categoryResult->free();
```

### Vì sao cần truy vấn này?

Danh sách danh mục sẽ được dùng để tạo:

``` html
<select>
```

trên giao diện.

Mỗi lựa chọn cần:

``` text
CategoryID   → value
CategoryName → nội dung hiển thị
```

Ví dụ:

``` html
<option value="2">Điện thoại</option>
```

------------------------------------------------------------------------

## 4. Nhận điều kiện tìm kiếm từ URL

Ngay sau phần lấy danh mục, thêm:

``` php
$categoryID = isset($_GET['category'])
    ? (int) $_GET['category']
    : 0;

$keyword = isset($_GET['keyword'])
    ? trim($_GET['keyword'])
    : '';
```

Trong đó:

``` php
$categoryID
```

nhận danh mục được chọn.

Giá trị:

``` text
0
```

được dùng để biểu diễn **Tất cả danh mục**.

Biến:

``` php
$keyword
```

nhận nội dung người dùng nhập vào ô tìm kiếm.

Hàm:

``` php
trim()
```

loại bỏ khoảng trắng thừa ở đầu và cuối chuỗi.

------------------------------------------------------------------------

## 5. Cấu hình phân trang

Tiếp tục thêm:

``` php
$itemsPerPage = 6;

$page = isset($_GET['page'])
    ? (int) $_GET['page']
    : 1;

if ($page < 1) {
    $page = 1;
}

$searchKeyword = '%' . $keyword . '%';
```

Trong bài này:

``` php
$itemsPerPage = 6;
```

nghĩa là mỗi trang hiển thị tối đa 6 sản phẩm.

Biến:

``` php
$searchKeyword
```

được chuẩn bị cho toán tử SQL:

``` sql
LIKE
```

Ví dụ, nếu người dùng nhập:

``` text
phone
```

thì giá trị truyền vào truy vấn là:

``` text
%phone%
```

cho phép tìm chuỗi `phone` ở bất kỳ vị trí nào trong tên hoặc mã sản
phẩm.

------------------------------------------------------------------------

## 6. Đếm tổng số sản phẩm phù hợp

Phân trang cần biết tổng cộng có bao nhiêu sản phẩm thỏa điều kiện.

Thay phần truy vấn danh sách sản phẩm của Hands-on 09 bằng phần xử lý
mới bắt đầu từ truy vấn đếm sau:

``` php
$sqlCount = "
    SELECT
        COUNT(*) AS TotalProducts
    FROM
        products p,
        categories c
    WHERE
        p.CategoryID = c.CategoryID
        AND p.IsActive = 1
";
```

Sau đó bổ sung điều kiện danh mục khi người dùng có chọn:

``` php
if ($categoryID > 0) {
    $sqlCount .= "
        AND p.CategoryID = ?
    ";
}
```

Bổ sung điều kiện tìm kiếm khi từ khóa không rỗng:

``` php
if ($keyword !== '') {
    $sqlCount .= "
        AND (
            p.ProductName LIKE ?
            OR p.ProductCode LIKE ?
        )
    ";
}
```

### 6.1. Gắn tham số cho truy vấn đếm

Thêm:

``` php
$stmtCount = $conn->prepare($sqlCount);
```

Tùy điều kiện người dùng nhập, số lượng và kiểu tham số sẽ khác nhau:

``` php
if ($categoryID > 0 && $keyword !== '') {

    $stmtCount->bind_param(
        'iss',
        $categoryID,
        $searchKeyword,
        $searchKeyword
    );

} elseif ($categoryID > 0) {

    $stmtCount->bind_param(
        'i',
        $categoryID
    );

} elseif ($keyword !== '') {

    $stmtCount->bind_param(
        'ss',
        $searchKeyword,
        $searchKeyword
    );
}
```

Thực thi:

``` php
$stmtCount->execute();

$countResult = $stmtCount->get_result();
$countRow = $countResult->fetch_assoc();

$totalProducts = (int) $countRow['TotalProducts'];

$countResult->free();
$stmtCount->close();
```

!!! note "Một kết quả -- một lần get_result()" Sau `execute()`, lấy kết
quả của statement vào một biến:

    ```php
    $countResult = $stmtCount->get_result();
    ```

    rồi đọc dữ liệu từ chính biến đó. Không cần gọi lại `get_result()` cho cùng một lần thực thi.

------------------------------------------------------------------------

## 7. Tính tổng số trang và vị trí bắt đầu

Sau phần đếm, thêm:

``` php
$totalPages = (int) ceil(
    $totalProducts / $itemsPerPage
);
```

Ví dụ:

``` text
13 sản phẩm
6 sản phẩm/trang
```

thì:

``` text
ceil(13 / 6) = 3 trang
```

### 7.1. Xử lý số trang vượt giới hạn

Thêm:

``` php
if ($totalPages > 0 && $page > $totalPages) {
    $page = $totalPages;
}
```

Nếu URL yêu cầu một trang lớn hơn tổng số trang, ứng dụng sử dụng trang
cuối cùng.

### 7.2. Tính `OFFSET`

Thêm:

``` php
$offset = ($page - 1) * $itemsPerPage;
```

Ví dụ với 6 sản phẩm mỗi trang:

    Trang   OFFSET
  ------- --------
        1        0
        2        6
        3       12

------------------------------------------------------------------------

## 8. Xây dựng truy vấn lấy sản phẩm

Tiếp tục thêm truy vấn:

``` php
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
";
```

Đây vẫn là truy vấn từ Hands-on 09, nhưng chưa kết thúc vì các điều kiện
tìm kiếm sẽ được bổ sung tùy dữ liệu người dùng nhập.

### 8.1. Thêm điều kiện danh mục

``` php
if ($categoryID > 0) {
    $sql .= "
        AND p.CategoryID = ?
    ";
}
```

### 8.2. Thêm điều kiện từ khóa

``` php
if ($keyword !== '') {
    $sql .= "
        AND (
            p.ProductName LIKE ?
            OR p.ProductCode LIKE ?
        )
    ";
}
```

### 8.3. Thêm sắp xếp và phân trang

``` php
$sql .= "
    ORDER BY
        p.ProductID DESC
    LIMIT ?
    OFFSET ?
";
```

Trong đó:

``` sql
LIMIT ?
```

xác định số sản phẩm tối đa trên một trang.

``` sql
OFFSET ?
```

xác định số mẩu tin cần bỏ qua trước khi lấy dữ liệu của trang hiện tại.

------------------------------------------------------------------------

## 9. Gắn tham số cho truy vấn sản phẩm

Chuẩn bị statement:

``` php
$stmt = $conn->prepare($sql);
```

Tùy điều kiện, gắn tham số như sau:

``` php
if ($categoryID > 0 && $keyword !== '') {

    $stmt->bind_param(
        'issii',
        $categoryID,
        $searchKeyword,
        $searchKeyword,
        $itemsPerPage,
        $offset
    );

} elseif ($categoryID > 0) {

    $stmt->bind_param(
        'iii',
        $categoryID,
        $itemsPerPage,
        $offset
    );

} elseif ($keyword !== '') {

    $stmt->bind_param(
        'ssii',
        $searchKeyword,
        $searchKeyword,
        $itemsPerPage,
        $offset
    );

} else {

    $stmt->bind_param(
        'ii',
        $itemsPerPage,
        $offset
    );
}
```

Thực thi:

``` php
$stmt->execute();

$result = $stmt->get_result();
```

Sau đó giữ nguyên phần:

``` php
$pageTitle = 'Sản phẩm';

require_once '/var/www/src/includes/frontend/header.php';
require_once '/var/www/src/includes/frontend/navbar.php';
?>
```

### 9.1. Đọc chuỗi kiểu dữ liệu

Ví dụ:

``` php
'issii'
```

có nghĩa là:

``` text
i → categoryID
s → keyword cho ProductName
s → keyword cho ProductCode
i → LIMIT
i → OFFSET
```

Việc số lượng ký tự trong chuỗi kiểu dữ liệu khớp với số tham số truyền
vào là bắt buộc.

------------------------------------------------------------------------

## 10. Tạo giao diện tìm kiếm và lọc

Trong phần HTML, tìm:

``` html
<div class="mb-4">

    <h1>Sản phẩm</h1>

    <p class="text-muted">
        Khám phá các sản phẩm hiện có tại cửa hàng.
    </p>

</div>
```

Ngay sau khối này, thêm:

``` php
<form
    method="get"
    action="/products.php"
    class="row g-3 mb-4"
>

    <div class="col-md-6 col-lg-4">

        <label
            for="keyword"
            class="form-label"
        >
            Tìm sản phẩm
        </label>

        <input
            type="text"
            name="keyword"
            id="keyword"
            class="form-control"
            value="<?= htmlspecialchars($keyword) ?>"
            placeholder="Nhập tên hoặc mã sản phẩm"
        >

    </div>

    <div class="col-md-6 col-lg-4">

        <label
            for="category"
            class="form-label"
        >
            Danh mục
        </label>

        <select
            name="category"
            id="category"
            class="form-select"
        >

            <option value="0">
                Tất cả danh mục
            </option>

            <?php foreach ($categories as $category): ?>

                <option
                    value="<?= (int) $category['CategoryID'] ?>"
                    <?=
                        $categoryID ===
                        (int) $category['CategoryID']
                            ? 'selected'
                            : ''
                    ?>
                >
                    <?=
                        htmlspecialchars(
                            $category['CategoryName']
                        )
                    ?>
                </option>

            <?php endforeach; ?>

        </select>

    </div>

    <div class="col-md-auto align-self-end">

        <button
            type="submit"
            class="btn btn-primary"
        >
            Tìm kiếm
        </button>

        <a
            href="/products.php"
            class="btn btn-outline-secondary"
        >
            Xóa bộ lọc
        </a>

    </div>

</form>
```

### Vì sao dùng `GET`?

Form sử dụng:

``` html
method="get"
```

nên điều kiện tìm kiếm xuất hiện trên URL.

Ví dụ:

``` text
/products.php?keyword=laptop&category=3
```

Điều này phù hợp với chức năng tìm kiếm vì người dùng có thể:

-   nhìn thấy điều kiện hiện tại;
-   tải lại trang mà vẫn giữ điều kiện;
-   sao chép URL;
-   chuyển trang mà vẫn giữ bộ lọc.

------------------------------------------------------------------------

## 11. Hiển thị số lượng kết quả

Ngay sau form, thêm:

``` php
<p class="text-muted">
    Tìm thấy
    <strong><?= $totalProducts ?></strong>
    sản phẩm.
</p>
```

Biến:

``` php
$totalProducts
```

đến từ truy vấn `COUNT(*)`, vì vậy đây là tổng số sản phẩm thỏa điều
kiện trên **tất cả các trang**, không chỉ số Card đang hiển thị ở trang
hiện tại.

------------------------------------------------------------------------

## 12. Giữ nguyên phần hiển thị Card

Phần danh sách Card đã xây dựng ở Hands-on 09 không cần viết lại.

Tiếp tục sử dụng:

``` php
<?php if ($result->num_rows > 0): ?>
```

và vòng lặp:

``` php
<?php while ($product = $result->fetch_assoc()): ?>
```

Ảnh vẫn sử dụng:

``` text
/uploads/products/
```

!!! note "Không thay đổi đường dẫn ảnh" Tìm kiếm và phân trang chỉ thay
đổi cách truy vấn dữ liệu. Cấu trúc lưu hình ảnh không thay đổi.

------------------------------------------------------------------------

## 13. Thêm phân trang

Tìm vị trí ngay sau phần kết thúc danh sách sản phẩm và trước:

``` html
</main>
```

Thêm:

``` php
<?php if ($totalPages > 1): ?>

    <nav
        class="mt-5"
        aria-label="Phân trang sản phẩm"
    >

        <ul
            class="pagination
                   justify-content-center
                   flex-wrap"
        >
```

### 13.1. Tạo nút Trước

Tiếp tục:

``` php
<?php
$previousQuery = http_build_query([
    'keyword' => $keyword,
    'category' => $categoryID,
    'page' => max(1, $page - 1)
]);
?>

<li
    class="page-item <?=
        $page <= 1
            ? 'disabled'
            : ''
    ?>"
>
    <a
        class="page-link"
        href="/products.php?<?= $previousQuery ?>"
    >
        &laquo; Trước
    </a>
</li>
```

Hàm:

``` php
http_build_query()
```

tạo query string từ các giá trị hiện tại.

Ví dụ:

``` text
keyword=laptop&category=2&page=1
```

Nhờ đó, khi nhấn **Trước**, điều kiện tìm kiếm và danh mục không bị mất.

------------------------------------------------------------------------

## 14. Tạo các số trang

Ngay sau nút Trước, thêm:

``` php
<?php for (
    $pageNumber = 1;
    $pageNumber <= $totalPages;
    $pageNumber++
): ?>

    <?php
    $query = http_build_query([
        'keyword' => $keyword,
        'category' => $categoryID,
        'page' => $pageNumber
    ]);
    ?>

    <li
        class="page-item <?=
            $pageNumber === $page
                ? 'active'
                : ''
        ?>"
    >
        <a
            class="page-link"
            href="/products.php?<?= $query ?>"
        >
            <?= $pageNumber ?>
        </a>
    </li>

<?php endfor; ?>
```

Trang hiện tại nhận lớp Bootstrap:

``` text
active
```

để người dùng nhận biết đang ở trang nào.

------------------------------------------------------------------------

## 15. Tạo nút Sau

Tiếp tục:

``` php
<?php
$nextQuery = http_build_query([
    'keyword' => $keyword,
    'category' => $categoryID,
    'page' => min(
        $totalPages,
        $page + 1
    )
]);
?>

<li
    class="page-item <?=
        $page >= $totalPages
            ? 'disabled'
            : ''
    ?>"
>
    <a
        class="page-link"
        href="/products.php?<?= $nextQuery ?>"
    >
        Sau &raquo;
    </a>
</li>
```

Đóng các thẻ phân trang:

``` php
        </ul>

    </nav>

<?php endif; ?>
```

Như vậy phân trang chỉ xuất hiện khi:

``` php
$totalPages > 1
```

------------------------------------------------------------------------

## 16. Đóng kết quả truy vấn đúng vị trí

Ở cuối `products.php`, phần sau phải được thực hiện **sau khi HTML đã sử
dụng xong `$result`**:

``` php
<?php

$result->free();
$stmt->close();

require_once '/var/www/src/includes/frontend/footer.php';
```

`$result` chứa các sản phẩm của trang hiện tại, còn `$stmt` là prepared
statement đã dùng để lấy danh sách.

------------------------------------------------------------------------

## 17. Kiểm tra cú pháp

Lưu tập tin bằng:

``` text
Ctrl + S
```

Sau đó chạy:

``` cmd
docker compose exec web php -l /var/www/html/products.php
```

Kết quả đúng:

``` text
No syntax errors detected in /var/www/html/products.php
```

!!! success "Checkpoint" Chỉ chuyển sang kiểm thử chức năng khi kiểm tra
cú pháp đã đạt.

------------------------------------------------------------------------

## 18. Kiểm thử tìm kiếm

Mở:

``` text
http://localhost:8080/products.php
```

### 18.1. Tìm theo tên sản phẩm

Nhập một phần tên sản phẩm, ví dụ:

``` text
phone
```

Nhấn **Tìm kiếm**.

Kiểm tra:

-   URL có `keyword=...`;
-   chỉ các sản phẩm phù hợp xuất hiện;
-   ô tìm kiếm vẫn giữ từ khóa sau khi trang tải lại;
-   dòng `Tìm thấy ... sản phẩm` đúng với kết quả.

### 18.2. Tìm theo mã sản phẩm

Nhập một phần mã sản phẩm đã có trong dữ liệu.

Kiểm tra sản phẩm phù hợp vẫn được tìm thấy.

### 18.3. Từ khóa không có kết quả

Nhập một chuỗi không tồn tại.

Kết quả phải hiển thị:

``` text
Tìm thấy 0 sản phẩm.
```

và thông báo:

``` text
Hiện chưa có sản phẩm nào để hiển thị.
```

------------------------------------------------------------------------

## 19. Kiểm thử lọc theo danh mục

Xóa bộ lọc bằng nút:

``` text
Xóa bộ lọc
```

Chọn một danh mục và nhấn **Tìm kiếm**.

Kiểm tra:

-   URL có `category=...`;
-   danh mục vừa chọn vẫn được giữ trong `<select>`;
-   tất cả sản phẩm hiển thị thuộc danh mục đó;
-   số lượng kết quả được cập nhật.

Chọn:

``` text
Tất cả danh mục
```

để quay lại toàn bộ sản phẩm đang hoạt động.

------------------------------------------------------------------------

## 20. Kiểm thử kết hợp tìm kiếm và lọc

Chọn một danh mục và đồng thời nhập từ khóa.

Ví dụ URL có thể có dạng:

``` text
/products.php?keyword=phone&category=2
```

Kết quả phải thỏa **đồng thời**:

``` text
đúng danh mục
AND
tên hoặc mã chứa từ khóa
```

Điều này tương ứng với điều kiện SQL:

``` sql
AND p.CategoryID = ?
AND (
    p.ProductName LIKE ?
    OR p.ProductCode LIKE ?
)
```

------------------------------------------------------------------------

## 21. Kiểm thử phân trang

Phân trang chỉ xuất hiện khi số kết quả lớn hơn:

``` php
$itemsPerPage
```

Nếu dữ liệu hiện tại chưa đủ để tạo nhiều trang, **chỉ trong lúc kiểm
thử**, đổi:

``` php
$itemsPerPage = 6;
```

thành:

``` php
$itemsPerPage = 3;
```

Lưu và tải lại trang.

Kiểm tra:

-   xuất hiện nhiều số trang;
-   trang hiện tại được đánh dấu;
-   nhấn số trang chuyển đúng dữ liệu;
-   nút **Trước** bị vô hiệu hóa ở trang đầu;
-   nút **Sau** bị vô hiệu hóa ở trang cuối;
-   tìm kiếm và danh mục vẫn được giữ khi chuyển trang.

Sau khi kiểm thử xong, đổi lại:

``` php
$itemsPerPage = 6;
```

!!! warning "Không commit cấu hình kiểm thử" Nếu bạn tạm đổi số sản phẩm
mỗi trang để kiểm thử, hãy trả về giá trị chính thức trước khi commit.

------------------------------------------------------------------------

## 22. Kiểm thử số trang không hợp lệ

### 22.1. Trang nhỏ hơn 1

Thử:

``` text
/products.php?page=0
```

hoặc:

``` text
/products.php?page=-5
```

Ứng dụng phải xử lý như trang 1.

### 22.2. Trang lớn hơn tổng số trang

Thử một giá trị lớn:

``` text
/products.php?page=999
```

Nếu có sản phẩm, ứng dụng phải sử dụng trang cuối cùng thay vì tạo
`OFFSET` vượt khỏi dữ liệu.

------------------------------------------------------------------------

## 23. Kiểm tra responsive

Nhấn:

``` text
F12
```

sau đó:

``` text
Ctrl + Shift + M
```

Thử trên màn hình nhỏ.

Kiểm tra:

-   ô tìm kiếm và danh mục tự xếp phù hợp;
-   các nút không làm vỡ bố cục;
-   Card sản phẩm vẫn responsive;
-   phân trang có thể xuống dòng nhờ `flex-wrap`;
-   ảnh sản phẩm vẫn giữ tỷ lệ và chiều cao cân đối.

------------------------------------------------------------------------

## 24. Kiểm tra toàn bộ Hands-on 10

Đánh dấu sau khi hoàn thành:

  Nội dung             Kết quả mong đợi
  -------------------- ------------------------------------------
  Danh sách danh mục   Được lấy từ bảng `categories`
  Tìm theo tên         Hoạt động
  Tìm theo mã          Hoạt động
  Lọc danh mục         Hoạt động
  Tìm + lọc            Hoạt động đồng thời
  `COUNT(*)`           Đếm đúng tổng số kết quả
  `LIMIT`              Giới hạn đúng số sản phẩm mỗi trang
  `OFFSET`             Lấy đúng dữ liệu theo trang
  Số trang             Hiển thị đúng
  Trước/Sau            Hoạt động đúng ở trang đầu/cuối
  Query string         Giữ keyword và category khi chuyển trang
  `page < 1`           Xử lý như trang 1
  `page` quá lớn       Chuyển về giới hạn trang cuối
  Không có kết quả     Hiển thị thông báo phù hợp
  Responsive           Form, Card và pagination hiển thị tốt

------------------------------------------------------------------------

## 25. Luồng xử lý sau Hands-on 10

Trang sản phẩm hiện xử lý theo luồng:

``` text
Người dùng nhập điều kiện
        ↓
GET: keyword, category, page
        ↓
Đọc danh sách Category
        ↓
COUNT(*) theo điều kiện
        ↓
Tính totalPages và OFFSET
        ↓
SELECT sản phẩm
        ↓
LIMIT + OFFSET
        ↓
Hiển thị Card
        ↓
Tạo liên kết phân trang
        ↓
Giữ lại keyword + category
```

Điểm quan trọng là **truy vấn đếm và truy vấn lấy dữ liệu phải sử dụng
cùng điều kiện tìm kiếm**. Nếu hai truy vấn dùng điều kiện khác nhau, số
trang sẽ không khớp với dữ liệu thực tế.

------------------------------------------------------------------------

## 26. Lưu phiên bản bằng Git

Trước tiên kiểm tra:

``` cmd
git status
```

Kiểm tra lỗi khoảng trắng:

``` cmd
git diff --check
```

Nếu mọi thứ đúng, thêm thay đổi:

``` cmd
git add public\products.php
```

Nếu trong quá trình thực hành bạn chỉ thay `products.php`, không cần
`git add .`.

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
git commit -m "feat: add product search filtering and pagination"
```

Đẩy lên GitHub:

``` cmd
git push
```

------------------------------------------------------------------------

## 27. Kết quả đạt được

Sau Hands-on 10, Frontend sản phẩm đã phát triển từ danh sách cơ bản
thành trang có khả năng khai thác dữ liệu thực tế hơn:

``` text
Sản phẩm
├── Tìm theo tên hoặc mã
├── Lọc theo danh mục
├── Kết hợp tìm kiếm + lọc
├── Hiển thị tổng số kết quả
└── Phân trang
    ├── Trước
    ├── Số trang
    └── Sau
```

Các chức năng này vẫn sử dụng trang chi tiết sản phẩm đã xây dựng ở
Hands-on 09.

Ứng dụng hiện đã có một Frontend đủ nền tảng để tiếp tục phát triển
luồng mua hàng.

------------------------------------------------------------------------

## Câu hỏi củng cố

1.  Vì sao form tìm kiếm sử dụng phương thức `GET` thay vì `POST`?
2.  Hai ký tự `%` trong giá trị `%phone%` có ý nghĩa gì khi dùng với
    `LIKE`?
3.  Vì sao cần truy vấn `COUNT(*)` trước khi phân trang?
4.  `LIMIT` và `OFFSET` có vai trò gì?
5.  Với 6 sản phẩm mỗi trang, `OFFSET` của trang 4 bằng bao nhiêu?
6.  Vì sao truy vấn đếm và truy vấn lấy danh sách phải sử dụng cùng điều
    kiện?
7.  Vì sao cần giữ `keyword` và `category` khi tạo liên kết sang trang
    khác?
8.  `http_build_query()` giúp ích gì khi xây dựng URL phân trang?
9.  Vì sao cần kiểm tra `$page < 1` và `$page > $totalPages`?
10. Vì sao số lượng tham số trong `bind_param()` phải khớp với các
    placeholder `?` trong câu SQL?

## Bài tập củng cố

Thực hiện các trường hợp kiểm thử sau trên dữ liệu của bạn:

1.  Tìm một sản phẩm bằng một phần tên.
2.  Tìm sản phẩm bằng một phần mã.
3.  Lọc một danh mục có nhiều sản phẩm.
4.  Kết hợp từ khóa với danh mục.
5.  Tìm một từ khóa không có kết quả.
6.  Chuyển qua nhiều trang và kiểm tra bộ lọc có được giữ lại hay không.
7.  Thử `page=0`, `page=-1` và một số trang lớn hơn tổng số trang.
8.  Kiểm tra giao diện trên màn hình desktop và thiết bị di động.
