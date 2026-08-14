# Hands-on 03 – Kết nối PHP với MySQL và hiển thị danh mục

## 1. Mục tiêu

Sau khi hoàn thành Hands-on này, người học có thể:

- giải thích vai trò của tầng PHP khi làm việc với MySQL;
- tổ chức mã nguồn PHP thành phần công khai (`public`) và mã dùng nội bộ (`src`);
- truyền cấu hình cơ sở dữ liệu từ `.env` vào container Web;
- kết nối PHP với MySQL bằng **MySQLi**;
- giải thích vì sao PHP trong container `web` kết nối MySQL bằng host `db` thay vì `localhost`;
- thiết lập `utf8mb4` cho kết nối để xử lý đúng tiếng Việt;
- tách các thành phần giao diện dùng chung thành `header.php`, `navbar.php`, `footer.php`;
- truy vấn bảng `categories` và hiển thị dữ liệu thật bằng Bootstrap 5.

> **Phạm vi Hands-on 03:** kết nối cơ sở dữ liệu, tổ chức layout dùng chung, thực hiện `SELECT` và hiển thị danh sách danh mục. Các chức năng **Thêm – Sửa – Xóa** sẽ được thực hiện ở Hands-on tiếp theo.

---

## 2. Điều kiện trước khi bắt đầu

Hands-on 02 phải hoàn thành và project có cấu trúc cơ bản:

```text
php-mysql-sales/
├── database/
│   ├── schema.sql
│   └── seed.sql
├── docker/
│   └── php/
│       └── Dockerfile
├── public/
│   └── index.php
├── src/
│   ├── config/
│   └── includes/
├── .env
├── .env.example
├── .gitignore
├── compose.yaml
└── README.md
```

Cơ sở dữ liệu `ql_banhang` đã được khởi tạo và có dữ liệu mẫu trong bảng `categories`.

---

## 3. Kiểm tra trạng thái ban đầu

### Lệnh thực thi

Mở CMD tại project:

```cmd
cd /d D:\PTUDW-ST-2026\php-mysql-sales

git status
docker compose up -d
docker compose ps
```

Mở trình duyệt:

```text
http://localhost:8080
```

### Kết quả mong đợi

- Git không có thay đổi chưa xử lý từ Hands-on trước.
- Container `web` và `db` đang chạy.
- Trang Web tại `http://localhost:8080` hoạt động bình thường.

> Không sử dụng `docker compose down -v` trong Hands-on này nếu muốn giữ dữ liệu MySQL đã tạo ở Hands-on 02. Tùy chọn `-v` sẽ xóa named volume chứa dữ liệu.

---

# 4. Cho container Web truy cập mã nguồn trong `src`

Ở Hands-on trước, service `web` chỉ mount thư mục:

```yaml
volumes:
  - ./public:/var/www/html
```

Cấu hình này đủ để Apache phục vụ các file trong `public`, nhưng Hands-on 03 cần thêm các file nội bộ trong `src`.

## 4.1. Cập nhật `compose.yaml`

Bổ sung một bind mount cho `src`:

```yaml
services:
  web:
    build:
      context: .
      dockerfile: docker/php/Dockerfile
    ports:
      - "8080:80"
    volumes:
      - ./public:/var/www/html
      - ./src:/var/www/src

  db:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./database/schema.sql:/docker-entrypoint-initdb.d/01-schema.sql:ro
      - ./database/seed.sql:/docker-entrypoint-initdb.d/02-seed.sql:ro

volumes:
  mysql_data:
```

### Giải thích

```yaml
- ./public:/var/www/html
```

ánh xạ thư mục `public` trên máy vào Web root của Apache.

```yaml
- ./src:/var/www/src
```

ánh xạ mã nguồn dùng nội bộ vào `/var/www/src` trong container.

Ta có thể hình dung:

```text
Máy Windows                         Container web

public/  ----------------------->  /var/www/html
                                     Web root của Apache

src/     ----------------------->  /var/www/src
                                     mã PHP dùng nội bộ
```

Việc tách hai khu vực giúp người học nhận ra rằng **không phải toàn bộ mã nguồn của ứng dụng đều cần được trình duyệt truy cập trực tiếp**.

## 4.2. Khởi động lại các service

### Lệnh thực thi

```cmd
docker compose down
docker compose up -d
docker compose ps
```

### Kiểm tra

Mở lại:

```text
http://localhost:8080
```

Trang Web phải tiếp tục hoạt động.

---

# 5. Truyền cấu hình cơ sở dữ liệu vào container Web

Các biến `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD` hiện đã được service `db` sử dụng, nhưng PHP trong service `web` cũng cần các giá trị này để kết nối MySQL.

## 5.1. Cập nhật service `web`

Bổ sung `environment` và `depends_on`:

```yaml
services:
  web:
    build:
      context: .
      dockerfile: docker/php/Dockerfile
    ports:
      - "8080:80"
    volumes:
      - ./public:/var/www/html
      - ./src:/var/www/src
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    depends_on:
      - db
```

Phần `db` giữ nguyên.

### Giải thích `environment`

Ví dụ:

```yaml
MYSQL_DATABASE: ${MYSQL_DATABASE}
```

Compose đọc giá trị `MYSQL_DATABASE` từ `.env` và đưa nó vào môi trường của container `web`.

PHP sau đó có thể đọc giá trị bằng:

```php
getenv('MYSQL_DATABASE');
```

Luồng cấu hình:

```text
.env
  ↓
compose.yaml
  ↓
container web
  ↓
getenv(...)
  ↓
PHP
```

Cách này giúp tránh viết trực tiếp tên database, username và password trong mã PHP.

### Giải thích `depends_on`

```yaml
depends_on:
  - db
```

cho biết service `web` phụ thuộc vào service `db`.

Docker Compose sẽ khởi động `db` trước `web`. Tuy nhiên, cần phân biệt:

> `depends_on` điều phối thứ tự khởi động container, nhưng không mặc nhiên bảo đảm MySQL đã hoàn toàn sẵn sàng nhận kết nối.

## 5.2. Áp dụng cấu hình

```cmd
docker compose down
docker compose up -d
docker compose ps
```

## 5.3. Kiểm tra biến môi trường

```cmd
docker compose exec web printenv MYSQL_DATABASE
docker compose exec web printenv MYSQL_USER
docker compose exec web printenv MYSQL_PASSWORD
```

Kết quả phải tương ứng với các giá trị đã khai báo trong `.env`.

> **Lưu ý bảo mật:** chỉ thực hiện lệnh in mật khẩu trong môi trường thực hành cục bộ. Không chụp hoặc chia sẻ mật khẩu thật trong tài liệu công khai.

---

# 6. Tạo file kết nối MySQL bằng MySQLi

## 6.1. Tạo file

### Lệnh thực thi

```cmd
type nul > src\config\database.php
```

Mở `src/config/database.php` và nhập:

```php
<?php

$host = 'db';
$database = getenv('MYSQL_DATABASE');
$username = getenv('MYSQL_USER');
$password = getenv('MYSQL_PASSWORD');

$conn = new mysqli(
    $host,
    $username,
    $password,
    $database
);

if ($conn->connect_error) {
    die('Kết nối cơ sở dữ liệu thất bại: ' . $conn->connect_error);
}

$conn->set_charset('utf8mb4');
```

## 6.2. Giải thích

### Host `db`

```php
$host = 'db';
```

Trong Docker Compose, `web` và `db` là hai container khác nhau.

Nếu PHP dùng:

```php
$host = 'localhost';
```

thì `localhost` được hiểu là **chính container `web`**, không phải container MySQL.

Docker Compose tạo mạng nội bộ và cho phép các service liên lạc bằng tên service:

```text
web  -------------------->  db
          MySQL
```

Do đó host kết nối là:

```php
$host = 'db';
```

### Đọc cấu hình

```php
$database = getenv('MYSQL_DATABASE');
$username = getenv('MYSQL_USER');
$password = getenv('MYSQL_PASSWORD');
```

Các dòng này lấy cấu hình đã được Compose đưa vào container `web`.

### Tạo kết nối

```php
$conn = new mysqli(
    $host,
    $username,
    $password,
    $database
);
```

`mysqli` là lớp PHP dùng để tạo kết nối tới MySQL.

### Kiểm tra lỗi

```php
if ($conn->connect_error) {
    die('Kết nối cơ sở dữ liệu thất bại: ' . $conn->connect_error);
}
```

Nếu không thể kết nối, chương trình dừng và hiển thị thông báo lỗi để phục vụ quá trình thực hành.

### Thiết lập charset

```php
$conn->set_charset('utf8mb4');
```

Dòng này thiết lập bộ ký tự cho kết nối PHP–MySQL. Đây là một phần quan trọng để dữ liệu tiếng Việt được trao đổi đúng.

### Vì sao không có `?>`?

`database.php` là file chỉ chứa mã PHP nên không bắt buộc đóng bằng:

```php
?>
```

Việc bỏ thẻ đóng trong file PHP thuần giúp tránh khoảng trắng hoặc ký tự xuống dòng ngoài ý muốn sau `?>`, đặc biệt hữu ích khi ứng dụng sử dụng `header()`, session hoặc cookie.

---

# 7. Kiểm tra kết nối cơ sở dữ liệu

Trước khi truy vấn dữ liệu, nên kiểm tra riêng kết nối để dễ xác định lỗi.

## 7.1. Tạo file kiểm tra

### Lệnh thực thi

```cmd
type nul > public\db-test.php
```

Nội dung:

```php
<?php

require_once '/var/www/src/config/database.php';

echo 'Kết nối MySQL thành công.';
```

## 7.2. Truy cập

```text
http://localhost:8080/db-test.php
```

### Kết quả mong đợi

```text
Kết nối MySQL thành công.
```

Luồng đã kiểm tra:

```text
Browser
   ↓
Apache
   ↓
PHP
   ↓
database.php
   ↓
MySQLi
   ↓
db
   ↓
ql_banhang
```

Sau khi kiểm tra thành công, file này chỉ còn vai trò thử nghiệm và sẽ được xóa trước khi commit.

---

# 8. Tách giao diện dùng chung

Ứng dụng sẽ có nhiều trang nhưng cùng sử dụng phần đầu trang, thanh điều hướng và chân trang. Không nên sao chép toàn bộ HTML vào từng trang.

Ta tổ chức:

```text
src/includes/
├── header.php
├── navbar.php
└── footer.php
```

## 8.1. Tạo các file

### Lệnh thực thi

```cmd
type nul > src\includes\header.php
type nul > src\includes\navbar.php
type nul > src\includes\footer.php
```

Kiểm tra:

```cmd
tree src /F
```

Kết quả dự kiến:

```text
src
├── config
│   └── database.php
└── includes
    ├── footer.php
    ├── header.php
    └── navbar.php
```

---

# 9. Tạo `header.php`

Mở `src/includes/header.php`:

```php
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title><?= htmlspecialchars($pageTitle ?? 'Quản lý bán hàng') ?></title>

    <link
        href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.8/dist/css/bootstrap.min.css"
        rel="stylesheet"
    >

    <script
        src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.8/dist/js/bootstrap.bundle.min.js">
    </script>
</head>

<body>

<div class="p-5 bg-primary text-white text-center">
    <h1>Hệ thống quản lý bán hàng</h1>
    <p>Phát triển ứng dụng Web mã nguồn mở</p>
</div>
```

## Giải thích

```php
$pageTitle ?? 'Quản lý bán hàng'
```

sử dụng `$pageTitle` nếu trang hiện tại đã gán giá trị; nếu chưa có thì dùng `"Quản lý bán hàng"`.

Ví dụ:

```php
$pageTitle = 'Quản lý danh mục';
```

sẽ tạo:

```html
<title>Quản lý danh mục</title>
```

`htmlspecialchars()` chuyển các ký tự đặc biệt sang dạng an toàn khi hiển thị trong HTML.

---

# 10. Tạo `navbar.php`

Mở `src/includes/navbar.php`:

```php
<nav class="navbar navbar-expand-lg bg-dark navbar-dark">
    <div class="container">
        <a class="navbar-brand" href="/">Sales Management</a>

        <button
            class="navbar-toggler"
            type="button"
            data-bs-toggle="collapse"
            data-bs-target="#mainNavbar"
        >
            <span class="navbar-toggler-icon"></span>
        </button>

        <div class="collapse navbar-collapse" id="mainNavbar">
            <ul class="navbar-nav">

                <li class="nav-item">
                    <a class="nav-link" href="/">Trang chủ</a>
                </li>

                <li class="nav-item">
                    <a class="nav-link" href="/categories/">
                        Danh mục
                    </a>
                </li>

                <li class="nav-item">
                    <a class="nav-link" href="#">
                        Sản phẩm
                    </a>
                </li>

                <li class="nav-item">
                    <a class="nav-link" href="#">
                        Đơn hàng
                    </a>
                </li>

            </ul>
        </div>
    </div>
</nav>
```

Ở giai đoạn này:

- `Trang chủ` và `Danh mục` có đường dẫn;
- `Sản phẩm` và `Đơn hàng` được giữ làm vị trí cho các Hands-on sau.

---

# 11. Tạo `footer.php`

Mở `src/includes/footer.php`:

```php
<footer class="mt-5 p-4 bg-dark text-white text-center">
    <p class="mb-0">
        Hệ thống quản lý bán hàng
    </p>
</footer>

</body>
</html>
```

Như vậy mỗi trang không cần viết lại phần kết thúc giao diện.

---

# 12. Tạo trang quản lý danh mục

## 12.1. Tạo thư mục và file

### Lệnh thực thi

```cmd
mkdir public\categories
type nul > public\categories\index.php
```

## 12.2. Viết truy vấn và giao diện

Mở `public/categories/index.php`:

```php
<?php

$pageTitle = 'Quản lý danh mục';

require_once '/var/www/src/config/database.php';

$sql = "
    SELECT
        CategoryID,
        CategoryName,
        Description
    FROM categories
    ORDER BY CategoryID
";

$result = $conn->query($sql);

require_once '/var/www/src/includes/header.php';
require_once '/var/www/src/includes/navbar.php';

?>

<div class="container mt-4">

    <div class="d-flex justify-content-between align-items-center mb-3">

        <h2>Quản lý danh mục</h2>

        <a href="#" class="btn btn-primary">
            Thêm danh mục
        </a>

    </div>

    <div class="table-responsive">

        <table class="table table-bordered table-striped">

            <thead class="table-dark">
                <tr>
                    <th>ID</th>
                    <th>Tên danh mục</th>
                    <th>Mô tả</th>
                    <th>Thao tác</th>
                </tr>
            </thead>

            <tbody>

            <?php while ($category = $result->fetch_assoc()): ?>

                <tr>

                    <td>
                        <?= $category['CategoryID'] ?>
                    </td>

                    <td>
                        <?= htmlspecialchars($category['CategoryName']) ?>
                    </td>

                    <td>
                        <?= htmlspecialchars($category['Description'] ?? '') ?>
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

# 13. Phân tích luồng xử lý

## 13.1. Kết nối database

```php
require_once '/var/www/src/config/database.php';
```

File `database.php` tạo biến `$conn` chứa kết nối MySQLi.

## 13.2. Câu truy vấn

```php
$sql = "
    SELECT
        CategoryID,
        CategoryName,
        Description
    FROM categories
    ORDER BY CategoryID
";
```

Câu lệnh yêu cầu MySQL trả về ba cột của bảng `categories` và sắp xếp theo `CategoryID`.

## 13.3. Gửi truy vấn

```php
$result = $conn->query($sql);
```

PHP gửi câu SQL đến MySQL thông qua kết nối `$conn`. Kết quả truy vấn được lưu trong `$result`.

## 13.4. Đọc từng bản ghi

```php
while ($category = $result->fetch_assoc()):
```

`fetch_assoc()` lấy một bản ghi và trả về dưới dạng mảng kết hợp.

Ví dụ:

```php
$category['CategoryName']
```

tương ứng với cột `CategoryName` trong kết quả truy vấn.

Vòng lặp tiếp tục cho đến khi không còn bản ghi.

## 13.5. Hiển thị dữ liệu an toàn

```php
<?= htmlspecialchars($category['CategoryName']) ?>
```

`<?= ... ?>` là cú pháp rút gọn của `echo`.

`htmlspecialchars()` được dùng cho dữ liệu văn bản trước khi đưa vào HTML.

## 13.6. Đóng kết nối

```php
$conn->close();
```

Kết thúc kết nối MySQL sau khi trang đã hoàn thành công việc với cơ sở dữ liệu.

---

# 14. Kiểm tra kết quả

Mở:

```text
http://localhost:8080/categories/
```

Trang phải hiển thị dữ liệu danh mục đã tạo trong Hands-on 02, ví dụ:

| ID | Tên danh mục | Mô tả |
|---:|---|---|
| 1 | Điện thoại | Các sản phẩm điện thoại di động |
| 2 | Máy tính | Máy tính xách tay và máy tính để bàn |
| 3 | Phụ kiện | Phụ kiện dành cho thiết bị công nghệ |

Các nút:

```text
Thêm danh mục
Sửa
Xóa
```

đã xuất hiện nhưng **chưa thực hiện chức năng** trong Hands-on này.

Luồng hoàn chỉnh:

```text
Browser
   ↓
public/categories/index.php
   ↓
src/config/database.php
   ↓
MySQLi
   ↓
MySQL / categories
   ↓
$result
   ↓
fetch_assoc()
   ↓
Bootstrap table
```

---

# 15. Xóa file kiểm tra tạm thời

Sau khi kết nối đã được xác nhận, xóa `db-test.php`:

```cmd
del public\db-test.php
```

File kiểm tra đã hoàn thành nhiệm vụ và không cần giữ trong phiên bản project chính thức.

---

# 16. Kiểm tra trước khi commit

### Lệnh thực thi

```cmd
git status
git diff
```

Kiểm tra các thay đổi đúng với Hands-on 03.

Các file chính dự kiến:

```text
compose.yaml
src/config/database.php
src/includes/header.php
src/includes/navbar.php
src/includes/footer.php
public/categories/index.php
```

Đồng thời kiểm tra lại:

```text
http://localhost:8080
http://localhost:8080/categories/
```

---

# 17. Commit và push lên GitHub

## 17.1. Stage

```cmd
git add .
git status
```

Kiểm tra danh sách file trước khi commit.

## 17.2. Commit

```cmd
git commit -m "Add MySQLi connection and category listing"
```

## 17.3. Push

```cmd
git push
```

## 17.4. Kiểm tra

```cmd
git status
git log -1 --oneline
```

Kết quả mong đợi:

```text
nothing to commit, working tree clean
```

và commit mới nhất có nội dung:

```text
Add MySQLi connection and category listing
```

---

# 18. Checkpoint Hands-on 03

Trước khi kết thúc, xác nhận:

```text
[ ] web và db đang chạy
[ ] public được mount vào /var/www/html
[ ] src được mount vào /var/www/src
[ ] biến cấu hình database được truyền vào container web
[ ] database.php kết nối MySQL bằng MySQLi
[ ] host kết nối là db
[ ] kết nối sử dụng utf8mb4
[ ] header.php hoạt động
[ ] navbar.php hoạt động
[ ] footer.php hoạt động
[ ] categories/index.php truy vấn được MySQL
[ ] dữ liệu tiếng Việt hiển thị đúng
[ ] danh sách danh mục hiển thị từ dữ liệu thật
[ ] db-test.php đã được xóa
[ ] thay đổi đã được commit và push
```

---

# 19. Kiến thức cần ghi nhớ

## 19.1. `public` và `src` có vai trò khác nhau

```text
public/
→ tài nguyên/trang được Web server phục vụ

src/
→ mã dùng nội bộ của ứng dụng
```

Không cần đặt toàn bộ mã nguồn vào Web root.

## 19.2. Các container liên lạc bằng tên service

Trong Compose:

```text
web → db
```

Do đó PHP sử dụng:

```php
$host = 'db';
```

không phải `localhost`.

## 19.3. Không hard-code thông tin đăng nhập trong PHP

Luồng phù hợp:

```text
.env
 ↓
compose.yaml
 ↓
environment
 ↓
getenv()
 ↓
PHP
```

## 19.4. MySQLi là cầu nối PHP–MySQL

```text
PHP
 ↓
MySQLi
 ↓
MySQL
```

## 19.5. Tách layout giúp tái sử dụng giao diện

Thay vì sao chép header/navbar/footer vào từng trang:

```text
header.php
navbar.php
footer.php
       ↓
require_once
       ↓
nhiều trang cùng sử dụng
```

## 19.6. Dữ liệu Web đã chuyển từ tĩnh sang động

Trước Hands-on 03:

```text
HTML → dữ liệu viết trực tiếp trong file
```

Sau Hands-on 03:

```text
MySQL
  ↓
PHP
  ↓
HTML
  ↓
Browser
```

Đây là bước chuyển quan trọng từ trang HTML tĩnh sang ứng dụng Web có dữ liệu động.

---

# 20. Chuẩn bị cho Hands-on tiếp theo

Hands-on 03 mới thực hiện thao tác **đọc dữ liệu**:

```text
SELECT
```

Các nút `Thêm danh mục`, `Sửa`, `Xóa` mới chỉ là giao diện.

Hands-on tiếp theo sẽ phát triển chức năng quản lý danh mục hoàn chỉnh:

```text
Create
Read
Update
Delete
```

hay:

```text
CRUD Category
```

Từ đó form Bootstrap quản lý danh mục sẽ được kết nối với MySQL để thực hiện thao tác dữ liệu thật.
