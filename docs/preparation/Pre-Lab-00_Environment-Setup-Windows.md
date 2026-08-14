# PRE-LAB 00 – CHUẨN BỊ MÔI TRƯỜNG THỰC HÀNH

## 1. Mục tiêu

Tài liệu này hướng dẫn chuẩn bị môi trường cần thiết trước khi thực hiện chuỗi hands-on của dự án **PHP – MySQL Sales Management**.

Sau khi hoàn thành, người học cần có:

- Git.
- Visual Studio Code.
- Docker Desktop.
- WSL 2 hoạt động trên Windows.
- Tài khoản GitHub.
- Trình duyệt Web hiện đại.
- Khả năng chạy Docker container và Docker Compose từ Windows Command Prompt (CMD).

> **Môi trường chuẩn của tài liệu:** Windows 10/11 64-bit, Windows Command Prompt (CMD).

---

## 2. Danh sách phần mềm cần chuẩn bị

| Thành phần | Vai trò |
|---|---|
| Git | Quản lý phiên bản mã nguồn |
| GitHub account | Lưu trữ repository và làm việc với GitHub |
| Visual Studio Code | Soạn thảo mã nguồn |
| WSL 2 | Backend Linux được Docker Desktop sử dụng trên Windows |
| Docker Desktop | Chạy môi trường PHP, Apache và MySQL bằng container |
| Web browser | Chạy và kiểm tra ứng dụng Web |

---

## 3. Kiểm tra cấu hình máy trước khi cài Docker Desktop

Theo tài liệu Docker Desktop dành cho Windows, máy cần đáp ứng các điều kiện phần cứng phù hợp với WSL 2, trong đó đáng chú ý:

- Bộ xử lý 64-bit hỗ trợ SLAT.
- RAM hệ thống tối thiểu 8 GB.
- Hardware virtualization được bật trong BIOS/UEFI.
- Windows hỗ trợ WSL 2.

Nếu Docker Desktop không khởi động được, một trong những nguyên nhân cần kiểm tra đầu tiên là **Virtualization** hoặc **WSL 2**.

### Kiểm tra Virtualization

Nhấn:

```text
Ctrl + Shift + Esc
```

Mở:

```text
Task Manager → Performance → CPU
```

Tìm thông tin:

```text
Virtualization: Enabled
```

Nếu hiển thị `Disabled`, cần bật virtualization trong BIOS/UEFI trước khi tiếp tục.

---

## 4. Cài đặt hoặc kiểm tra WSL 2

### 4.1. Kiểm tra WSL

Mở CMD và chạy:

```cmd
wsl --status
```

Kiểm tra phiên bản:

```cmd
wsl --version
```

Có thể kiểm tra các distribution bằng:

```cmd
wsl -l -v
```

Nếu WSL đã được cài đúng, distribution đang sử dụng nên ở `VERSION 2`.

### 4.2. Cài WSL nếu chưa có

Mở **Windows Terminal hoặc PowerShell với quyền Administrator** và chạy:

```powershell
wsl --install
```

Khởi động lại máy nếu Windows yêu cầu.

Sau khi khởi động lại, kiểm tra:

```cmd
wsl --status
```

### 4.3. Cập nhật WSL

Có thể chạy:

```cmd
wsl --update
```

> Tài liệu chính thức:  
> https://learn.microsoft.com/windows/wsl/install

---

## 5. Cài Git for Windows

Tải Git for Windows từ trang chính thức:

```text
https://git-scm.com/install/windows
```

Chạy bộ cài và có thể giữ các lựa chọn mặc định nếu chưa có yêu cầu cấu hình đặc biệt.

Sau khi cài đặt, đóng CMD cũ, mở lại CMD và kiểm tra:

```cmd
git --version
```

Kết quả có dạng:

```text
git version x.x.x.windows.x
```

### Cấu hình tên và email Git

Nếu chưa từng cấu hình Git trên máy, chạy:

```cmd
git config --global user.name "Ho Ten"
git config --global user.email "email@example.com"
```

Kiểm tra:

```cmd
git config --global user.name
git config --global user.email
```

Nên sử dụng email gắn với tài khoản GitHub.

---

## 6. Tạo tài khoản GitHub

Nếu chưa có tài khoản, truy cập:

```text
https://github.com/
```

Tạo tài khoản và xác minh email.

Sau khi đăng nhập, cần bảo đảm có thể:

- Tạo repository.
- Clone repository.
- Push mã nguồn lên GitHub.

> Trong hands-on, người học sẽ sử dụng Git/GitHub trực tiếp thay vì phụ thuộc vào GitHub Classroom.

---

## 7. Cài Visual Studio Code

Tải Visual Studio Code từ:

```text
https://code.visualstudio.com/
```

Trang hướng dẫn cài đặt Windows:

```text
https://code.visualstudio.com/docs/setup/windows
```

Nên sử dụng **User Setup** nếu không cần cài cho toàn bộ tài khoản trên máy.

Sau khi cài xong, đóng CMD cũ và mở lại CMD.

Kiểm tra:

```cmd
code --version
```

Nếu lệnh hoạt động, có thể mở thư mục hiện tại bằng:

```cmd
code .
```

### Extension khuyến nghị

Có thể cài thêm:

- PHP Intelephense.
- Docker.
- GitHub Pull Requests.

Các extension này hỗ trợ thao tác thuận tiện hơn nhưng không phải điều kiện bắt buộc để chạy dự án.

---

## 8. Cài Docker Desktop

Tải Docker Desktop từ trang chính thức:

```text
https://www.docker.com/products/docker-desktop/
```

Tài liệu cài đặt chính thức:

```text
https://docs.docker.com/desktop/setup/install/windows-install/
```

Trong quá trình cài đặt, nên sử dụng backend **WSL 2**.

Sau khi cài:

1. Khởi động lại máy nếu được yêu cầu.
2. Mở **Docker Desktop** từ Start Menu.
3. Chờ Docker Desktop hoàn tất khởi động.
4. Vào:

```text
Settings → General
```

kiểm tra tùy chọn:

```text
Use WSL 2 based engine
```

Tài liệu Docker về WSL 2:

```text
https://docs.docker.com/desktop/features/wsl/
```

---

## 9. Kiểm tra Docker sau khi cài đặt

### 9.1. Kiểm tra Docker CLI

```cmd
docker --version
```

### 9.2. Kiểm tra Docker Engine

```cmd
docker version
```

Kết quả hợp lệ cần có cả:

```text
Client:
...
Server:
...
```

> Chỉ có kết quả `docker --version` chưa đủ để kết luận Docker đã sẵn sàng. Docker CLI có thể tồn tại trong khi Docker Engine chưa chạy.

### 9.3. Kiểm tra Docker Compose

```cmd
docker compose version
```

### 9.4. Chạy container kiểm tra

```cmd
docker run --rm hello-world
```

Nếu Docker hoạt động bình thường, terminal sẽ hiển thị thông báo xác nhận Docker đã chạy thành công.

---

## 10. Lỗi thường gặp: Docker Desktop chưa được bật

Nếu chạy:

```cmd
docker compose up -d --build
```

và xuất hiện lỗi tương tự:

```text
open //./pipe/dockerDesktopLinuxEngine:
The system cannot find the file specified.
```

trong khi:

```cmd
docker --version
```

vẫn chạy được, nguyên nhân có thể đơn giản là **Docker Desktop chưa được khởi động**.

Cách xử lý:

1. Mở Docker Desktop.
2. Chờ Docker Engine sẵn sàng.
3. Chạy:

```cmd
docker version
```

4. Xác nhận có cả `Client` và `Server`.
5. Chạy lại lệnh Docker Compose.

---

## 11. Lỗi thường gặp: Docker Desktop không chạy với WSL 2

Kiểm tra:

```cmd
wsl --status
wsl -l -v
```

Nếu WSL chưa có hoặc chưa ở phiên bản 2, thực hiện:

```powershell
wsl --install
```

hoặc cập nhật:

```cmd
wsl --update
```

Nếu cần kiểm tra cấu hình WSL 2 trong Docker Desktop:

```text
Docker Desktop
→ Settings
→ General
→ Use WSL 2 based engine
```

---

## 12. Kiểm tra trình duyệt

Có thể sử dụng một trong các trình duyệt hiện đại:

- Microsoft Edge.
- Google Chrome.
- Mozilla Firefox.

Trong hands-on, ứng dụng local sẽ được truy cập theo dạng:

```text
http://localhost:8080
```

---

## 13. Kiểm tra toàn bộ môi trường

Trước khi bắt đầu Hands-on 01, mở CMD và chạy lần lượt:

```cmd
git --version
code --version
wsl --status
docker --version
docker version
docker compose version
```

Sau đó:

```cmd
docker run --rm hello-world
```

### Checklist

```text
[ ] Git hoạt động
[ ] GitHub account đã sẵn sàng
[ ] Visual Studio Code hoạt động
[ ] WSL 2 hoạt động
[ ] Docker CLI hoạt động
[ ] Docker Engine hoạt động
[ ] Docker Compose hoạt động
[ ] hello-world container chạy thành công
[ ] Trình duyệt Web hoạt động
```

Chỉ nên bắt đầu Hands-on 01 khi các mục trên đã đạt.

---

## 14. Cấu trúc thư mục đề nghị trên máy

Nên tạo một thư mục ngắn gọn và dễ quản lý, ví dụ:

```text
D:\PTUDWMNM-project\
```

Các repository được clone vào đây:

```text
D:\PTUDWMNM-project\
└── php-mysql-sales-template\
```

Hạn chế đặt repository trong các đường dẫn quá dài hoặc khó quản lý.

---

## 15. Những phần không cần cài riêng

Với kiến trúc dự án sử dụng Docker, người học **không cần cài riêng**:

- PHP.
- Apache.
- MySQL.
- phpMyAdmin.
- XAMPP.
- WAMP.

Các thành phần cần thiết sẽ được cung cấp qua Docker container trong quá trình thực hành.

Điều này giúp giảm khác biệt môi trường giữa các máy và tránh xung đột phiên bản.

---

## 16. Kết quả cần đạt trước Hands-on 01

Môi trường được xem là sẵn sàng khi luồng sau hoạt động:

```text
Windows
   │
   ├── Git
   ├── VS Code
   ├── WSL 2
   └── Docker Desktop
          ↓
      Docker Engine
          ↓
     Docker Compose
          ↓
      Containers
```

Sau khi đạt checkpoint này, người học có thể chuyển sang:

```text
Hands-on 01
→ Tạo repository
→ Clone project
→ Tạo cấu trúc
→ Build PHP + Apache
→ Chạy ứng dụng đầu tiên
```

---

## 17. Nguồn tham khảo chính thức

- Docker Desktop for Windows:  
  https://docs.docker.com/desktop/setup/install/windows-install/

- Docker Desktop với WSL 2:  
  https://docs.docker.com/desktop/features/wsl/

- Microsoft WSL installation:  
  https://learn.microsoft.com/windows/wsl/install

- Git for Windows:  
  https://git-scm.com/install/windows

- Visual Studio Code for Windows:  
  https://code.visualstudio.com/docs/setup/windows

- GitHub:  
  https://github.com/
