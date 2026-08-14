# MINI HANDS-ON -- LƯU VÀ ĐỒNG BỘ MÃ NGUỒN LÊN GITHUB

## 1. Mục tiêu

Sau khi hoàn thành nội dung này, người học có thể:

-   Phân biệt mã nguồn đang có trên máy, thay đổi đã được stage, commit
    cục bộ và commit đã được push lên GitHub.
-   Kiểm tra đúng repository và remote trước khi cập nhật.
-   Thực hiện quy trình Git cơ bản:
    `status → diff → add → commit → push`.
-   Kiểm tra lại trạng thái repository sau khi push.
-   Xử lý trường hợp repository chưa có commit nào và thực hiện cập nhật
    ở các buổi tiếp theo.

> Bối cảnh: dự án đã chạy thành công trên máy cá nhân nhưng `git status`
> cho thấy `No commits yet` và các file vẫn là `Untracked files`.

------------------------------------------------------------------------

## 2. Nhận biết trạng thái hiện tại

Ví dụ:

``` cmd
git status
```

Kết quả:

``` text
On branch main

No commits yet

Untracked files:
        .env.example
        .gitignore
        README.md
        compose.yaml
        docker/
        public/
```

Điều này có nghĩa:

-   Repository Git đã được khởi tạo.
-   Đang làm việc trên nhánh `main`.
-   Chưa có commit nào.
-   Các file đang tồn tại trên máy nhưng Git chưa theo dõi.
-   Mã nguồn chưa được lưu thành một phiên bản Git.
-   Chưa có commit để push lên GitHub.

Ghi nhớ:

``` text
ỨNG DỤNG CHẠY ĐƯỢC
        ≠
ĐÃ COMMIT VÀO GIT
        ≠
ĐÃ PUSH LÊN GITHUB
```

------------------------------------------------------------------------

## 3. Mô hình làm việc của Git

``` text
Working Directory
      │
      │ git add
      ▼
Staging Area
      │
      │ git commit
      ▼
Local Repository
      │
      │ git push
      ▼
GitHub Repository
```

  Khu vực             Ý nghĩa
  ------------------- ---------------------------------------------
  Working Directory   Các file và thay đổi đang có trên máy
  Staging Area        Các thay đổi đã chọn cho commit tiếp theo
  Local Repository    Lịch sử commit được lưu trên máy
  GitHub Repository   Repository từ xa nhận các commit qua `push`

------------------------------------------------------------------------

# 4. Phần A -- Push lần đầu lên GitHub

## Bước 1 -- Vào thư mục dự án

``` cmd
D:
cd D:\PTUDW-ST-2026\php-mysql-sales
git status
```

## Bước 2 -- Kiểm tra remote

``` cmd
git remote -v
```

Kết quả phải có dạng:

``` text
origin  https://github.com/<username>/<repository>.git (fetch)
origin  https://github.com/<username>/<repository>.git (push)
```

Kiểm tra:

-   `<username>` đúng tài khoản.
-   `<repository>` đúng dự án.
-   Không push nếu `origin` đang trỏ nhầm repository.

> **Nguyên tắc:** Trước lần push đầu tiên, phải biết mã nguồn sẽ được
> push đến repository nào.

## Bước 3 -- Stage các file

``` cmd
git add .
git status
```

Các file phải chuyển từ:

``` text
Untracked files:
```

sang:

``` text
Changes to be committed:
```

Ví dụ:

``` text
new file:   .env.example
new file:   .gitignore
new file:   README.md
new file:   compose.yaml
new file:   docker/php/Dockerfile
new file:   public/index.php
```

## Bước 4 -- Tạo commit đầu tiên

``` cmd
git commit -m "Initialize PHP Docker development environment"
```

Kiểm tra:

``` cmd
git log -1 --oneline
```

Kết quả có dạng:

``` text
<commit-id> Initialize PHP Docker development environment
```

## Bước 5 -- Push lần đầu

``` cmd
git push -u origin main
```

Tùy chọn `-u` thiết lập nhánh `main` cục bộ theo dõi `origin/main`.

Từ các lần sau, thông thường chỉ cần:

``` cmd
git push
```

## Bước 6 -- Kiểm tra sau khi push

``` cmd
git status
git log -1 --oneline
```

Mong đợi:

``` text
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

Mở repository trên GitHub và xác nhận:

-   Các file đã xuất hiện.
-   Commit vừa tạo đã xuất hiện.
-   Đúng repository và đúng nhánh.

### Checkpoint

``` text
[ ] origin đúng repository
[ ] git add thành công
[ ] commit được tạo
[ ] push thành công
[ ] GitHub có mã nguồn mới
[ ] working tree clean
```

------------------------------------------------------------------------

# 5. Phần B -- Bảng hướng dẫn cập nhật mã nguồn lên GitHub

Sau lần push đầu tiên, sử dụng bảng sau mỗi khi hoàn thành một phần công
việc:

  ------------------------------------------------------------------------------------------
                   Bước Mục đích         Lệnh                               Kết quả cần kiểm
                                                                            tra
  --------------------- ---------------- ---------------------------------- ----------------
                      1 Xem trạng thái   `git status`                       Biết file mới,
                                                                            sửa hoặc xóa

                      2 Xem nội dung     `git diff`                         Xác nhận thay
                        thay đổi                                            đổi đúng

                      3 Chọn thay đổi    `git add <file>` hoặc `git add .`  File được đưa
                        cần lưu                                             vào staging

                      4 Kiểm tra staging `git status`                       Chỉ stage các
                                                                            file mong muốn

                      5 Tạo phiên bản    `git commit -m "Mo ta thay doi"`   Có commit mới

                      6 Kiểm tra commit  `git log -1 --oneline`             Thấy commit vừa
                                                                            tạo

                      7 Đồng bộ GitHub   `git push`                         Commit được gửi
                                                                            lên remote

                      8 Kiểm tra cuối    `git status`                       Working tree
                                                                            sạch, nhánh đồng
                                                                            bộ

                      9 Xác nhận GitHub  Mở repository                      File và commit
                                                                            mới đã xuất hiện
  ------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 6. Quy trình cập nhật chuẩn

### Kiểm tra trước khi commit

``` cmd
git status
git diff
```

### Stage

Một file:

``` cmd
git add public\index.php
```

Hoặc toàn bộ thay đổi đã được kiểm tra:

``` cmd
git add .
```

Sau đó:

``` cmd
git status
```

### Commit

Ví dụ:

``` cmd
git commit -m "Add database connection"
```

Thông điệp commit nên ngắn gọn và mô tả đúng nội dung.

Ví dụ tốt:

``` text
Initialize PHP Docker development environment
Add MySQL database service
Add database connection
Add category listing
Add product management
Add REST API endpoints
```

Tránh:

``` text
update
fix
abc
test
123
```

### Push và kiểm tra

``` cmd
git push
git status
git log -1 --oneline
```

------------------------------------------------------------------------

## 7. Bảng tra cứu nhanh

  Tình huống                           Lệnh
  ------------------------------------ ----------------------------------
  Xem trạng thái                       `git status`
  Xem thay đổi chưa stage              `git diff`
  Xem remote                           `git remote -v`
  Stage một file                       `git add <ten-file>`
  Stage toàn bộ thay đổi đã kiểm tra   `git add .`
  Tạo commit                           `git commit -m "Mo ta thay doi"`
  Xem commit gần nhất                  `git log -1 --oneline`
  Push lần đầu nhánh `main`            `git push -u origin main`
  Push các lần tiếp theo               `git push`
  Lấy cập nhật từ remote               `git pull`

------------------------------------------------------------------------

## 8. Workflow đầu và cuối buổi thực hành

### Đầu buổi

``` cmd
cd <thu-muc-du-an>
git status
git pull
docker compose up -d
docker compose ps
```

### Trong buổi

``` text
Edit
 ↓
Run / Test
 ↓
git status
 ↓
git diff
```

### Cuối buổi

``` cmd
git status
git diff
git add .
git status
git commit -m "<mo-ta-thay-doi>"
git push
git status
```

Nếu kết thúc môi trường Docker:

``` cmd
docker compose down
```

------------------------------------------------------------------------

## 9. Một số trạng thái Git thường gặp

### `No commits yet`

Repository chưa có commit.

``` cmd
git add .
git status
git commit -m "<noi-dung-commit>"
git push -u origin main
```

### `Untracked files`

Git phát hiện file mới nhưng chưa theo dõi.

``` cmd
git add <ten-file>
```

### `Changes not staged for commit`

File đã được Git theo dõi nhưng có thay đổi chưa stage.

``` cmd
git diff
git add <ten-file>
```

### `Changes to be committed`

Thay đổi đã nằm trong Staging Area. Nếu đúng:

``` cmd
git commit -m "<mo-ta-thay-doi>"
```

### `nothing to commit, working tree clean`

Không có thay đổi chưa commit. Nếu đồng thời có:

``` text
Your branch is up to date with 'origin/main'.
```

thì nhánh cục bộ đang đồng bộ với remote tracking branch.

------------------------------------------------------------------------

## 10. Nguyên tắc làm việc

1.  **Kiểm tra trạng thái trước khi thao tác:** dùng `git status`.
2.  **Xem thay đổi trước khi commit:** dùng `git diff`.
3.  **Commit theo đơn vị công việc có ý nghĩa:** không chờ đến cuối dự
    án mới commit.
4.  **Không đưa thông tin bí mật vào Git:** không commit `.env`, mật
    khẩu, API key hoặc access token; dùng `.env.example` cho giá trị
    mẫu.
5.  **Push sau checkpoint ổn định:**

``` text
Code
 ↓
Run
 ↓
Test
 ↓
PASS
 ↓
git status / git diff
 ↓
Commit
 ↓
Push
```

------------------------------------------------------------------------

## 11. Câu hỏi củng cố

1.  `git add` và `git commit` khác nhau như thế nào?
2.  Vì sao ứng dụng chạy thành công chưa có nghĩa mã nguồn đã có trên
    GitHub?
3.  `git push` thực hiện việc gì?
4.  Vì sao cần kiểm tra `git remote -v` trước lần push đầu tiên?
5.  `git status` và `git diff` khác nhau như thế nào?
6.  Vì sao `.env` không nên được commit?
7.  Khi nào dùng `git push -u origin main` và khi nào chỉ cần
    `git push`?
8.  Vì sao nên tạo các commit có ý nghĩa theo từng checkpoint?

------------------------------------------------------------------------

## 12. Checklist

-   [ ] Đang đứng đúng thư mục repository.
-   [ ] Đã kiểm tra `git status`.
-   [ ] Đã kiểm tra `git remote -v`.
-   [ ] `origin` đúng repository.
-   [ ] Đã kiểm tra các file cần đưa vào Git.
-   [ ] Đã thực hiện `git add`.
-   [ ] Đã kiểm tra staging bằng `git status`.
-   [ ] Đã tạo commit với thông điệp có ý nghĩa.
-   [ ] Đã kiểm tra commit bằng `git log -1 --oneline`.
-   [ ] Đã push lên GitHub.
-   [ ] Đã kiểm tra repository trên GitHub.
-   [ ] `git status` cuối cùng ở trạng thái sạch.

------------------------------------------------------------------------

## 13. Ghi nhớ

``` text
git status
     ↓
git diff
     ↓
git add
     ↓
git status
     ↓
git commit
     ↓
git push
     ↓
git status
```

Về mặt khái niệm:

``` text
MÃ NGUỒN TRÊN MÁY
        ↓ git add
   STAGING AREA
        ↓ git commit
 LOCAL REPOSITORY
        ↓ git push
GITHUB REPOSITORY
```
