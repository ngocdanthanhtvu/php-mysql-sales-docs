# Hands-on 04 – Xây dựng CRUD quản lý danh mục

## 1. Mục tiêu

Sau khi hoàn thành Hands-on này, người học có thể:

- giải thích khái niệm CRUD trong ứng dụng Web;
- phân biệt vai trò của HTTP `GET` và `POST` trong các thao tác dữ liệu cơ bản;
- nhận dữ liệu biểu mẫu bằng `$_POST`;
- thực hiện `INSERT`, `SELECT`, `UPDATE`, `DELETE` với MySQLi;
- sử dụng **prepared statement** và `bind_param()` khi đưa dữ liệu vào câu SQL;
- truyền mã danh mục qua URL khi cần xác định bản ghi để chỉnh sửa;
- sử dụng `input type="hidden"` để gửi mã bản ghi khi xóa;
- chuyển hướng người dùng bằng `header('Location: ...')`;
- giải thích vì sao thao tác xóa không nên được thực hiện trực tiếp bằng một liên kết GET;
- nhận biết ảnh hưởng của khóa ngoại khi xóa dữ liệu đang được bảng khác tham chiếu.

> **Nguyên tắc thực hành:** Project được phát triển từng bước từ kết quả Hands-on 03. Với file đã tồn tại, chỉ sửa hoặc bổ sung phần cần thiết. Không thay toàn bộ file nếu chỉ cần thay một thuộc tính hoặc một đoạn mã nhỏ.

---

## 2. Kết quả đầu vào từ Hands-on 03

Trang:

```text
http://localhost:8080/categories/
```

đã đọc dữ liệu từ bảng `categories` và hiển thị bằng Bootstrap.

Cấu trúc liên quan:

```text
public/
└── categories/
    └── index.php

src/
├── config/
│   └── database.php
└── includes/
    ├── header.php
    ├── navbar.php
    └── footer.php
```

Trong `index.php`, giao diện đã có các nút:

```text
Thêm danh mục
Sửa
Xóa
```

nhưng các nút này chưa thực hiện thao tác dữ liệu.

---

# 3. CRUD là gì?

CRUD là bốn thao tác cơ bản khi làm việc với dữ liệu:

| CRUD | SQL | Chức năng trong bài |
|---|---|---|
| Create | `INSERT` | Thêm danh mục |
| Read | `SELECT` | Xem danh sách danh mục |
| Update | `UPDATE` | Sửa danh mục |
| Delete | `DELETE` | Xóa danh mục |

Phần **Read** đã được hoàn thành ở Hands-on 03.

Trong Hands-on này, ta tiếp tục:

```text
Create
   ↓
Update
   ↓
Delete
```

---

# 4. Chức năng Create – Thêm danh mục

## 4.1. Tạo trang thêm danh mục

### Lệnh thực thi

```cmd
type nul > public\categories\create.php
```

Đây là file mới nên ta xây dựng đầy đủ nội dung.

Mở `public/categories/create.php` và nhập:

```php
<?php

$pageTitle = 'Thêm danh mục';

require_once '/var/www/src/includes/header.php';
require_once '/var/www/src/includes/navbar.php';

?>

<div class="container mt-4">

    <h2 class="mb-4">Thêm danh mục</h2>

    <form method="post">

        <div class="mb-3">
            <label for="categoryName" class="form-label">
                Tên danh mục
            </label>

            <input
                type="text"
                class="form-control"
                id="categoryName"
                name="category_name"
                required
            >
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
            ></textarea>
        </div>

        <button type="submit" class="btn btn-primary">
            Lưu
        </button>

        <a href="/categories/" class="btn btn-secondary">
            Hủy
        </a>

    </form>

</div>

<?php

require_once '/var/www/src/includes/footer.php';
```

### Kiến thức mới

```html
<form method="post">
```

cho biết dữ liệu biểu mẫu sẽ được gửi bằng HTTP POST.

Do không khai báo `action`, dữ liệu được gửi về chính:

```text
/categories/create.php
```

Luồng ban đầu:

```text
GET create.php
      ↓
hiển thị form
      ↓
người dùng nhập dữ liệu
      ↓
POST create.php
```

Hai thuộc tính:

```html
name="category_name"
name="description"
```

xác định tên dữ liệu mà PHP sẽ nhận.

Ví dụ:

```php
$_POST['category_name']
$_POST['description']
```

Thuộc tính:

```html
required
```

yêu cầu trình duyệt kiểm tra trường tên danh mục trước khi gửi form.

---

# 5. Liên kết nút “Thêm danh mục”

`public/categories/index.php` đã tồn tại từ Hands-on 03. Vì vậy **không thay toàn bộ thẻ hoặc toàn bộ file**.

Tìm nút **Thêm danh mục** đang có:

```text
href="#"
```

Chỉ thay giá trị `href` bằng:

```text
/categories/create.php
```

Sau thay đổi, nhấn nút **Thêm danh mục** phải mở:

```text
http://localhost:8080/categories/create.php
```

## Checkpoint 04-01

Kiểm tra:

```text
[ ] create.php hiển thị
[ ] nút Thêm danh mục đã liên kết tới create.php
[ ] form có Tên danh mục
[ ] form có Mô tả
[ ] trường Tên danh mục có required
[ ] nút Hủy quay về danh sách
```

Ở checkpoint này chưa ghi dữ liệu vào MySQL.

---

# 6. Xử lý dữ liệu POST

Bây giờ bổ sung phần xử lý vào `create.php`.

## 6.1. Kết nối database

Trong phần PHP ở đầu file, ngay sau:

```php
$pageTitle = 'Thêm danh mục';
```

thêm:

```php
require_once '/var/www/src/config/database.php';

$error = '';
```

`$error` dùng để lưu thông báo khi dữ liệu không hợp lệ hoặc không thể ghi vào database.

---

## 6.2. Nhận dữ liệu từ form

Ngay sau phần trên, thêm:

```php
if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    $categoryName = trim($_POST['category_name'] ?? '');
    $description = trim($_POST['description'] ?? '');

    if ($categoryName === '') {

        $error = 'Tên danh mục không được để trống.';

    } else {

        // Phần INSERT sẽ bổ sung ở bước tiếp theo.

    }
}
```

### Giải thích

```php
$_SERVER['REQUEST_METHOD']
```

cho biết phương thức HTTP của request.

Điều kiện:

```php
$_SERVER['REQUEST_METHOD'] === 'POST'
```

chỉ xử lý dữ liệu khi người dùng thực sự gửi form.

Toán tử:

```php
?? ''
```

cung cấp chuỗi rỗng nếu phần tử tương ứng không tồn tại.

Hàm:

```php
trim(...)
```

loại bỏ khoảng trắng thừa ở đầu và cuối chuỗi.

Ta vẫn kiểm tra:

```php
$categoryName === ''
```

ở phía server dù HTML đã có `required`.

> Kiểm tra ở trình duyệt giúp trải nghiệm người dùng tốt hơn; kiểm tra ở server mới là phần xử lý mà ứng dụng phải kiểm soát.

---

# 7. Thực hiện INSERT bằng prepared statement

Thay dòng chú thích:

```php
// Phần INSERT sẽ bổ sung ở bước tiếp theo.
```

bằng:

```php
$sql = "
    INSERT INTO categories
        (CategoryName, Description)
    VALUES
        (?, ?)
";

$stmt = $conn->prepare($sql);

$stmt->bind_param(
    'ss',
    $categoryName,
    $description
);

if ($stmt->execute()) {

    header('Location: /categories/');
    exit;

} else {

    $error = 'Không thể thêm danh mục.';
}

$stmt->close();
```

## 7.1. Vì sao dùng dấu `?`

Trong câu SQL:

```sql
VALUES (?, ?)
```

hai dấu `?` là các **placeholder**.

Ta không ghép trực tiếp dữ liệu người dùng vào chuỗi SQL.

Dữ liệu được gắn sau bằng:

```php
$stmt->bind_param(
    'ss',
    $categoryName,
    $description
);
```

Trong đó:

```text
s → string
s → string
```

Hai giá trị đều là chuỗi.

Prepared statement giúp tách **cấu trúc câu SQL** khỏi **dữ liệu đầu vào**, đồng thời là biện pháp quan trọng để hạn chế SQL Injection.

---

# 8. Hiển thị lỗi và giữ lại dữ liệu đã nhập

## 8.1. Hiển thị thông báo lỗi

Trong phần HTML của `create.php`, ngay dưới:

```html
<h2 class="mb-4">Thêm danh mục</h2>
```

chèn:

```php
<?php if ($error !== ''): ?>

    <div class="alert alert-danger">
        <?= htmlspecialchars($error) ?>
    </div>

<?php endif; ?>
```

---

## 8.2. Giữ lại tên danh mục

Trong thẻ `<input>` của tên danh mục, bổ sung thuộc tính:

```php
value="<?= htmlspecialchars($_POST['category_name'] ?? '') ?>"
```

Không cần thay toàn bộ thẻ `<input>`.

Mục đích: nếu server phát hiện lỗi, dữ liệu người dùng vừa nhập không bị mất khỏi form.

---

## 8.3. Giữ lại mô tả

Trong `<textarea>`, thay phần nội dung rỗng giữa thẻ mở và đóng bằng:

```php
<?= htmlspecialchars($_POST['description'] ?? '') ?>
```

Kết quả về ý nghĩa:

```text
POST không hợp lệ
      ↓
hiển thị lại form
      ↓
giữ dữ liệu người dùng vừa nhập
```

---

# 9. Redirect sau khi thêm thành công

Đoạn:

```php
header('Location: /categories/');
exit;
```

yêu cầu trình duyệt chuyển về trang danh sách sau khi `INSERT` thành công.

Luồng:

```text
POST create.php
      ↓
INSERT thành công
      ↓
Location: /categories/
      ↓
GET /categories/
      ↓
hiển thị danh sách mới
```

`exit` kết thúc script ngay sau khi gửi lệnh chuyển hướng.

---

# 10. Kiểm tra Create

Mở:

```text
http://localhost:8080/categories/create.php
```

Thử nhập:

```text
Tên danh mục: Thiết bị mạng
Mô tả: Router, switch và các thiết bị mạng
```

Nhấn **Lưu**.

Kết quả mong đợi:

1. dữ liệu được thêm vào MySQL;
2. trình duyệt quay về `/categories/`;
3. danh mục mới xuất hiện trong bảng;
4. tiếng Việt hiển thị đúng.

## Checkpoint 04-02

```text
[ ] PHP nhận dữ liệu bằng POST
[ ] dữ liệu được trim
[ ] server kiểm tra tên danh mục rỗng
[ ] INSERT dùng prepared statement
[ ] bind_param dùng kiểu ss
[ ] INSERT thành công
[ ] redirect về danh sách
[ ] danh mục mới xuất hiện
```

---

# 11. Chức năng Update – Sửa danh mục

Chức năng sửa cần biết **bản ghi nào** đang được chỉnh sửa.

Ta sử dụng URL dạng:

```text
/categories/edit.php?id=2
```

Trong đó:

```text
id=2
```

xác định danh mục có `CategoryID = 2`.

---

# 12. Cập nhật liên kết “Sửa”

Trong `public/categories/index.php`, tìm nút **Sửa**.

Hiện tại thuộc tính là:

```text
href="#"
```

Chỉ thay giá trị `href` bằng:

```php
/categories/edit.php?id=<?= $category['CategoryID'] ?>
```

Không thay toàn bộ thẻ `<a>`.

Sau thay đổi, mỗi dòng sẽ tạo URL khác nhau, ví dụ:

```text
/categories/edit.php?id=1
/categories/edit.php?id=2
/categories/edit.php?id=3
```

---

# 13. Tạo `edit.php`

### Lệnh thực thi

```cmd
type nul > public\categories\edit.php
```

Đây là file mới nên nhập đầy đủ nội dung:

```php
<?php

$pageTitle = 'Sửa danh mục';

require_once '/var/www/src/config/database.php';

$error = '';

$categoryID = isset($_GET['id'])
    ? (int) $_GET['id']
    : 0;

if ($categoryID <= 0) {
    die('Mã danh mục không hợp lệ.');
}

/*
 * Đọc dữ liệu hiện tại của danh mục
 */
$sql = "
    SELECT
        CategoryID,
        CategoryName,
        Description
    FROM categories
    WHERE CategoryID = ?
";

$stmt = $conn->prepare($sql);
$stmt->bind_param('i', $categoryID);
$stmt->execute();

$result = $stmt->get_result();
$category = $result->fetch_assoc();

$stmt->close();

if (!$category) {
    die('Không tìm thấy danh mục.');
}


/*
 * Xử lý khi người dùng gửi form
 */
if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    $categoryName = trim($_POST['category_name'] ?? '');
    $description = trim($_POST['description'] ?? '');

    if ($categoryName === '') {

        $error = 'Tên danh mục không được để trống.';

    } else {

        $sql = "
            UPDATE categories
            SET
                CategoryName = ?,
                Description = ?
            WHERE CategoryID = ?
        ";

        $stmt = $conn->prepare($sql);

        $stmt->bind_param(
            'ssi',
            $categoryName,
            $description,
            $categoryID
        );

        if ($stmt->execute()) {

            header('Location: /categories/');
            exit;

        } else {

            $error = 'Không thể cập nhật danh mục.';
        }

        $stmt->close();
    }
}

require_once '/var/www/src/includes/header.php';
require_once '/var/www/src/includes/navbar.php';

?>

<div class="container mt-4">

    <h2 class="mb-4">Sửa danh mục</h2>

    <?php if ($error !== ''): ?>

        <div class="alert alert-danger">
            <?= htmlspecialchars($error) ?>
        </div>

    <?php endif; ?>

    <form method="post">

        <div class="mb-3">

            <label class="form-label">
                Mã danh mục
            </label>

            <input
                type="text"
                class="form-control"
                value="<?= $category['CategoryID'] ?>"
                disabled
            >

        </div>

        <div class="mb-3">

            <label for="categoryName" class="form-label">
                Tên danh mục
            </label>

            <input
                type="text"
                class="form-control"
                id="categoryName"
                name="category_name"
                value="<?= htmlspecialchars(
                    $_POST['category_name']
                    ?? $category['CategoryName']
                ) ?>"
                required
            >

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
                $_POST['description']
                ?? $category['Description']
                ?? ''
            ) ?></textarea>

        </div>

        <button type="submit" class="btn btn-warning">
            Cập nhật
        </button>

        <a href="/categories/" class="btn btn-secondary">
            Hủy
        </a>

    </form>

</div>

<?php

require_once '/var/www/src/includes/footer.php';

$conn->close();
```

---

# 14. Phân tích luồng Update

## 14.1. GET xác định bản ghi

Ví dụ:

```text
GET /categories/edit.php?id=2
```

PHP đọc:

```php
$_GET['id']
```

sau đó chuyển sang số nguyên:

```php
(int) $_GET['id']
```

Nếu ID không hợp lệ:

```php
if ($categoryID <= 0)
```

script dừng.

---

## 14.2. SELECT dữ liệu hiện tại

```sql
SELECT
    CategoryID,
    CategoryName,
    Description
FROM categories
WHERE CategoryID = ?
```

Prepared statement tiếp tục được sử dụng ngay cả khi ID đã được ép sang số nguyên.

Kiểu:

```text
i → integer
```

được khai báo bằng:

```php
$stmt->bind_param('i', $categoryID);
```

---

## 14.3. Hiển thị dữ liệu cũ

Kết quả `SELECT` được lấy bằng:

```php
$category = $result->fetch_assoc();
```

Sau đó đưa vào form.

Ví dụ:

```php
$category['CategoryName']
```

là tên danh mục hiện tại.

---

## 14.4. POST cập nhật dữ liệu

Khi nhấn **Cập nhật**, request trở thành:

```text
POST /categories/edit.php?id=2
```

ID vẫn xác định bản ghi cần sửa; dữ liệu mới nằm trong `$_POST`.

Câu SQL:

```sql
UPDATE categories
SET
    CategoryName = ?,
    Description = ?
WHERE CategoryID = ?
```

---

## 14.5. Ý nghĩa `ssi`

```php
$stmt->bind_param(
    'ssi',
    $categoryName,
    $description,
    $categoryID
);
```

Trong đó:

```text
s → CategoryName → string
s → Description  → string
i → CategoryID   → integer
```

---

# 15. Kiểm tra Update

Tại:

```text
http://localhost:8080/categories/
```

nhấn **Sửa** một danh mục.

Ví dụ URL:

```text
http://localhost:8080/categories/edit.php?id=2
```

Kiểm tra form phải hiển thị dữ liệu hiện tại.

Thay đổi tên hoặc mô tả rồi nhấn:

```text
Cập nhật
```

Kết quả mong đợi:

```text
GET edit.php?id=...
       ↓
SELECT dữ liệu cũ
       ↓
hiển thị form
       ↓
POST dữ liệu mới
       ↓
UPDATE
       ↓
redirect
       ↓
danh sách đã thay đổi
```

## Checkpoint 04-03

```text
[ ] nút Sửa truyền đúng CategoryID
[ ] edit.php đọc id từ GET
[ ] ID được kiểm tra
[ ] dữ liệu hiện tại được SELECT
[ ] form hiển thị dữ liệu cũ
[ ] UPDATE dùng prepared statement
[ ] bind_param dùng ssi
[ ] cập nhật thành công
[ ] redirect về danh sách
```

---

# 16. Chức năng Delete – Xóa danh mục

Không nên thiết kế thao tác xóa bằng URL như:

```text
/categories/delete.php?id=4
```

rồi xóa dữ liệu ngay khi URL được truy cập.

Trong bài này ta sử dụng:

```text
POST → delete.php
```

thay vì:

```text
GET → delete.php?id=...
```

---

# 17. Thay nút Xóa bằng form POST

Ở bước này thay đổi nhiều hơn một thuộc tính vì phần tử ban đầu là liên kết `<a>`, trong khi thao tác xóa cần một `<form method="post">`.

Trong `public/categories/index.php`, **chỉ tìm đúng thẻ nút Xóa hiện tại**:

```php
<a href="#" class="btn btn-sm btn-danger">
    Xóa
</a>
```

Thay **riêng thẻ này** bằng:

```php
<form
    action="/categories/delete.php"
    method="post"
    class="d-inline"
    onsubmit="return confirm('Bạn có chắc muốn xóa danh mục này?');"
>
    <input
        type="hidden"
        name="id"
        value="<?= $category['CategoryID'] ?>"
    >

    <button
        type="submit"
        class="btn btn-sm btn-danger"
    >
        Xóa
    </button>
</form>
```

Không thay các phần còn lại của `index.php`.

---

# 18. Phân tích form xóa

## 18.1. Gửi bằng POST

```html
method="post"
```

cho biết đây là thao tác làm thay đổi dữ liệu.

Ở mức cơ bản của Hands-on:

```text
GET
→ đọc/truy cập dữ liệu

POST
→ gửi dữ liệu để thực hiện thay đổi
```

---

## 18.2. Hidden input

```html
<input
    type="hidden"
    name="id"
    value="<?= $category['CategoryID'] ?>"
>
```

ID cần được gửi đến server nhưng người dùng không cần nhập lại.

PHP sẽ đọc bằng:

```php
$_POST['id']
```

---

## 18.3. Xác nhận trước khi xóa

```html
onsubmit="return confirm('Bạn có chắc muốn xóa danh mục này?');"
```

Nếu người dùng chọn **Cancel**, form không được gửi.

Nếu chọn **OK**, form gửi POST tới `delete.php`.

> `confirm()` chỉ là bước xác nhận phía giao diện. Quyết định xử lý dữ liệu vẫn phải được thực hiện ở phía server.

---

# 19. Tạo `delete.php`

### Lệnh thực thi

```cmd
type nul > public\categories\delete.php
```

Nhập:

```php
<?php

require_once '/var/www/src/config/database.php';

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    header('Location: /categories/');
    exit;
}

$categoryID = isset($_POST['id'])
    ? (int) $_POST['id']
    : 0;

if ($categoryID <= 0) {
    header('Location: /categories/');
    exit;
}

$sql = "
    DELETE FROM categories
    WHERE CategoryID = ?
";

$stmt = $conn->prepare($sql);
$stmt->bind_param('i', $categoryID);

$stmt->execute();

$stmt->close();
$conn->close();

header('Location: /categories/');
exit;
```

---

# 20. Phân tích xử lý Delete

## 20.1. Chỉ chấp nhận POST

```php
if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    header('Location: /categories/');
    exit;
}
```

Nếu người dùng cố mở trực tiếp:

```text
/categories/delete.php
```

bằng GET, script không xóa dữ liệu mà chuyển về danh sách.

---

## 20.2. Nhận ID

```php
$categoryID = isset($_POST['id'])
    ? (int) $_POST['id']
    : 0;
```

ID được lấy từ hidden input và chuyển sang số nguyên.

---

## 20.3. DELETE bằng prepared statement

```sql
DELETE FROM categories
WHERE CategoryID = ?
```

và:

```php
$stmt->bind_param('i', $categoryID);
```

tiếp tục áp dụng cùng nguyên tắc đã học ở Create và Update.

---

# 21. Kiểm tra Delete

Nên thử với danh mục vừa tạo trong phần Create, ví dụ:

```text
Thiết bị mạng
```

Mở:

```text
http://localhost:8080/categories/
```

Nhấn **Xóa**.

Luồng mong đợi:

```text
[Xóa]
  ↓
confirm()
  ↓
OK
  ↓
POST delete.php
  ↓
DELETE
  ↓
redirect
  ↓
danh mục biến mất
```

## Checkpoint 04-04

```text
[ ] nút Xóa gửi bằng POST
[ ] CategoryID được gửi bằng hidden input
[ ] có hộp xác nhận
[ ] delete.php chỉ chấp nhận POST
[ ] ID được kiểm tra
[ ] DELETE dùng prepared statement
[ ] xóa được danh mục thử nghiệm
[ ] redirect về danh sách
```

---

# 22. Trường hợp không thể xóa do khóa ngoại

Bảng `products` có thể chứa các sản phẩm đang tham chiếu đến `categories`.

Ví dụ:

```text
categories
CategoryID = 1
      ↑
      │
products
CategoryID = 1
```

Nếu cố xóa danh mục đang được sản phẩm sử dụng, MySQL có thể từ chối thao tác để bảo vệ tính toàn vẹn tham chiếu.

Điều này giúp tránh tình trạng:

```text
sản phẩm còn tồn tại
       ↓
CategoryID trỏ tới danh mục không còn tồn tại
```

Vì vậy khi thử Delete trong Hands-on này, nên xóa **danh mục thử nghiệm vừa tạo**, thay vì xóa các danh mục seed đang được sản phẩm sử dụng.

> Ở phiên bản hiện tại, Hands-on chưa xây dựng giao diện xử lý lỗi khóa ngoại hoàn chỉnh. Mục tiêu ở đây là nhận biết vai trò của ràng buộc dữ liệu. Việc xử lý lỗi thân thiện hơn có thể được bổ sung khi ứng dụng được hoàn thiện.

---

# 23. Tổng hợp luồng CRUD

Sau Hands-on 04:

```text
                 categories
                     │
       ┌─────────────┼─────────────┐
       │             │             │
     INSERT        SELECT        UPDATE
       │             │             │
       ↓             ↓             ↓
    Create          Read         Update
       │                           │
       └─────────────┬─────────────┘
                     │
                   DELETE
                     ↓
                   Delete
```

Hay ngắn gọn:

```text
Create  → INSERT
Read    → SELECT
Update  → UPDATE
Delete  → DELETE
```

---

# 24. Kiểm tra trước khi commit

### Lệnh thực thi

```cmd
git status
git diff
```

Các thay đổi chính dự kiến:

```text
public/categories/index.php
public/categories/create.php
public/categories/edit.php
public/categories/delete.php
```

Kiểm tra lại:

```text
http://localhost:8080/categories/
```

và thử lần lượt:

```text
Thêm
Sửa
Xóa
```

---

# 25. Commit và push lên GitHub

### Stage

```cmd
git add .
git status
```

### Commit

```cmd
git commit -m "Add category CRUD operations"
```

### Push

```cmd
git push
```

### Kiểm tra

```cmd
git status
git log -1 --oneline
```

Kết quả checkpoint của bài thực hành:

```text
3f3e863 Add category CRUD operations
```

và:

```text
nothing to commit, working tree clean
```

---

# 26. Checkpoint Hands-on 04

```text
[ ] Create hoạt động
[ ] Read hoạt động
[ ] Update hoạt động
[ ] Delete hoạt động
[ ] dữ liệu form được nhận bằng POST
[ ] INSERT dùng prepared statement
[ ] UPDATE dùng prepared statement
[ ] DELETE dùng prepared statement
[ ] hiểu ý nghĩa s, i trong bind_param()
[ ] thao tác xóa không dùng GET trực tiếp
[ ] có xác nhận trước khi xóa
[ ] hiểu vai trò của hidden input
[ ] hiểu vai trò của redirect
[ ] nhận biết ảnh hưởng của khóa ngoại
[ ] dữ liệu tiếng Việt hiển thị đúng
[ ] Git working tree clean
[ ] đã push commit lên GitHub
```

---

# 27. Kiến thức cần ghi nhớ

### GET và POST

```text
GET
→ thường dùng để lấy/truy cập tài nguyên

POST
→ gửi dữ liệu để thực hiện xử lý/thay đổi
```

Trong CRUD thực tế còn có các HTTP method khác như `PUT`, `PATCH`, `DELETE`. Chúng sẽ phù hợp hơn khi xây dựng REST API ở các Hands-on sau.

### `$_GET`

Đọc dữ liệu truyền qua query string:

```text
edit.php?id=2
```

```php
$_GET['id']
```

### `$_POST`

Đọc dữ liệu gửi từ form:

```php
$_POST['category_name']
```

### Prepared statement

Thay vì ghép dữ liệu vào SQL:

```text
SQL + dữ liệu người dùng
```

ta dùng:

```text
SQL có placeholder
        ↓
prepare()
        ↓
bind_param()
        ↓
execute()
```

### `bind_param()`

Một số kiểu thường gặp:

```text
s → string
i → integer
d → double
```

Ví dụ:

```php
'ssi'
```

có nghĩa:

```text
string
string
integer
```

### Redirect

```php
header('Location: /categories/');
exit;
```

chuyển người dùng về trang danh sách sau khi xử lý thành công.

### Khóa ngoại

Khóa ngoại giúp bảo vệ mối quan hệ giữa các bảng và ngăn một số thao tác có thể làm dữ liệu mất tính nhất quán.

---

# 28. Từ trang Web động đến CRUD

Hands-on 03 đã chuyển:

```text
HTML tĩnh
   ↓
PHP + MySQL
   ↓
dữ liệu động
```

Hands-on 04 tiếp tục:

```text
dữ liệu động
   ↓
Create
Read
Update
Delete
   ↓
ứng dụng quản lý dữ liệu
```

Điểm quan trọng không chỉ là các nút **Thêm – Sửa – Xóa** hoạt động, mà là hiểu được mỗi thao tác giao diện được ánh xạ như thế nào sang HTTP, PHP, MySQLi và SQL.

---

# 29. Chuẩn bị cho Hands-on tiếp theo

Sau khi CRUD Category hoàn chỉnh, project đã có nền tảng để phát triển các chức năng có quan hệ dữ liệu phức tạp hơn.

Ở các bước tiếp theo, có thể mở rộng sang quản lý **sản phẩm**, nơi người học sẽ phải làm việc với:

```text
products
   ↓
CategoryID
   ↓
categories
```

và các vấn đề mới như:

```text
khóa ngoại
JOIN
chọn danh mục
giá sản phẩm
tồn kho
hình ảnh sản phẩm
```

Đây là bước tiếp theo để chuyển từ CRUD một bảng độc lập sang xử lý dữ liệu có quan hệ trong ứng dụng bán hàng.
