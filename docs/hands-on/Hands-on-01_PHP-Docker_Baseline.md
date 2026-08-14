# HANDS-ON 01 -- KHỞI TẠO DỰ ÁN PHP VỚI GIT/GITHUB VÀ DOCKER

## 1. Mục tiêu

Sau phần thực hành này, người học có thể:

-   Tạo repository trên GitHub và clone về máy.
-   Tạo cấu trúc thư mục ban đầu cho dự án PHP.
-   Tạo trang PHP đầu tiên.
-   Cấu hình PHP 8.3 và Apache bằng Docker.
-   Kích hoạt extension MySQLi trong PHP container.
-   Chạy ứng dụng bằng Docker Compose.
-   Kiểm tra PHP, MySQLi và ứng dụng trên trình duyệt.
-   Lưu phiên bản đầu tiên của dự án bằng Git và push lên GitHub.

> **Môi trường sử dụng trong hướng dẫn:** Windows Command Prompt (CMD),
> Git, Docker Desktop và trình duyệt Web.

------------------------------------------------------------------------

## 2. Tạo repository trên GitHub

Tạo repository mới với thông tin:

``` text
Repository name:
php-mysql-sales-template

Description:
Hands-on project for Open Source Web Application Development

Visibility:
Public
```

Ở thời điểm này:

-   Không chọn **Add a README file**.
-   Không thêm `.gitignore`.
-   Không chọn license.
-   Chưa bật **Template repository**.

Repository được tạo ở trạng thái rỗng để có thể kiểm soát toàn bộ cấu
trúc dự án từ đầu.

------------------------------------------------------------------------

## 3. Clone repository về máy

Tại CMD, chuyển đến thư mục dùng để lưu dự án và chạy:

``` cmd
git clone https://github.com/<username>/php-mysql-sales-template.git
cd php-mysql-sales-template
```

Kiểm tra:

``` cmd
git status
git remote -v
```

Kết quả mong đợi:

``` text
On branch main

No commits yet

nothing to commit
```

`origin` phải trỏ đến repository vừa tạo trên GitHub.

------------------------------------------------------------------------

## 4. Tạo cấu trúc ban đầu của dự án

### 4.1. Tạo thư mục

Trong CMD:

``` cmd
mkdir public
mkdir src
mkdir src\config
mkdir src\includes
mkdir database
mkdir docker
mkdir docker\php
```

### 4.2. Tạo các file ban đầu

``` cmd
type nul > public\index.php
type nul > .gitignore
type nul > .env.example
type nul > compose.yaml
type nul > README.md
```

Cấu trúc ban đầu:

``` text
php-mysql-sales-template/
│
├── database/
├── docker/
│   └── php/
├── public/
│   └── index.php
├── src/
│   ├── config/
│   └── includes/
├── .env.example
├── .gitignore
├── compose.yaml
└── README.md
```

Kiểm tra bằng:

``` cmd
tree /F
git status
```

### Lưu ý về Git

Git không trực tiếp theo dõi thư mục rỗng. Vì vậy các thư mục như
`database`, `src\config`, `src\includes` hoặc `docker\php` có thể chưa
xuất hiện trong `git status` nếu chưa chứa file.

------------------------------------------------------------------------

## 5. Tạo trang PHP đầu tiên

Mở file:

``` text
public\index.php
```

và nhập:

``` php
<?php
$appName = "Hệ thống quản lý bán hàng";
?>

<!doctype html>
<html lang="vi">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title><?= $appName ?></title>
</head>
<body>

    <h1><?= $appName ?></h1>
    <p>Ứng dụng PHP đang hoạt động.</p>

</body>
</html>
```

Ở giai đoạn này chưa sử dụng Bootstrap. Mục tiêu là kiểm tra riêng
luồng:

``` text
Docker
   ↓
Apache
   ↓
PHP
   ↓
index.php
   ↓
Browser
```

------------------------------------------------------------------------

## 6. Tạo Dockerfile cho PHP và Apache

Tạo file:

``` cmd
type nul > docker\php\Dockerfile
```

Mở `docker\php\Dockerfile` và nhập:

``` dockerfile
FROM php:8.3-apache

RUN docker-php-ext-install mysqli

WORKDIR /var/www/html
```

Dòng:

``` dockerfile
RUN docker-php-ext-install mysqli
```

cài extension **MySQLi** để PHP có thể làm việc với MySQL trong các phần
thực hành tiếp theo.

Kiểm tra lại cấu trúc:

``` cmd
tree /F
```

------------------------------------------------------------------------

## 7. Cấu hình Docker Compose

Mở file:

``` text
compose.yaml
```

và nhập:

``` yaml
services:
  web:
    build:
      context: .
      dockerfile: docker/php/Dockerfile
    ports:
      - "8080:80"
    volumes:
      - ./public:/var/www/html
```

Cấu hình này tạo service `web`:

``` text
web
 ├── build từ docker/php/Dockerfile
 ├── port 8080 của máy → port 80 của container
 └── public → /var/www/html
```

------------------------------------------------------------------------

## 8. Khởi động ứng dụng bằng Docker

### 8.1. Kiểm tra Docker Desktop

Trước khi chạy Docker Compose, cần bảo đảm **Docker Desktop đã được khởi
động và Docker Engine đang hoạt động**.

Có thể kiểm tra:

``` cmd
docker version
```

Kết quả bình thường phải có cả phần **Client** và **Server**.

### 8.2. Build và chạy container

``` cmd
docker compose up -d --build
```

Kiểm tra:

``` cmd
docker compose ps
```

Service `web` phải ở trạng thái `Up` và có ánh xạ cổng tương tự:

``` text
0.0.0.0:8080->80/tcp
```

### 8.3. Kiểm tra PHP

``` cmd
docker compose exec web php -v
```

Trong lần kiểm thử dự án, môi trường đã chạy thành công với PHP 8.3.

### 8.4. Kiểm tra MySQLi

``` cmd
docker compose exec web php -m | findstr mysqli
```

Kết quả mong đợi:

``` text
mysqli
```

### 8.5. Kiểm tra trên trình duyệt

Truy cập:

``` text
http://localhost:8080
```

Trang Web phải hiển thị:

``` text
Hệ thống quản lý bán hàng

Ứng dụng PHP đang hoạt động.
```

Như vậy đã kiểm chứng được chuỗi hoạt động:

``` text
Windows
   ↓
Docker Desktop
   ↓
Docker Compose
   ↓
PHP 8.3 + Apache
   ↓
MySQLi
   ↓
public/index.php
   ↓
Browser
```

------------------------------------------------------------------------

## 9. Lỗi thường gặp: Docker Desktop chưa được bật

Nếu chạy:

``` cmd
docker compose up -d --build
```

và xuất hiện lỗi tương tự:

``` text
open //./pipe/dockerDesktopLinuxEngine:
The system cannot find the file specified.
```

trong khi lệnh `docker` vẫn tồn tại, một nguyên nhân thường gặp là
**Docker Desktop chưa được khởi động**.

Cách kiểm tra:

1.  Mở Docker Desktop.
2.  Chờ Docker Engine sẵn sàng.
3.  Chạy:

``` cmd
docker version
```

4.  Khi đã có cả `Client` và `Server`, chạy lại:

``` cmd
docker compose up -d --build
```

Không nên chỉnh `Dockerfile` hoặc `compose.yaml` ngay khi lỗi thực tế
xuất phát từ Docker Engine chưa chạy.

------------------------------------------------------------------------

## 10. Lưu checkpoint đầu tiên bằng Git

Kiểm tra:

``` cmd
git status
git diff
```

Stage toàn bộ file:

``` cmd
git add .
```

Kiểm tra lại:

``` cmd
git status
```

Commit:

``` cmd
git commit -m "Initialize PHP Docker development environment"
```

Push lên GitHub:

``` cmd
git push -u origin main
```

Kiểm tra cuối cùng:

``` cmd
git status
git log -1 --oneline
```

Kết quả mong đợi:

``` text
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

Checkpoint thử nghiệm thực tế đã tạo:

``` text
29a72ab Initialize PHP Docker development environment
```

------------------------------------------------------------------------

## 11. Kết quả đạt được

Sau phần thực hành này, dự án đã có baseline chạy được:

``` text
GitHub Repository
        ↓
Docker Compose
        ↓
PHP 8.3 + Apache
        ↓
MySQLi enabled
        ↓
public/index.php
        ↓
localhost:8080
```

Các thành phần MySQL, Bootstrap 5, REST API và GitHub Codespaces **chưa
được triển khai trong checkpoint này**. Chúng sẽ được bổ sung và kiểm
thử từng bước ở các giai đoạn tiếp theo.

------------------------------------------------------------------------

## 12. Trạng thái dự án tại checkpoint

``` text
php-mysql-sales-template/
│
├── database/
├── docker/
│   └── php/
│       └── Dockerfile
├── public/
│   └── index.php
├── src/
│   ├── config/
│   └── includes/
├── .env.example
├── .gitignore
├── compose.yaml
└── README.md
```

> **Checkpoint:** PHP/Apache chạy thành công bằng Docker Desktop local
> và extension MySQLi đã được xác nhận hoạt động.
