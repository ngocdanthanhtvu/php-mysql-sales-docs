# Hands-on 14 -- Đăng ký, đăng nhập và tài khoản khách hàng

## Mục tiêu

Sau khi hoàn thành Hands-on này, bạn có thể:

-   mở rộng bảng `customers` để hỗ trợ tài khoản khách hàng;
-   phân biệt khách vãng lai và khách có tài khoản trên cùng một bảng dữ
    liệu;
-   tạo chức năng đăng ký tài khoản với email duy nhất;
-   lưu mật khẩu an toàn bằng `password_hash()` thay vì lưu mật khẩu
    gốc;
-   xác thực tài khoản bằng `password_verify()`;
-   sử dụng PHP Session để duy trì trạng thái đăng nhập;
-   thay đổi Navbar theo trạng thái đăng nhập;
-   đăng xuất mà không làm mất giỏ hàng đang lưu trong Session;
-   sử dụng `CustomerID` trong Session để nhận diện khách hàng khi
    Checkout;
-   tự động đọc thông tin khách hàng từ cơ sở dữ liệu khi đặt hàng;
-   tái sử dụng cùng một `CustomerID` cho nhiều đơn hàng của khách đã
    đăng nhập;
-   vẫn duy trì luồng đặt hàng dành cho khách vãng lai.

> **Phạm vi Hands-on 14:** xây dựng đăng ký, đăng nhập và nhận diện
> **khách hàng** ở frontend. Hands-on này chưa xây dựng đăng nhập hoặc
> phân quyền cho khu vực Admin. Khách chưa có tài khoản vẫn có thể mua
> hàng.

------------------------------------------------------------------------

## 1. Chuẩn bị

Hands-on này tiếp tục trực tiếp từ **Hands-on 13**. Hệ thống hiện đã có:

``` text
Frontend
   ├── Danh sách sản phẩm
   ├── Chi tiết sản phẩm
   ├── Giỏ hàng
   └── Checkout
          ↓
       customers
          ↓
        orders
          ↓
     orderdetail

Admin
   └── Quản lý đơn hàng
```

Ở phiên bản hiện tại, mỗi lần khách vãng lai Checkout, hệ thống tạo một
mẩu tin mới trong `customers`.

Trong Hands-on này, ta bổ sung thêm một khả năng:

``` text
Khách vãng lai
→ vẫn đặt hàng như trước

Khách có tài khoản
→ đăng nhập
→ hệ thống nhận diện CustomerID
→ dùng lại CustomerID khi đặt những đơn tiếp theo
```

Mở **Command Prompt** và chuyển đến thư mục dự án:

``` cmd
cd /d D:\PTUDW-ST-2026\php-mysql-sales
```

Kiểm tra Git:

``` cmd
git status
```

Nên bắt đầu khi có:

``` text
nothing to commit, working tree clean
```

Kiểm tra Docker:

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

## 2. Phân tích thiết kế tài khoản khách hàng

### 2.1. Vì sao tiếp tục sử dụng bảng `customers`?

Bảng `customers` đã được `orders` tham chiếu thông qua `CustomerID`. Vì
vậy, thay vì tạo thêm một bảng tài khoản tách biệt trong phạm vi bài
thực hành này, ta mở rộng `customers` bằng hai thuộc tính:

``` text
Email
PasswordHash
```

Khi đó:

``` text
customers
├── CustomerID
├── CustomerName
├── Phone
├── Address
├── ...
├── Email
└── PasswordHash
```

Một khách hàng có tài khoản sẽ có `Email` và `PasswordHash`. Khách vãng
lai vẫn được lưu trong cùng bảng nhưng hai trường này có thể để `NULL`.

### 2.2. Vì sao phải thay đổi cơ sở dữ liệu trước?

Trong Hands-on này, **cơ sở dữ liệu phải được cập nhật trước khi viết
`register.php` và `login.php`**.

Lý do là các chức năng đăng ký và đăng nhập sẽ truy cập trực tiếp hai
cột `Email` và `PasswordHash`. Nếu viết PHP trước nhưng bảng `customers`
chưa có các cột này, câu lệnh SQL sẽ thất bại ngay khi chạy.

Thứ tự triển khai phù hợp là:

``` text
1. Mở rộng cấu trúc customers
        ↓
2. Tạo chức năng đăng ký
        ↓
3. Tạo chức năng đăng nhập
        ↓
4. Hiển thị trạng thái đăng nhập trên Navbar
        ↓
5. Tạo chức năng đăng xuất
        ↓
6. Tích hợp tài khoản với Checkout
        ↓
7. Kiểm thử cả khách đăng nhập và khách vãng lai
```

Đây cũng là thứ tự được sử dụng trong Hands-on này.

------------------------------------------------------------------------

## 3. Mở rộng bảng `customers`

### 3.1. Cập nhật `database/schema.sql`

Mở:

``` text
database/schema.sql
```

Tìm phần khai báo bảng:

``` sql
CREATE TABLE customers (
```

Trong bảng này, tìm đoạn:

``` sql
Country VARCHAR(100),
Phone VARCHAR(20)
```

Thay bằng:

``` sql
Country VARCHAR(100),
Phone VARCHAR(20),
Email VARCHAR(255) NULL,
PasswordHash VARCHAR(255) NULL,
CONSTRAINT uq_customers_email UNIQUE (Email)
```

Sau thay đổi, phần khai báo bảng `customers` có dạng:

``` sql
CREATE TABLE customers (
    CustomerID INT AUTO_INCREMENT PRIMARY KEY,
    CustomerName VARCHAR(100) NOT NULL,
    ContactName VARCHAR(100),
    Address VARCHAR(200),
    City VARCHAR(100),
    PostalCode VARCHAR(20),
    Country VARCHAR(100),
    Phone VARCHAR(20),
    Email VARCHAR(255) NULL,
    PasswordHash VARCHAR(255) NULL,
    CONSTRAINT uq_customers_email UNIQUE (Email)
) CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

`Email` và `PasswordHash` được phép `NULL` vì hệ thống vẫn hỗ trợ khách
vãng lai.

Ràng buộc:

``` sql
UNIQUE (Email)
```

đảm bảo hai tài khoản không thể đăng ký cùng một email. MySQL vẫn cho
phép nhiều mẩu tin có `Email = NULL`, do đó không ảnh hưởng đến khách
vãng lai.

### 3.2. Cập nhật cơ sở dữ liệu đang sử dụng

Việc sửa `schema.sql` chỉ có tác dụng khi tạo database mới. Database
hiện đang chạy đã được tạo từ các Hands-on trước, vì vậy cần cập nhật
cấu trúc hiện tại.

Truy cập MySQL:

``` cmd
docker compose exec db mysql -u root -p
```

Chọn database:

``` sql
USE ql_banhang;
```

Thực hiện:

``` sql
ALTER TABLE customers
ADD COLUMN Email VARCHAR(255) NULL AFTER Phone,
ADD COLUMN PasswordHash VARCHAR(255) NULL AFTER Email;
```

Sau đó thêm ràng buộc:

``` sql
ALTER TABLE customers
ADD CONSTRAINT uq_customers_email UNIQUE (Email);
```

Kiểm tra:

``` sql
DESCRIBE customers;
```

Cần thấy:

``` text
Email         varchar(255)   YES   UNI
PasswordHash  varchar(255)   YES
```

> `schema.sql` mô tả cấu trúc dùng khi dựng hệ thống mới, còn
> `ALTER TABLE` cập nhật database đang chạy. Vì vậy cần thực hiện **cả
> hai**.

------------------------------------------------------------------------

## 4. Xây dựng chức năng đăng ký

Tạo tập tin mới:

``` text
public/register.php
```

Nhập toàn bộ nội dung:

``` php
<?php

require_once '/var/www/src/config/session.php';
require_once '/var/www/src/config/database.php';

$pageTitle = 'Đăng ký tài khoản';

$customerName = '';
$phone = '';
$email = '';
$errorMessage = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    $customerName =
        trim($_POST['customer_name'] ?? '');

    $phone =
        trim($_POST['phone'] ?? '');

    $email =
        trim($_POST['email'] ?? '');

    $password =
        $_POST['password'] ?? '';

    $confirmPassword =
        $_POST['confirm_password'] ?? '';

    if ($customerName === ''
        || $email === ''
        || $password === ''
        || $confirmPassword === '') {

        $errorMessage =
            'Vui lòng nhập đầy đủ thông tin bắt buộc.';

    } elseif (!filter_var(
        $email,
        FILTER_VALIDATE_EMAIL
    )) {

        $errorMessage =
            'Email không hợp lệ.';

    } elseif (strlen($password) < 6) {

        $errorMessage =
            'Mật khẩu phải có ít nhất 6 ký tự.';

    } elseif ($password !== $confirmPassword) {

        $errorMessage =
            'Mật khẩu xác nhận không khớp.';

    } else {

        $sqlCheck = "
            SELECT CustomerID
            FROM customers
            WHERE Email = ?
        ";

        $stmtCheck =
            $conn->prepare($sqlCheck);

        $stmtCheck->bind_param(
            's',
            $email
        );

        $stmtCheck->execute();

        $resultCheck =
            $stmtCheck->get_result();

        $existingCustomer =
            $resultCheck->fetch_assoc();

        $resultCheck->free();
        $stmtCheck->close();

        if ($existingCustomer) {

            $errorMessage =
                'Email này đã được sử dụng.';

        } else {

            $passwordHash =
                password_hash(
                    $password,
                    PASSWORD_DEFAULT
                );

            $sqlInsert = "
                INSERT INTO customers (
                    CustomerName,
                    Phone,
                    Email,
                    PasswordHash
                )
                VALUES (?, ?, ?, ?)
            ";

            $stmtInsert =
                $conn->prepare($sqlInsert);

            $stmtInsert->bind_param(
                'ssss',
                $customerName,
                $phone,
                $email,
                $passwordHash
            );

            $stmtInsert->execute();
            $stmtInsert->close();

            header(
                'Location: /login.php?registered=1'
            );
            exit;
        }
    }
}

require_once '/var/www/src/includes/frontend/header.php';
require_once '/var/www/src/includes/frontend/navbar.php';

?>

<div class="container py-4">

    <div class="row justify-content-center">

        <div class="col-md-7 col-lg-5">

            <h1 class="h3 mb-4">
                Đăng ký tài khoản
            </h1>

            <?php if ($errorMessage !== ''): ?>

                <div class="alert alert-danger">
                    <?= htmlspecialchars($errorMessage) ?>
                </div>

            <?php endif; ?>

            <form method="post">

                <div class="mb-3">
                    <label
                        for="customer_name"
                        class="form-label"
                    >
                        Họ và tên
                    </label>

                    <input
                        type="text"
                        class="form-control"
                        id="customer_name"
                        name="customer_name"
                        value="<?= htmlspecialchars(
                            $customerName
                        ) ?>"
                        required
                    >
                </div>

                <div class="mb-3">
                    <label
                        for="phone"
                        class="form-label"
                    >
                        Số điện thoại
                    </label>

                    <input
                        type="text"
                        class="form-control"
                        id="phone"
                        name="phone"
                        value="<?= htmlspecialchars($phone) ?>"
                    >
                </div>

                <div class="mb-3">
                    <label
                        for="email"
                        class="form-label"
                    >
                        Email
                    </label>

                    <input
                        type="email"
                        class="form-control"
                        id="email"
                        name="email"
                        value="<?= htmlspecialchars($email) ?>"
                        required
                    >
                </div>

                <div class="mb-3">
                    <label
                        for="password"
                        class="form-label"
                    >
                        Mật khẩu
                    </label>

                    <input
                        type="password"
                        class="form-control"
                        id="password"
                        name="password"
                        required
                    >
                </div>

                <div class="mb-3">
                    <label
                        for="confirm_password"
                        class="form-label"
                    >
                        Xác nhận mật khẩu
                    </label>

                    <input
                        type="password"
                        class="form-control"
                        id="confirm_password"
                        name="confirm_password"
                        required
                    >
                </div>

                <button
                    type="submit"
                    class="btn btn-primary"
                >
                    Đăng ký
                </button>

            </form>

        </div>

    </div>

</div>

<?php

require_once '/var/www/src/includes/frontend/footer.php';

$conn->close();
```

### 4.1. Vì sao không lưu mật khẩu trực tiếp?

Không thực hiện:

``` php
PasswordHash = $password;
```

Thay vào đó:

``` php
$passwordHash =
    password_hash(
        $password,
        PASSWORD_DEFAULT
    );
```

Database chỉ lưu giá trị băm. Mật khẩu gốc không được lưu trực tiếp.

### 4.2. Kiểm thử đăng ký

Truy cập:

``` text
http://localhost:8080/register.php
```

Tạo một tài khoản thử nghiệm.

Sau đó kiểm tra:

``` sql
SELECT
    CustomerID,
    CustomerName,
    Phone,
    Email,
    PasswordHash
FROM customers
ORDER BY CustomerID DESC
LIMIT 1;
```

`PasswordHash` phải là chuỗi băm, không phải mật khẩu đã nhập.

Thử đăng ký lại cùng email. Giao diện phải báo:

``` text
Email này đã được sử dụng.
```

------------------------------------------------------------------------

## 5. Xây dựng chức năng đăng nhập

Tạo:

``` text
public/login.php
```

Nhập nội dung:

``` php
<?php

require_once '/var/www/src/config/session.php';
require_once '/var/www/src/config/database.php';

$pageTitle = 'Đăng nhập';

if (isset($_SESSION['customer_id'])) {
    header('Location: /');
    exit;
}

$email = '';
$errorMessage = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    $email =
        trim($_POST['email'] ?? '');

    $password =
        $_POST['password'] ?? '';

    if ($email === '' || $password === '') {

        $errorMessage =
            'Vui lòng nhập email và mật khẩu.';

    } elseif (!filter_var(
        $email,
        FILTER_VALIDATE_EMAIL
    )) {

        $errorMessage =
            'Email không hợp lệ.';

    } else {

        $sql = "
            SELECT
                CustomerID,
                CustomerName,
                Email,
                PasswordHash
            FROM customers
            WHERE Email = ?
        ";

        $stmt =
            $conn->prepare($sql);

        $stmt->bind_param(
            's',
            $email
        );

        $stmt->execute();

        $result =
            $stmt->get_result();

        $customer =
            $result->fetch_assoc();

        $result->free();
        $stmt->close();

        if ($customer
            && !empty($customer['PasswordHash'])
            && password_verify(
                $password,
                $customer['PasswordHash']
            )) {

            session_regenerate_id(true);

            $_SESSION['customer_id'] =
                (int) $customer['CustomerID'];

            $_SESSION['customer_name'] =
                $customer['CustomerName'];

            header('Location: /');
            exit;

        } else {

            $errorMessage =
                'Email hoặc mật khẩu không đúng.';
        }
    }
}

require_once '/var/www/src/includes/frontend/header.php';
require_once '/var/www/src/includes/frontend/navbar.php';

?>

<div class="container py-4">

    <div class="row justify-content-center">

        <div class="col-md-7 col-lg-5">

            <h1 class="h3 mb-4">
                Đăng nhập
            </h1>

            <?php if (isset($_GET['registered'])): ?>

                <div class="alert alert-success">
                    Đăng ký thành công.
                    Bạn có thể đăng nhập.
                </div>

            <?php endif; ?>

            <?php if ($errorMessage !== ''): ?>

                <div class="alert alert-danger">
                    <?= htmlspecialchars($errorMessage) ?>
                </div>

            <?php endif; ?>

            <form method="post">

                <div class="mb-3">

                    <label
                        for="email"
                        class="form-label"
                    >
                        Email
                    </label>

                    <input
                        type="email"
                        class="form-control"
                        id="email"
                        name="email"
                        value="<?= htmlspecialchars($email) ?>"
                        required
                    >

                </div>

                <div class="mb-3">

                    <label
                        for="password"
                        class="form-label"
                    >
                        Mật khẩu
                    </label>

                    <input
                        type="password"
                        class="form-control"
                        id="password"
                        name="password"
                        required
                    >

                </div>

                <button
                    type="submit"
                    class="btn btn-primary"
                >
                    Đăng nhập
                </button>

            </form>

        </div>

    </div>

</div>

<?php

require_once '/var/www/src/includes/frontend/footer.php';

$conn->close();
```

Khi đăng nhập, hệ thống không so sánh hai chuỗi mật khẩu trực tiếp mà sử
dụng:

``` php
password_verify(
    $password,
    $customer['PasswordHash']
)
```

Sau khi xác thực thành công:

``` php
session_regenerate_id(true);
```

được thực hiện trước khi lưu thông tin nhận diện khách hàng vào Session:

``` php
$_SESSION['customer_id']
$_SESSION['customer_name']
```

Session chỉ giữ thông tin cần thiết để nhận diện người dùng. Các thông
tin như `Phone`, `Email`, `Address` không cần sao chép vào Session vì có
thể đọc lại từ database theo `CustomerID`.

------------------------------------------------------------------------

## 6. Thay đổi Navbar theo trạng thái đăng nhập

Mở:

``` text
src/includes/frontend/navbar.php
```

Ngay sau phần tính số lượng sản phẩm trong giỏ:

``` php
$cartCount = array_sum($_SESSION['cart'] ?? []);
```

bổ sung:

``` php
$isLoggedIn = isset($_SESSION['customer_id']);

$customerName =
    $_SESSION['customer_name'] ?? '';
```

Tại khu vực bên phải Navbar, giữ nút **Giỏ hàng** và **Quản trị**, đồng
thời bổ sung phần hiển thị theo trạng thái:

``` php
<?php if ($isLoggedIn): ?>

    <span class="text-light">
        <?= htmlspecialchars($customerName) ?>
    </span>

    <a
        class="btn btn-outline-light btn-sm"
        href="/logout.php"
    >
        Đăng xuất
    </a>

<?php else: ?>

    <a
        class="btn btn-outline-light btn-sm"
        href="/register.php"
    >
        Đăng ký
    </a>

    <a
        class="btn btn-outline-light btn-sm"
        href="/login.php"
    >
        Đăng nhập
    </a>

<?php endif; ?>
```

Kết quả mong đợi:

``` text
Chưa đăng nhập
Giỏ hàng | Đăng ký | Đăng nhập | Quản trị

Đã đăng nhập
Giỏ hàng | Tên khách hàng | Đăng xuất | Quản trị
```

------------------------------------------------------------------------

## 7. Xây dựng chức năng đăng xuất

Tạo:

``` text
public/logout.php
```

Nhập toàn bộ nội dung:

``` php
<?php

require_once '/var/www/src/config/session.php';

/*
 * Chỉ xóa dữ liệu xác thực khách hàng.
 * Session còn được sử dụng cho giỏ hàng.
 */
unset(
    $_SESSION['customer_id'],
    $_SESSION['customer_name']
);

session_regenerate_id(true);

header('Location: /');
exit;
```

### 7.1. Vì sao không dùng `session_destroy()`?

Ứng dụng hiện sử dụng cùng Session cho:

``` text
Thông tin đăng nhập
+
Giỏ hàng
```

Nếu gọi:

``` php
session_destroy();
```

toàn bộ Session có thể bị hủy, trong đó có dữ liệu:

``` php
$_SESSION['cart']
```

Trong Hands-on này, đăng xuất chỉ có nghĩa là bỏ trạng thái xác thực:

``` php
unset(
    $_SESSION['customer_id'],
    $_SESSION['customer_name']
);
```

Nhờ đó khách có thể đăng xuất nhưng giỏ hàng vẫn được giữ lại.

------------------------------------------------------------------------

## 8. Tích hợp tài khoản khách hàng với Checkout

Ở Hands-on 12, `checkout.php` luôn tạo một customer mới trước khi tạo
đơn hàng.

Sau khi có tài khoản, quy tắc cần thay đổi:

``` text
CHECKOUT
   │
   ├── Chưa đăng nhập
   │      └── tạo customer mới
   │
   └── Đã đăng nhập
          └── sử dụng CustomerID hiện tại
```

Mở:

``` text
public/checkout.php
```

### 8.1. Xác định trạng thái đăng nhập

Sau phần lấy giỏ hàng, bổ sung:

``` php
$isLoggedIn = isset($_SESSION['customer_id']);

$customerID = null;
$customerName = '';
$email = '';
$phone = '';
$address = '';
$errorMessage = '';
```

Nếu trong tập tin hiện có các biến `$customerName`, `$phone`,
`$address`, `$errorMessage` được khởi tạo riêng, thay phần khởi tạo cũ
bằng đoạn trên để tránh khai báo lặp.

### 8.2. Đọc thông tin tài khoản từ database

Ngay sau phần khởi tạo biến, thêm:

``` php
if ($isLoggedIn) {

    $customerID =
        (int) $_SESSION['customer_id'];

    $sqlCustomerAccount = "
        SELECT
            CustomerID,
            CustomerName,
            Email,
            Phone,
            Address
        FROM customers
        WHERE CustomerID = ?
          AND Email IS NOT NULL
    ";

    $stmtCustomerAccount =
        $conn->prepare($sqlCustomerAccount);

    $stmtCustomerAccount->bind_param(
        'i',
        $customerID
    );

    $stmtCustomerAccount->execute();

    $customerResult =
        $stmtCustomerAccount->get_result();

    $customer =
        $customerResult->fetch_assoc();

    $customerResult->free();
    $stmtCustomerAccount->close();

    if (!$customer) {

        unset(
            $_SESSION['customer_id'],
            $_SESSION['customer_name']
        );

        header('Location: /login.php');
        exit;
    }

    $customerName =
        $customer['CustomerName'];

    $email =
        $customer['Email'] ?? '';

    $phone =
        $customer['Phone'] ?? '';

    $address =
        $customer['Address'] ?? '';
}
```

Điểm quan trọng là Session chỉ cung cấp:

``` php
$_SESSION['customer_id']
```

Sau đó hệ thống truy vấn database để lấy thông tin hiện tại của khách
hàng. Nhờ vậy, Checkout không phụ thuộc vào một bản sao `Phone`, `Email`
hoặc `Address` đã cũ trong Session.

### 8.3. Nhận dữ liệu POST theo hai trường hợp

Trong phần xử lý:

``` php
if ($_SERVER['REQUEST_METHOD'] === 'POST'
    && isset($_POST['place_order'])) {
```

thay phần lấy `$customerName`, `$phone`, `$address` bằng:

``` php
if ($isLoggedIn) {

    $phone =
        trim($_POST['phone'] ?? '');

    $address =
        trim($_POST['address'] ?? '');

} else {

    $customerName =
        trim($_POST['customer_name'] ?? '');

    $phone =
        trim($_POST['phone'] ?? '');

    $address =
        trim($_POST['address'] ?? '');
}
```

Khách đã đăng nhập không gửi lại tên tài khoản để server xác định danh
tính. Danh tính được lấy từ `CustomerID` trong Session.

### 8.4. Thay bước luôn tạo customer mới

Trong transaction, tìm phần đang có chú thích tương tự:

``` php
/*
 * 2. Tạo khách hàng.
 */
```

và đoạn `INSERT INTO customers`.

Thay toàn bộ bước này bằng:

``` php
/*
 * 2. Xác định khách hàng.
 */
if ($isLoggedIn) {

    $sqlAccount = "
        SELECT CustomerID
        FROM customers
        WHERE CustomerID = ?
          AND Email IS NOT NULL
        FOR UPDATE
    ";

    $stmtAccount =
        $conn->prepare($sqlAccount);

    $stmtAccount->bind_param(
        'i',
        $customerID
    );

    $stmtAccount->execute();

    $accountResult =
        $stmtAccount->get_result();

    $account =
        $accountResult->fetch_assoc();

    $accountResult->free();
    $stmtAccount->close();

    if (!$account) {
        throw new Exception(
            'Tài khoản khách hàng không còn hợp lệ.'
        );
    }

    $sqlUpdateCustomer = "
        UPDATE customers
        SET
            Phone = ?,
            Address = ?
        WHERE CustomerID = ?
    ";

    $stmtUpdateCustomer =
        $conn->prepare($sqlUpdateCustomer);

    $stmtUpdateCustomer->bind_param(
        'ssi',
        $phone,
        $address,
        $customerID
    );

    $stmtUpdateCustomer->execute();
    $stmtUpdateCustomer->close();

} else {

    $sqlCustomer = "
        INSERT INTO customers (
            CustomerName,
            Address,
            Phone
        )
        VALUES (?, ?, ?)
    ";

    $stmtCustomer =
        $conn->prepare($sqlCustomer);

    $stmtCustomer->bind_param(
        'sss',
        $customerName,
        $address,
        $phone
    );

    $stmtCustomer->execute();

    $customerID =
        $conn->insert_id;

    $stmtCustomer->close();
}
```

Ở nhánh đã đăng nhập, `CustomerID` được khóa và kiểm tra lại trong
transaction trước khi tạo đơn.

Ở nhánh khách vãng lai, cách xử lý của Hands-on 12 được giữ nguyên.

### 8.5. Điều chỉnh form Checkout

Ở vị trí trường **Họ và tên**, thay bằng cấu trúc điều kiện:

``` php
<?php if ($isLoggedIn): ?>

    <div class="mb-3">

        <label class="form-label">
            Họ và tên
        </label>

        <input
            type="text"
            class="form-control"
            value="<?= htmlspecialchars(
                $customerName
            ) ?>"
            readonly
        >

    </div>

    <div class="mb-3">

        <label class="form-label">
            Email
        </label>

        <input
            type="email"
            class="form-control"
            value="<?= htmlspecialchars(
                $email
            ) ?>"
            readonly
        >

    </div>

<?php else: ?>

    <div class="mb-3">

        <label
            for="customer_name"
            class="form-label"
        >
            Họ và tên
        </label>

        <input
            type="text"
            class="form-control"
            id="customer_name"
            name="customer_name"
            value="<?= htmlspecialchars(
                $customerName
            ) ?>"
            required
        >

    </div>

<?php endif; ?>
```

Giữ các trường `phone` và `address` hiện có.

Khi khách đã đăng nhập:

``` text
Họ và tên       readonly
Email           readonly
Số điện thoại   có thể chỉnh
Địa chỉ         có thể chỉnh
```

Khi khách vãng lai:

``` text
Họ và tên       nhập trực tiếp
Số điện thoại   nhập trực tiếp
Địa chỉ         nhập trực tiếp
```

Khi khách có tài khoản thay đổi số điện thoại hoặc địa chỉ trong
Checkout, thông tin này được cập nhật vào `customers`. Lần Checkout tiếp
theo, dữ liệu mới sẽ được tự động đọc lại.

------------------------------------------------------------------------

## 9. Kiểm tra cú pháp PHP

Kiểm tra các tập tin PHP mới hoặc đã thay đổi:

``` cmd
docker compose exec web php -l /var/www/html/register.php
docker compose exec web php -l /var/www/html/login.php
docker compose exec web php -l /var/www/html/logout.php
docker compose exec web php -l /var/www/html/checkout.php
docker compose exec web php -l /var/www/src/includes/frontend/navbar.php
```

Mỗi lệnh cần trả về:

``` text
No syntax errors detected
```

------------------------------------------------------------------------

## 10. Kiểm thử chức năng

### 10.1. Kiểm thử đăng ký

Mở:

``` text
http://localhost:8080/register.php
```

Đăng ký một tài khoản mới.

Kiểm tra trong MySQL:

``` sql
SELECT
    CustomerID,
    CustomerName,
    Phone,
    Email,
    PasswordHash
FROM customers
ORDER BY CustomerID DESC
LIMIT 1;
```

Xác nhận:

-   có `CustomerID`;
-   email được lưu;
-   `PasswordHash` không phải mật khẩu gốc.

Thử đăng ký lại cùng email. Hệ thống phải từ chối.

### 10.2. Kiểm thử đăng nhập

Thử đăng nhập bằng mật khẩu sai.

Hệ thống phải hiển thị:

``` text
Email hoặc mật khẩu không đúng.
```

Sau đó đăng nhập bằng mật khẩu đúng.

Navbar phải thay đổi từ:

``` text
Đăng ký | Đăng nhập
```

sang:

``` text
Tên khách hàng | Đăng xuất
```

### 10.3. Kiểm thử Checkout khi đã đăng nhập

Thêm sản phẩm vào giỏ và mở Checkout.

Form phải tự động hiển thị:

``` text
Họ và tên
Email
Số điện thoại
Địa chỉ
```

Tên và email ở chế độ chỉ đọc. Số điện thoại và địa chỉ có thể chỉnh.

Đặt hàng và ghi nhớ `CustomerID` của tài khoản.

Kiểm tra đơn mới:

``` sql
SELECT
    OrderID,
    CustomerID,
    TotalAmount,
    Status
FROM orders
ORDER BY OrderID DESC
LIMIT 1;
```

`CustomerID` của đơn phải bằng `CustomerID` của tài khoản đang đăng
nhập.

Kiểm tra lại khách hàng:

``` sql
SELECT
    CustomerID,
    CustomerName,
    Phone,
    Email,
    Address
FROM customers
WHERE CustomerID = <CustomerID>;
```

Thay `<CustomerID>` bằng mã tài khoản thực tế.

Nếu vừa cập nhật Phone hoặc Address khi Checkout, dữ liệu mới phải được
lưu.

Tiếp tục thêm sản phẩm và mở Checkout lần nữa. Phone và Address vừa lưu
phải tự động xuất hiện.

### 10.4. Kiểm thử đăng xuất và giỏ hàng

Thêm sản phẩm vào giỏ rồi đăng xuất.

Sau khi đăng xuất:

-   Navbar trở lại trạng thái `Đăng ký | Đăng nhập`;
-   tên khách hàng không còn hiển thị;
-   giỏ hàng vẫn còn sản phẩm.

Điều này xác nhận `logout.php` chỉ xóa dữ liệu xác thực, không hủy toàn
bộ Session.

### 10.5. Kiểm thử khách vãng lai

Trong trạng thái chưa đăng nhập:

1.  thêm sản phẩm vào giỏ;
2.  mở Checkout;
3.  nhập họ tên, số điện thoại và địa chỉ;
4.  xác nhận đặt hàng.

Kiểm tra customer mới:

``` sql
SELECT
    CustomerID,
    CustomerName,
    Phone,
    Email,
    Address
FROM customers
ORDER BY CustomerID DESC
LIMIT 1;
```

Khách vãng lai phải có:

``` text
CustomerID = mã mới
Email      = NULL
```

Kiểm tra đơn hàng:

``` sql
SELECT
    OrderID,
    CustomerID,
    TotalAmount,
    Status
FROM orders
ORDER BY OrderID DESC
LIMIT 1;
```

`CustomerID` của đơn phải trỏ đến customer vừa được tạo.

------------------------------------------------------------------------

## 11. Luồng hoạt động sau Hands-on 14

Sau Hands-on này, ứng dụng hỗ trợ đồng thời hai cách mua hàng.

### Khách vãng lai

``` text
Sản phẩm
   ↓
Giỏ hàng
   ↓
Checkout
   ↓
Nhập thông tin khách hàng
   ↓
Tạo customer mới
   ↓
Tạo order
```

### Khách có tài khoản

``` text
Đăng ký
   ↓
Đăng nhập
   ↓
Session
customer_id
   ↓
Sản phẩm
   ↓
Giỏ hàng
   ↓
Checkout
   ↓
Đọc customers theo CustomerID
   ↓
Cập nhật Phone / Address nếu cần
   ↓
Tạo order với cùng CustomerID
```

Điểm khác biệt quan trọng là một khách đã đăng nhập có thể có nhiều đơn
hàng cùng tham chiếu đến một `CustomerID`:

``` text
CustomerID = 9
     │
     ├── Order A
     ├── Order B
     └── Order C
```

Trong khi khách vãng lai vẫn có thể đặt hàng mà không bắt buộc tạo tài
khoản.

------------------------------------------------------------------------

## 12. Kiểm tra thay đổi với Git

Kiểm tra:

``` cmd
git status
```

Các tập tin của Hands-on này gồm:

``` text
database/schema.sql
public/checkout.php
public/login.php
public/logout.php
public/register.php
src/includes/frontend/navbar.php
```

Kiểm tra lỗi whitespace:

``` cmd
git diff --check
```

Nếu lệnh không có output thì không phát hiện lỗi whitespace cần xử lý.

Xem thống kê:

``` cmd
git diff --stat
```

Lưu ý: các tập tin mới chưa được Git theo dõi có thể chưa xuất hiện
trong `git diff --stat` cho đến khi được stage.

Stage đúng các tập tin:

``` cmd
git add database/schema.sql public/checkout.php public/login.php public/logout.php public/register.php src/includes/frontend/navbar.php
```

Kiểm tra lại:

``` cmd
git status
```

Commit:

``` cmd
git commit -m "Add customer authentication and account checkout"
```

Push:

``` cmd
git push origin main
```

Cuối cùng:

``` cmd
git status
```

Kết quả mong đợi:

``` text
nothing to commit, working tree clean
```

------------------------------------------------------------------------

## 13. Kết quả đạt được

Sau Hands-on 14, hệ thống đã có thêm lớp chức năng tài khoản khách hàng:

``` text
Frontend
├── Đăng ký
├── Đăng nhập
├── Đăng xuất
├── Session xác thực
├── Giỏ hàng
└── Checkout
      ├── khách vãng lai
      └── khách đã đăng nhập

Database
└── customers
      ├── thông tin khách hàng
      ├── Email
      └── PasswordHash
```

Các nguyên tắc quan trọng đã được áp dụng:

-   mật khẩu không được lưu trực tiếp;
-   email tài khoản là duy nhất;
-   Prepared Statement được sử dụng khi truy vấn dữ liệu người dùng;
-   Session chỉ lưu định danh cần thiết;
-   thông tin khách hàng được đọc lại từ database khi cần;
-   đăng xuất không làm mất giỏ hàng;
-   khách đã đăng nhập tái sử dụng `CustomerID`;
-   khách vãng lai vẫn có thể mua hàng;
-   Checkout tiếp tục sử dụng transaction và quy trình kiểm tra tồn kho
    đã xây dựng ở Hands-on 12.

> **Checkpoint:** hệ thống hiện đã hoàn chỉnh luồng mua hàng cơ bản từ
> xem sản phẩm, giỏ hàng, Checkout, tài khoản khách hàng đến quản lý đơn
> hàng. Hands-on tiếp theo sẽ chuyển sang khai thác dữ liệu ứng dụng
> thông qua **REST API và `fetch()`**.
