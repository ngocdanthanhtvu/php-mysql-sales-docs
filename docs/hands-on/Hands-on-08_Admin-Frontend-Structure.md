# Hands-on 08 -- Tổ chức khu vực quản trị và giao diện người dùng

## Mục tiêu

Sau khi hoàn thành Hands-on này, bạn có thể:

-   Phân biệt khu vực **quản trị (Admin)** và **giao diện người dùng
    (Frontend)** trong ứng dụng web.
-   Tổ chức các chức năng quản trị vào `public/admin/`.
-   Tách các thành phần giao diện dùng chung của Admin và Frontend.
-   Cập nhật đúng URL và đường dẫn `require_once` sau khi thay đổi cấu
    trúc thư mục.
-   Phân biệt **URL chức năng** với **đường dẫn tài nguyên**, đặc biệt
    là hình ảnh sản phẩm.
-   Kiểm tra lại ứng dụng sau khi tái tổ chức cấu trúc.

------------------------------------------------------------------------

## 1. Chuẩn bị

Hands-on này tiếp tục từ kết quả của **Hands-on 07**.

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

Ở thời điểm này, cấu trúc chính của dự án có dạng:

``` text
public/
├── index.php
├── categories/
├── products/
├── shippers/
└── uploads/
    └── products/

src/
├── config/
│   └── database.php
└── includes/
    ├── header.php
    ├── navbar.php
    └── footer.php
```

!!! note "Lưu ý" Cấu trúc của bạn có thể khác đôi chút tùy các chức năng
đã hoàn thành ở những Hands-on trước.

    Ví dụ, `shippers` có thể chỉ có `index.php`, hoặc đã có thêm `create.php`, `edit.php`, `delete.php`.

    Trong Hands-on này, hãy thao tác trên **toàn bộ các tập tin chức năng hiện có** của từng đối tượng. Không cần tạo thêm tập tin chỉ để giống cấu trúc minh họa.

------------------------------------------------------------------------

## 2. Vì sao cần tách Admin và Frontend?

Đến Hands-on 07, các trang quản lý danh mục, sản phẩm và người giao hàng
đều nằm trực tiếp trong `public/`.

Khi ứng dụng tiếp tục phát triển, cần phân biệt hai khu vực:

``` text
Ứng dụng
│
├── Frontend
│   └── Giao diện dành cho người dùng
│
└── Admin
    └── Giao diện và chức năng quản trị
```

Các chức năng CRUD đã xây dựng thuộc khu vực Admin:

-   quản lý danh mục;
-   quản lý sản phẩm;
-   quản lý hình ảnh sản phẩm;
-   quản lý người giao hàng;
-   các chức năng quản trị khác đã xây dựng.

Trong khi đó:

``` text
public/uploads/products/
```

là nơi lưu hình ảnh sản phẩm. Đây là **tài nguyên dùng chung**, không
phải một chức năng quản trị.

Vì vậy, sau khi tổ chức lại:

``` text
/admin/products/
```

là URL chức năng quản trị sản phẩm, còn:

``` text
/uploads/products/
```

vẫn là URL dùng để truy cập hình ảnh.

------------------------------------------------------------------------

## 3. Tạo khu vực Admin

### 3.1. Tạo thư mục `admin`

Tại thư mục gốc của dự án, chạy:

``` cmd
mkdir public\admin
```

Kiểm tra:

``` cmd
tree public /F
```

Bạn phải thấy thư mục:

``` text
public
└── admin
```

### 3.2. Di chuyển các chức năng quản trị

Di chuyển các thư mục đã xây dựng:

``` cmd
move public\categories public\admin\categories
move public\products public\admin\products
move public\shippers public\admin\shippers
```

Nếu dự án của bạn có thêm đối tượng quản trị khác, thực hiện tương tự
với đối tượng đó.

!!! warning "Không di chuyển thư mục hình ảnh" Giữ nguyên:

    ```text
    public/uploads/products/
    ```

    Không chuyển thư mục này vào `public/admin/`.

Kiểm tra lại:

``` cmd
tree public /F
```

Cấu trúc lúc này phải có dạng:

``` text
public/
├── index.php
├── admin/
│   ├── categories/
│   ├── products/
│   └── shippers/
└── uploads/
    └── products/
```

------------------------------------------------------------------------

## 4. Tổ chức các thành phần giao diện của Admin

Các trang quản trị hiện đang dùng chung:

``` text
src/includes/header.php
src/includes/navbar.php
src/includes/footer.php
```

Ta chuyển ba tập tin này thành layout riêng của Admin.

### 4.1. Tạo thư mục `includes/admin`

Chạy:

``` cmd
mkdir src\includes\admin
```

### 4.2. Di chuyển các tập tin giao diện

Chạy:

``` cmd
move src\includes\header.php src\includes\admin\header.php
move src\includes\navbar.php src\includes\admin\navbar.php
move src\includes\footer.php src\includes\admin\footer.php
```

Kiểm tra:

``` cmd
tree src\includes /F
```

Kết quả cần có:

``` text
src/includes/
└── admin/
    ├── header.php
    ├── navbar.php
    └── footer.php
```

------------------------------------------------------------------------

## 5. Cập nhật `require_once` của các trang Admin

Sau khi di chuyển layout, các trang quản trị vẫn tham chiếu đến đường
dẫn cũ.

Ta cần đổi:

``` php
/var/www/src/includes/header.php
```

thành:

``` php
/var/www/src/includes/admin/header.php
```

Tương tự với `navbar.php` và `footer.php`.

### 5.1. Thay đường dẫn `header.php`

Trong VS Code:

1.  Nhấn **Ctrl + Shift + H** để mở Search/Replace trên toàn dự án.
2.  Tại **Files to include**, nhập:

``` text
public/admin/**/*.php
```

3.  Tại ô **Search**, nhập:

``` text
/var/www/src/includes/header.php
```

4.  Tại ô **Replace**, nhập:

``` text
/var/www/src/includes/admin/header.php
```

5.  Quan sát danh sách kết quả để chắc chắn phạm vi thay thế chỉ nằm
    trong các trang Admin.
6.  Chọn **Replace All**.
7.  Nhấn **Ctrl + S** nếu VS Code chưa tự lưu thay đổi.

Thực hiện tương tự với:

``` text
/var/www/src/includes/navbar.php
```

thành:

``` text
/var/www/src/includes/admin/navbar.php
```

và:

``` text
/var/www/src/includes/footer.php
```

thành:

``` text
/var/www/src/includes/admin/footer.php
```

!!! tip "Nguyên tắc khi dùng Replace All" Luôn xem danh sách kết quả
trước khi thay toàn bộ. Chỉ dùng Replace All khi chuỗi tìm kiếm đủ cụ
thể và các kết quả đều là những vị trí cần sửa.

### 5.2. Kiểm tra

Chạy:

``` cmd
findstr /s /n /i /c:"/var/www/src/includes/" public\*.php
```

Các trang trong `public/admin/` phải sử dụng:

``` text
/var/www/src/includes/admin/
```

------------------------------------------------------------------------

## 6. Cập nhật URL của Categories

Sau khi chuyển thư mục:

``` text
public/categories/
```

sang:

``` text
public/admin/categories/
```

URL chức năng cũng phải thay đổi.

Ví dụ:

``` text
/categories/
```

thành:

``` text
/admin/categories/
```

### 6.1. Thực hiện thay thế

Trong VS Code, nhấn **Ctrl + Shift + H**.

Tại **Files to include**, nhập:

``` text
public/admin/categories/*.php
```

Search:

``` text
/categories/
```

Replace:

``` text
/admin/categories/
```

Kiểm tra các kết quả tìm được, sau đó chọn **Replace All**.

Các vị trí được cập nhật có thể gồm:

``` php
header('Location: /admin/categories/');
```

``` html
<a href="/admin/categories/">
```

``` html
<a href="/admin/categories/edit.php?id=...">
```

``` html
<form action="/admin/categories/delete.php">
```

### 6.2. Kiểm tra

Chạy:

``` cmd
findstr /s /n /i /c:"/admin/categories/" public\admin\categories\*.php
```

Các URL chức năng Category phải bắt đầu bằng:

``` text
/admin/categories/
```

------------------------------------------------------------------------

## 7. Cập nhật URL của Products

Products cần được xử lý khác Categories vì trong mã nguồn có cả:

-   URL chức năng;
-   URL hình ảnh;
-   đường dẫn vật lý đến thư mục hình ảnh.

### 7.1. Kiểm tra các vị trí có `products/`

Chạy:

``` cmd
findstr /s /n /i /c:"products/" public\admin\products\*.php
```

Bạn có thể thấy các dạng như:

``` text
/products/
/products/create.php
/products/edit.php
/products/delete.php
/uploads/products/
/var/www/html/uploads/products/
```

Phân loại như sau:

  Loại              Trước khi chuyển                    Sau khi chuyển
  ----------------- ----------------------------------- ------------------------------
  URL danh sách     `/products/`                        `/admin/products/`
  URL thêm          `/products/create.php`              `/admin/products/create.php`
  URL sửa           `/products/edit.php`                `/admin/products/edit.php`
  URL xóa           `/products/delete.php`              `/admin/products/delete.php`
  URL hình ảnh      `/uploads/products/`                **Giữ nguyên**
  Thư mục lưu ảnh   `/var/www/html/uploads/products/`   **Giữ nguyên**

!!! warning "Không Replace All `/products/`" Không thay toàn bộ:

    ```text
    /products/
    ```

    thành:

    ```text
    /admin/products/
    ```

    vì thao tác này có thể làm thay đổi cả đường dẫn:

    ```text
    /uploads/products/
    ```

    và:

    ```text
    /var/www/html/uploads/products/
    ```

### 7.2. Cập nhật URL chức năng

Trong VS Code, mở các kết quả tìm kiếm trong:

``` text
public/admin/products/*.php
```

Chỉ cập nhật các URL chức năng để đạt các dạng:

``` php
header('Location: /admin/products/');
```

``` html
href="/admin/products/"
```

``` html
href="/admin/products/create.php"
```

``` html
href="/admin/products/edit.php?id=..."
```

``` html
action="/admin/products/delete.php"
```

Nếu trang chỉnh sửa sản phẩm có các nút sử dụng `formaction`, URL cũng
phải bắt đầu bằng:

``` text
/admin/products/edit.php
```

### 7.3. Giữ nguyên đường dẫn hình ảnh

Các đường dẫn sau **không thay đổi**:

``` text
/uploads/products/
```

``` text
/var/www/html/uploads/products/
```

### 7.4. Kiểm tra

Kiểm tra URL Admin:

``` cmd
findstr /s /n /i /c:"admin/products/" public\admin\products\*.php
```

Kiểm tra đường dẫn hình ảnh:

``` cmd
findstr /s /n /i /c:"uploads/products/" public\admin\products\*.php
```

Kết quả thứ hai vẫn phải chứa các đường dẫn dạng:

``` text
/uploads/products/
```

hoặc:

``` text
/var/www/html/uploads/products/
```

------------------------------------------------------------------------

## 8. Cập nhật Shippers và các đối tượng quản trị khác

Với Shippers, URL chức năng:

``` text
/shippers/
```

được đổi thành:

``` text
/admin/shippers/
```

Trong VS Code, giới hạn phạm vi:

``` text
public/admin/shippers/*.php
```

Search:

``` text
/shippers/
```

Replace:

``` text
/admin/shippers/
```

Kiểm tra kết quả rồi chọn **Replace All**.

!!! note "Tùy theo bài làm hiện tại" Nếu `shippers` của bạn đã có
`create.php`, `edit.php`, `delete.php`, hãy cập nhật URL trong tất cả
các tập tin đó.

    Nếu hiện tại chỉ có `index.php`, chỉ cần cập nhật những URL thực sự có trong `index.php`.

Với các đối tượng quản trị khác đã xây dựng, thực hiện theo cùng nguyên
tắc.

------------------------------------------------------------------------

## 9. Cập nhật thanh điều hướng Admin

Mở:

``` cmd
code src\includes\admin\navbar.php
```

Các liên kết đến chức năng quản trị cần sử dụng URL mới.

Ví dụ:

``` html
<a class="navbar-brand" href="/admin/">
    Sales Management
</a>
```

Các mục quản trị:

``` text
/admin/categories/
/admin/products/
/admin/shippers/
```

Ví dụ:

``` html
<li class="nav-item">
    <a class="nav-link" href="/admin/categories/">
        Danh mục
    </a>
</li>

<li class="nav-item">
    <a class="nav-link" href="/admin/products/">
        Sản phẩm
    </a>
</li>

<li class="nav-item">
    <a class="nav-link" href="/admin/shippers/">
        Người giao hàng
    </a>
</li>
```

Nếu navbar của bạn có thêm các chức năng đã xây dựng, cập nhật URL tương
ứng sang `/admin/...`.

Nhấn **Ctrl + S** để lưu.

------------------------------------------------------------------------

## 10. Tạo trang chủ Admin

Tạo tập tin:

``` cmd
type nul > public\admin\index.php
```

Mở:

``` cmd
code public\admin\index.php
```

Nhập:

``` php
<?php
$pageTitle = 'Quản trị hệ thống';

require_once '/var/www/src/includes/admin/header.php';
require_once '/var/www/src/includes/admin/navbar.php';
?>

<main class="container py-5">

    <h1>Quản trị hệ thống</h1>

    <p class="text-muted">
        Chọn chức năng quản lý từ thanh điều hướng.
    </p>

</main>

<?php
require_once '/var/www/src/includes/admin/footer.php';
```

Lưu bằng **Ctrl + S**.

Kiểm tra trên trình duyệt:

``` text
http://localhost:8080/admin/
```

Trang phải sử dụng header, navbar và footer của Admin.

------------------------------------------------------------------------

## 11. Kiểm tra khu vực Admin trước khi tạo Frontend

Không chuyển sang bước tiếp theo cho đến khi các chức năng Admin đang
hoạt động.

Kiểm tra lần lượt:

``` text
http://localhost:8080/admin/
http://localhost:8080/admin/categories/
http://localhost:8080/admin/products/
http://localhost:8080/admin/shippers/
```

Với Categories, kiểm tra các chức năng đã xây dựng:

-   xem danh sách;
-   thêm;
-   sửa;
-   xóa.

Với Products, kiểm tra:

-   xem danh sách;
-   thêm sản phẩm;
-   sửa sản phẩm;
-   xóa sản phẩm;
-   hiển thị hình ảnh;
-   thêm hình ảnh;
-   xóa hình ảnh;
-   đặt ảnh chính.

!!! success "Checkpoint" Chỉ tiếp tục khi các chức năng đã có trước đó
vẫn hoạt động sau khi chuyển sang `/admin/`.

------------------------------------------------------------------------

## 12. Tạo layout cho Frontend

Admin đã có layout riêng. Bây giờ tạo layout dành cho người dùng.

### 12.1. Tạo thư mục

Chạy:

``` cmd
mkdir src\includes\frontend
```

### 12.2. Tạo ba tập tin

Chạy:

``` cmd
type nul > src\includes\frontend\header.php
type nul > src\includes\frontend\navbar.php
type nul > src\includes\frontend\footer.php
```

Kiểm tra:

``` cmd
tree src\includes /F
```

Cấu trúc cần có:

``` text
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

------------------------------------------------------------------------

## 13. Xây dựng `frontend/header.php`

Mở:

``` cmd
code src\includes\frontend\header.php
```

Nhập:

``` php
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>
        <?= htmlspecialchars($pageTitle ?? 'Sales Management') ?>
    </title>

    <link
        href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.8/dist/css/bootstrap.min.css"
        rel="stylesheet"
    >

    <script
        src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.8/dist/js/bootstrap.bundle.min.js">
    </script>
</head>

<body>
```

Lưu bằng **Ctrl + S**.

------------------------------------------------------------------------

## 14. Xây dựng `frontend/navbar.php`

Mở:

``` cmd
code src\includes\frontend\navbar.php
```

Nhập:

``` php
<nav class="navbar navbar-expand-lg bg-dark navbar-dark">

    <div class="container">

        <a class="navbar-brand" href="/">
            Sales Management
        </a>

        <button
            class="navbar-toggler"
            type="button"
            data-bs-toggle="collapse"
            data-bs-target="#frontendNavbar"
        >
            <span class="navbar-toggler-icon"></span>
        </button>

        <div
            class="collapse navbar-collapse"
            id="frontendNavbar"
        >

            <ul class="navbar-nav">

                <li class="nav-item">
                    <a class="nav-link" href="/">
                        Trang chủ
                    </a>
                </li>

            </ul>

        </div>

    </div>

</nav>
```

Lưu bằng **Ctrl + S**.

!!! note "Vì sao chưa có mục Sản phẩm?" Ở Hands-on 08, Frontend mới được
tổ chức về cấu trúc. Trang danh sách sản phẩm dành cho người dùng sẽ
được xây dựng ở Hands-on tiếp theo.

    Vì vậy, chưa tạo liên kết đến một trang chưa tồn tại.

------------------------------------------------------------------------

## 15. Xây dựng `frontend/footer.php`

Mở:

``` cmd
code src\includes\frontend\footer.php
```

Nhập:

``` php
<footer class="mt-5 p-4 bg-dark text-white text-center">

    <p class="mb-0">
        Hệ thống quản lý bán hàng
    </p>

</footer>

</body>
</html>
```

Lưu bằng **Ctrl + S**.

------------------------------------------------------------------------

## 16. Cập nhật trang chủ Frontend

Mở tập tin:

``` cmd
code public\index.php
```

Thay nội dung hiện tại bằng:

``` php
<?php
$pageTitle = 'Trang chủ';

require_once '/var/www/src/includes/frontend/header.php';
require_once '/var/www/src/includes/frontend/navbar.php';
?>

<main class="container py-5">

    <h1>Hệ thống quản lý bán hàng</h1>

    <p class="text-muted">
        Ứng dụng PHP và MySQL đang hoạt động.
    </p>

</main>

<?php
require_once '/var/www/src/includes/frontend/footer.php';
```

Lưu bằng **Ctrl + S**.

Bây giờ:

``` text
/
```

sử dụng layout Frontend, còn:

``` text
/admin/
```

sử dụng layout Admin.

------------------------------------------------------------------------

## 17. Kiểm tra cú pháp PHP

Dự án đang chạy PHP trong Docker, vì vậy thực hiện kiểm tra cú pháp bằng
PHP trong container.

Kiểm tra trang chủ:

``` cmd
docker compose exec web php -l /var/www/html/index.php
```

Kiểm tra Admin:

``` cmd
docker compose exec web php -l /var/www/html/admin/index.php
```

Kiểm tra các trang chính:

``` cmd
docker compose exec web php -l /var/www/html/admin/categories/index.php
docker compose exec web php -l /var/www/html/admin/products/index.php
docker compose exec web php -l /var/www/html/admin/shippers/index.php
```

Kết quả đúng có dạng:

``` text
No syntax errors detected in ...
```

Nếu bạn có các tập tin `create.php`, `edit.php`, `delete.php`, cũng kiểm
tra cú pháp các tập tin đã thay đổi trước khi kiểm thử trên trình duyệt.

------------------------------------------------------------------------

## 18. Kiểm thử toàn bộ ứng dụng

Thực hiện lần lượt và đánh dấu kết quả.

  Nội dung kiểm tra      Kết quả mong đợi
  ---------------------- -------------------------------------------------
  `/`                    Trang chủ Frontend hiển thị đúng
  `/admin/`              Trang chủ Admin hiển thị đúng
  `/admin/categories/`   Danh sách Category hoạt động
  CRUD Category          Các chức năng đã xây dựng vẫn hoạt động
  `/admin/products/`     Danh sách Product và hình ảnh hiển thị đúng
  CRUD Product           Các chức năng đã xây dựng vẫn hoạt động
  Quản lý ảnh Product    Thêm, xóa, đặt ảnh chính hoạt động
  `/admin/shippers/`     Các chức năng Shipper đã xây dựng vẫn hoạt động

Đặc biệt, hình ảnh sản phẩm phải tiếp tục được truy cập từ:

``` text
/uploads/products/
```

không phải:

``` text
/admin/uploads/products/
```

hay:

``` text
/uploads/admin/products/
```

------------------------------------------------------------------------

## 19. Kiểm tra cấu trúc cuối cùng

Chạy:

``` cmd
tree public /F
```

và:

``` cmd
tree src\includes /F
```

Cấu trúc chính sau Hands-on 08:

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

------------------------------------------------------------------------

## 20. Lưu phiên bản bằng Git

Sau khi toàn bộ kiểm thử đã đạt, kiểm tra:

``` cmd
git status
```

Thêm các thay đổi:

``` cmd
git add .
```

Kiểm tra:

``` cmd
git diff --cached --check
```

Nếu không có thông báo lỗi, xem tóm tắt:

``` cmd
git diff --cached --stat
```

Commit:

``` cmd
git commit -m "refactor: separate admin and frontend areas"
```

Đẩy lên GitHub:

``` cmd
git push
```

------------------------------------------------------------------------

## 21. Kết quả đạt được

Sau Hands-on 08, ứng dụng đã được tổ chức thành hai khu vực:

``` text
Frontend
/
└── Giao diện dành cho người dùng

Admin
/admin/
├── categories/
├── products/
└── shippers/

Tài nguyên dùng chung
/uploads/products/
└── Hình ảnh sản phẩm
```

Việc tổ chức này tạo nền tảng để tiếp tục xây dựng các chức năng dành
cho người dùng mà không làm lẫn giao diện và URL với khu vực quản trị.

------------------------------------------------------------------------

## Câu hỏi củng cố

1.  Khu vực Admin và Frontend khác nhau về mục đích sử dụng như thế nào?
2.  Vì sao `public/uploads/products/` không được chuyển vào
    `public/admin/`?
3.  Phân biệt `/admin/products/` và `/uploads/products/`.
4.  Vì sao Admin và Frontend nên có các tập tin `navbar.php` riêng?
5.  Sau khi di chuyển một trang PHP sang thư mục khác, những loại đường
    dẫn nào cần được kiểm tra lại?
6.  Vì sao cần kiểm thử lại chức năng quản lý hình ảnh sau khi chuyển
    Products vào khu vực Admin?

## Bài tập củng cố

Rà soát dự án của bạn. Nếu đã xây dựng thêm các đối tượng quản trị ngoài
Categories, Products và Shippers:

1.  Chuyển các chức năng đó vào `public/admin/`.
2.  Cập nhật URL chức năng sang `/admin/...`.
3.  Sử dụng layout Admin.
4.  Giữ nguyên các đường dẫn tài nguyên dùng chung nếu chúng không thuộc
    riêng khu vực Admin.
5.  Kiểm tra cú pháp và kiểm thử lại chức năng sau khi hoàn thành.
