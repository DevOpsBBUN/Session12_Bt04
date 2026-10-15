# BÁO CÁO THỰC HÀNH: TỰ ĐỘNG TRIỂN KHAI ỨNG DỤNG SPRING BOOT (CONTAINER) LÊN VPS TỪ XA QUA SSH

---

- **Môn học**: DevOps Fundamentals (IT-209)
- **Session**: 12 - Tối ưu hóa CI/CD Pipeline
- **Bài tập**: Bài 4 - Tự động triển khai ứng dụng Spring Boot (Container) lên VPS từ xa qua SSH
- **Học viên**: Hồ Quốc Khải
- **Email**: khaiho6040@gmail.com
- **Đường dẫn gợi ý nộp bài**: `homework/session_12/ex4/`

---

## 1. MỤC TIÊU VÀ NGUYÊN LÝ BẢO MẬT TRIỂN KHAI QUA SSH

### 1.1. Mục tiêu
- Xây dựng quy trình Triển khai liên tục (Continuous Deployment - CD) tự động hóa hoàn toàn việc đưa phiên bản ứng dụng Spring Boot mới nhất từ kho lưu trữ container (**GitHub Container Registry - GHCR**) lên máy chủ từ xa (**VPS Linux**).
- Kết nối bảo mật và an toàn tới máy chủ từ xa thông qua giao thức SSH bằng action tiêu chuẩn **`appleboy/ssh-action@v1.0.3`**.
- Thực thi kịch bản cập nhật container không gián đoạn: Đăng nhập Registry -> Kéo image mới -> Dừng container cũ -> Khởi chạy container mới trên cổng 8080 -> Xác nhận trạng thái hoạt động HTTP 200 OK qua Actuator / API.

### 1.2. Nguyên lý bảo mật SSH & GitHub Secrets
1. **Xác thực khóa bất đối xứng (Public/Private Key Authentication)**:
   - Sử dụng cặp khóa SSH Key (ED25519 hoặc RSA 4096-bit). Khóa công khai (`id_ed25519.pub`) được lưu trong `~/.ssh/authorized_keys` trên máy chủ VPS. Khóa bí mật (`id_ed25519`) được lưu trữ bảo mật trong **GitHub Repository Secrets (`SSH_KEY`)**.
   - Tuyệt đối không sử dụng mật khẩu đăng nhập trực tiếp (Password-less authentication).
2. **Che giấu thông tin nhạy cảm (Secret Masking)**:
   - Toàn bộ thông số kết nối: IP máy chủ (`SSH_HOST`), tài khoản đăng nhập (`SSH_USER`), khóa riêng tư (`SSH_KEY`) và mã truy cập Registry đều được mã hóa bằng thuật toán `libsodium` và tự động che giấu bằng ký tự `***` trong toàn bộ log của GitHub Actions runner.
3. **Mối quan hệ phụ thuộc (Job Dependency `needs: build-image`)**:
   - Đảm bảo Job `deploy-vps` chỉ được cấp phát runner và thực thi sau khi Job `build-image` đã biên dịch, đóng gói và đẩy thành công Docker Image lên `ghcr.io`.

---

## 2. CẤU TRÚC THƯ MỤC DỰ ÁN

```
Ss12/4/
├── .github/
│   └── workflows/
│       └── deploy-vps.yml       # Kịch bản GitHub Actions CD triển khai VPS qua SSH
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar   # Gradle wrapper executable
│       └── gradle-wrapper.properties # Cấu hình phiên bản Gradle (9.5.1)
├── src/
│   ├── main/
│   │   ├── java/com/example/tasks/
│   │   │   ├── controller/
│   │   │   │   └── TaskController.java        # REST Controller quản lý công việc
│   │   │   ├── model/
│   │   │   │   └── Task.java                  # JPA Entity Task
│   │   │   ├── repository/
│   │   │   │   └── TaskRepository.java        # JpaRepository giao tiếp Database
│   │   │   └── TaskApiApplication.java       # Spring Boot Main Application
│   │   └── resources/
│   │       ├── db/
│   │       │   └── taskdb.sql                 # DDL khởi tạo cơ sở dữ liệu
│   │       └── application.properties        # Cấu hình Datasource, Actuator và Port 8080
│   └── test/
│       ├── java/com/example/tasks/
│       │   ├── controller/
│       │   │   └── TaskControllerTest.java    # Unit test JUnit 5 & Mockito
│       │   └── TaskApiApplicationTests.java   # Spring Boot Context load test
│       └── resources/
│           └── application.properties        # Cấu hình H2 in-memory DB cho test
├── .env.example.properties       # Mẫu biến môi trường
├── .gitattributes                # Quy định xử lý line endings
├── .gitignore                    # Loại trừ build artifacts, cache, logs
├── build.gradle                  # Cấu hình Gradle build & Actuator dependency
├── docker-compose.yml            # Docker Compose service
├── Dockerfile                    # Multi-stage Dockerfile phơi cổng 8080
├── gradlew                       # Gradle wrapper script (Linux/macOS)
├── gradlew.bat                   # Gradle wrapper script (Windows)
├── settings.gradle               # Tên root project
└── README.md                     # Báo cáo thực hành chi tiết bài 4
```

---

## 3. NỘI DUNG KỊCH BẢN GITHUB ACTIONS WORKFLOW

Tệp kịch bản được lưu tại: `.github/workflows/deploy-vps.yml`

```yaml
name: Deploy Spring Boot Container to Remote VPS

on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  build-image:
    name: Build & Push Docker Image to GHCR
    runs-on: ubuntu-latest
    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Đăng nhập vào GitHub Container Registry (GHCR)
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.REGISTRY_TOKEN || secrets.GITHUB_TOKEN }}

      - name: Trích xuất Docker Metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=raw,value=latest
            type=sha,format=short

      - name: Đóng gói và đẩy Docker Image lên GHCR
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  deploy-vps:
    name: Deploy to Remote VPS via SSH
    needs: build-image
    runs-on: ubuntu-latest
    steps:
      - name: Thực thi câu lệnh triển khai trên VPS qua SSH
        uses: appleboy/ssh-action@v1.0.3
        env:
          REGISTRY_PASSWORD: ${{ secrets.REGISTRY_TOKEN || secrets.GITHUB_TOKEN }}
          REGISTRY_USER: ${{ github.actor }}
          IMAGE_NAME: ghcr.io/${{ github.repository }}:latest
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          envs: REGISTRY_PASSWORD,REGISTRY_USER,IMAGE_NAME
          script: |
            echo "=========================================="
            echo "🚀 Bắt đầu quá trình triển khai trên VPS..."
            echo "=========================================="

            echo "1. Đăng nhập vào GitHub Container Registry trên VPS..."
            echo "$REGISTRY_PASSWORD" | docker login ghcr.io -u "$REGISTRY_USER" --password-stdin

            echo "2. Kéo Docker Image mới nhất: $IMAGE_NAME..."
            docker pull "$IMAGE_NAME"

            echo "3. Dừng và xóa container cũ nếu đang chạy..."
            docker stop app || true
            docker rm app || true

            echo "4. Khởi chạy container mới trên cổng 8080:8080..."
            docker run -d \
              --name app \
              --restart unless-stopped \
              -p 8080:8080 \
              "$IMAGE_NAME"

            echo "5. Kiểm tra danh sách container đang chạy..."
            docker ps --filter "name=app"

            echo "=========================================="
            echo "✅ Quá trình triển khai hoàn tất thành công!"
            echo "=========================================="
```

### Phân tích chi tiết từng khối chỉ thị:

1. **Khối Job `build-image`**:
   - Sử dụng `docker/build-push-action@v5` kết hợp `docker/metadata-action@v5` đóng gói image và gắn tag `:latest` cùng mã băm commit SHA ngắn.
   - Đẩy trực tiếp image lên `ghcr.io/${{ github.repository }}:latest`.

2. **Khối Job `deploy-vps` với `needs: build-image`**:
   - Chỉ chạy khi image đã được build và push thành công lên registry.
   - Action `appleboy/ssh-action@v1.0.3` mở phiên SSH bảo mật tới địa chỉ IP `${{ secrets.SSH_HOST }}` với user `${{ secrets.SSH_USER }}` sử dụng private key `${{ secrets.SSH_KEY }}`.

3. **Kịch bản lệnh thực thi trên VPS**:
   - `docker login ghcr.io`: Xác thực VPS với GHCR để có quyền kéo image (đặc biệt khi image ở chế độ riêng tư).
   - `docker pull $IMAGE_NAME`: Kéo phiên bản image `:latest` mới nhất vừa được build.
   - `docker stop app || true && docker rm app || true`: Dừng và gỡ bỏ container cũ một cách an toàn mà không làm gãy lệnh nếu container chưa từng tồn tại trước đó (`|| true`).
   - `docker run -d --name app --restart unless-stopped -p 8080:8080 $IMAGE_NAME`: Khởi chạy container mới ở chế độ nền (detached), gắn chính sách tự khởi động lại khi máy chủ reboot và ánh xạ cổng `8080:8080`.
   - `docker ps --filter "name=app"`: In trạng thái container để ghi nhận nhật ký kiểm tra.

---

## 4. HƯỚNG DẪN CẤU HÌNH SECRETS VÀ KIỂM TRA

### Bước 1: Khai báo Repository Secrets trên GitHub
Truy cập: **Settings** -> **Secrets and variables** -> **Actions** -> Nhấn **New repository secret** và thêm các biến:

| Tên Secret | Ý nghĩa | Ví dụ giá trị |
| :--- | :--- | :--- |
| **`SSH_HOST`** | Địa chỉ IP công khai của máy chủ VPS | `103.176.179.52` |
| **`SSH_USER`** | Tên người dùng SSH trên VPS | `ubuntu` hoặc `root` |
| **`SSH_KEY`** | Khóa bí mật OpenSSH Private Key | Toàn bộ nội dung tệp `id_ed25519` hoặc `id_rsa` |
| **`REGISTRY_TOKEN`** | Token có quyền `read:packages` (tùy chọn) | `ghp_xxxxxxxxxxxx` (nếu dùng PAT) |

### Bước 2: Tự đẩy code lên nhánh `main`
```bash
# Di chuyển vào thư mục bài 4
cd "d:\[IT-209] DevOps Fundamentals\Ss12\4"

# Khởi tạo git và push nhánh main
git init
git checkout -b main
git add .
git commit -m "feat: setup automated SSH VPS deployment workflow"
git remote add origin https://github.com/<your-username>/<your-repo-name>.git
git push -u origin main
```

### Bước 3: Kiểm tra kết quả
1. Truy cập tab **Actions** trên GitHub repository:
   - Theo dõi 2 Job tuần tự: `Build & Push Docker Image to GHCR` -> `Deploy to Remote VPS via SSH`.
   - Mở chi tiết Job `Deploy to Remote VPS via SSH`, xem toàn bộ log thực thi các lệnh `docker pull`, `docker stop`, `docker rm` và `docker run` thành công.
2. Mở trình duyệt web hoặc công cụ curl truy cập:
   - `http://<VPS_IP>:8080/actuator/health` -> Trả về: `{"status":"UP"}` (HTTP 200 OK).
   - `http://<VPS_IP>:8080/api/tasks` -> Trả về danh sách công việc dạng JSON (HTTP 200 OK).
3. Chụp ảnh màn hình log kết quả bước SSH Deploy trên tab Actions (lưu vào `Ss12/4/` ví dụ `ssh_deploy_log.png`) để hoàn tất sản phẩm nộp bài.
