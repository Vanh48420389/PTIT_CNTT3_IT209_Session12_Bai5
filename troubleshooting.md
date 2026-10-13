# BÁO CÁO CHẨN ĐOÁN VÀ KHẮC PHỤC 3 SỰ CỐ ĐỨT GÃY PIPELINE CI/CD THỰC TẾ

**Học phần:** Điện toán đám mây / CI-CD & DevOps (IT209)  
**Session:** 12 - Tối ưu hóa & Nâng cao CI/CD Pipeline với GitHub Actions  
**Bài tập:** Bài 5 - Chẩn đoán và khắc phục 3 sự cố đứt gãy Pipeline CI/CD thực tế  
**Thư mục bài làm:** `Lession12/Bai05`  
**GitHub Submission Path:** `homework/session_12/ex5/`  

---

## 1. Giới thiệu tổng quan

Trong quy trình phát triển phần mềm hiện đại, quy trình Tích hợp và Triển khai liên tục (CI/CD) giữ vai trò là "xương sống" đảm bảo chất lượng và tốc độ phân phối sản phẩm. Tuy nhiên, các kỹ sư DevOps thường xuyên phải đối mặt với tình trạng pipeline bị đứt gãy (Broken Pipeline) do các lỗi về phân quyền hệ điều hành, cấu hình xác thực bí mật (Secrets), hoặc sai lệch đường dẫn ngữ cảnh đóng gói container.

Báo cáo này trình bày chi tiết quá trình điều tra nguyên nhân gốc rễ (**Root Cause Analysis - RCA**), phân tích log thực tế trên GitHub Actions Runner, và thực thi giải pháp khắc phục triệt để đối với 3 sự cố nghiêm trọng trong kịch bản `.github/workflows/broken-pipeline.yml`.

---

## 2. Chi tiết phân tích & khắc phục 3 sự cố

### 2.1. Sự cố 1: Lỗi từ chối quyền thực thi Gradle Wrapper (Permission Denied)

#### a. Hiện tượng và Trích xuất Log lỗi từ Runner
Tại bước thực thi kiểm thử tự động `./gradlew test`, job đột ngột kết thúc với mã lỗi `Process completed with exit code 126`:

```text
Run ./gradlew test
/home/runner/work/_temp/8a93ef12-32b1-4c91-9cb8-67f70b749d21.sh: line 1: ./gradlew: Permission denied
Error: Process completed with exit code 126.
```

#### b. Phân tích nguyên nhân gốc rễ (Root Cause Analysis)
- **Cơ chế:** GitHub Actions Runner sử dụng môi trường hệ điều hành Linux (`ubuntu-latest`). Để một tệp script bash shell (`gradlew`) có thể thực thi trực tiếp bằng cú pháp `./gradlew`, tệp đó bắt buộc phải được gắn cờ quyền thực thi (`executable bit: +x`, tức mã quyền POSIX `755` hoặc `775`).
- **Nguyên nhân:** Khi lập trình viên phát triển dự án trên hệ điều hành Windows, hệ thống tệp NTFS không có thuộc tính cờ thực thi tương đương Linux POSIX. Khi commit mã nguồn lên Git repository, tệp `gradlew` mặc định chỉ có quyền đọc/ghi (`rw-r--r--` / `644`). Khi runner checkout mã nguồn về máy ảo Linux, tệp bị thiếu quyền thực thi dẫn đến hệ điều hành từ chối truy cập (`Permission denied`).

#### c. Giải pháp khắc phục
Thêm một bước thực thi lệnh `chmod +x gradlew` ngay sau khi checkout mã nguồn và trước khi gọi lệnh `./gradlew test`.

```diff
       - name: Thiết lập môi trường Java JDK 17
         uses: actions/setup-java@v5
         with:
           java-version: '17'
           distribution: 'temurin'
           cache: 'gradle'

+      - name: Cấp quyền thực thi cho Gradle Wrapper (Fix Lỗi 1)
+        run: chmod +x gradlew

       - name: Thực thi JUnit Test
         run: ./gradlew test
```

*(Mẹo bổ sung: Có thể cập nhật vĩnh viễn quyền thực thi vào Git index ngay từ local bằng lệnh: `git update-index --chmod=+x gradlew`)*

---

### 2.2. Sự cố 2: Lỗi xác thực Docker Registry (Authentication Failed)

#### a. Hiện tượng và Trích xuất Log lỗi từ Runner
Tại bước đăng nhập vào Docker Registry bằng `docker/login-action@v3`, tiến trình xác thực thất bại:

```text
Run docker/login-action@v3
  with:
    registry: ghcr.io
    username: Vanh48420389
    password: ***
Error: Username and password are required
hoặc
Error: Error response from daemon: Get "https://ghcr.io/v2/": unauthorized: incorrect username or password
Error: Process completed with exit code 1.
```

#### b. Phân tích nguyên nhân gốc rễ (Root Cause Analysis)
- **Cơ chế:** Action `docker/login-action` yêu cầu cung cấp cặp thông tin `username` và `password`/`token` hợp lệ để xác thực với Registry API.
- **Nguyên nhân:** Tác giả kịch bản ban đầu đã gõ sai chính tả tên biến secret, ví dụ: `${{ secrets.REGISTRY_TOKNE }}` (hoặc `${{ secrets.DOCKER_TOKNE }}`). Vì GitHub Actions không báo lỗi cú pháp khi một biến secret không tồn tại mà âm thầm trả về chuỗi rỗng (`""`), giá trị token được truyền vào lệnh `docker login` hoàn toàn trống rỗng, khiến Registry từ chối phiên đăng nhập (`unauthorized`).

#### c. Giải pháp khắc phục
- Sửa lại chính xác tên biến Secret thành `${{ secrets.REGISTRY_TOKEN }}`.
- Để đảm bảo tính linh hoạt tối đa khi chạy trên GitHub Container Registry (GHCR), cấu hình cơ chế fallback tự động sang `${{ secrets.GITHUB_TOKEN }}` và sử dụng username mặc định `${{ github.actor }}`.

```diff
       - name: Đăng nhập Docker Registry
         uses: docker/login-action@v3
         with:
           registry: ghcr.io
-          username: ${{ secrets.REGISTRY_USENAME }}
-          password: ${{ secrets.REGISTRY_TOKNE }}
+          username: ${{ secrets.REGISTRY_USERNAME || github.actor }}
+          password: ${{ secrets.REGISTRY_TOKEN || secrets.GITHUB_TOKEN }}
```

---

### 2.3. Sự cố 3: Lỗi ngữ cảnh đóng gói Docker (Build Context Error)

#### a. Hiện tượng và Trích xuất Log lỗi từ Runner
Tại bước đóng gói Docker Image bằng `docker/build-push-action@v5`, Docker Buildx engine báo lỗi không tìm thấy tệp cấu hình:

```text
Run docker/build-push-action@v5
  with:
    context: ./src
    file: ./docker/Dockerfile
    push: true
...
ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory
hoặc
ERROR: failed to solve: failed to compute cache key: "/gradlew" not found: not found
Error: Process completed with exit code 1.
```

#### b. Phân tích nguyên nhân gốc rễ (Root Cause Analysis)
- **Cơ chế:** Khi thực hiện lệnh `docker build`, tham số `context` đại diện cho thư mục làm việc gốc (Working Context) mà Docker daemon có quyền truy cập để copy tệp (lệnh `COPY` trong Dockerfile). Tham số `file` chỉ định đường dẫn cụ thể đến tệp `Dockerfile`.
- **Nguyên nhân:** Kịch bản gốc bị cấu hình sai lệch ngữ cảnh:
  - Chỉ định `context: ./src`: Khi Dockerfile thực hiện lệnh `COPY gradlew .` hoặc `COPY build.gradle .`, các tệp này nằm ở thư mục gốc chứ không nằm trong `./src`, dẫn đến lỗi không tìm thấy tệp.
  - Hoặc chỉ định sai đường dẫn `file: ./docker/Dockerfile` trong khi tệp `Dockerfile` thực tế nằm ngay tại thư mục gốc `./Dockerfile`.

#### c. Giải pháp khắc phục
Chỉnh sửa cấu hình ngữ cảnh về đúng thư mục gốc của repository: `context: .` và đường dẫn tệp `file: ./Dockerfile`.

```diff
       - name: Đóng gói và đẩy Docker Image lên Registry (Fix Lỗi 3)
         uses: docker/build-push-action@v5
         with:
-          context: ./src
-          file: ./docker/Dockerfile
+          context: .
+          file: ./Dockerfile
           push: true
           tags: ghcr.io/${{ env.REPO_LC }}:latest
```

---

## 3. Bảng ma trận tổng hợp khắc phục sự cố

| STT | Sự cố | Mã lỗi / Triệu chứng Log | Nguyên nhân cốt lõi | Biện pháp xử lý | Trạng thái |
|:---:|:---|:---|:---|:---|:---:|
| **1** | **Permission Denied** | `line 1: ./gradlew: Permission denied` (Exit code 126) | Tệp `gradlew` thiếu cờ quyền thực thi POSIX `+x` trên môi trường Linux Runner. | Bổ sung lệnh `chmod +x gradlew` trước khi chạy `./gradlew test`. | **Resolved** |
| **2** | **Authentication Failed** | `unauthorized: incorrect username or password` (Exit code 1) | Gõ sai chính tả tên biến bí mật (`REGISTRY_TOKNE`), giá trị token truyền vào bị rỗng. | Sửa đúng tên `${{ secrets.REGISTRY_TOKEN }}` và hỗ trợ fallback `${{ secrets.GITHUB_TOKEN }}`. | **Resolved** |
| **3** | **Build Context Error** | `failed to read dockerfile: no such file or directory` (Exit code 1) | Chỉ định sai ngữ cảnh build (`context: ./src`) hoặc sai đường dẫn `Dockerfile`. | Sửa thành `context: .` và `file: ./Dockerfile` chuẩn theo thư mục gốc. | **Resolved** |

---

## 4. Kịch bản `.github/workflows/broken-pipeline.yml` hoàn chỉnh

```yaml
name: Fixed CI/CD Pipeline

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

permissions:
  contents: read
  packages: write

jobs:
  build-and-deploy:
    name: Build, Test & Docker Publish
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      # =======================================================================
      # [KHẮC PHỤC LỖI 1] Cấp quyền thực thi cho Gradle Wrapper
      # =======================================================================
      - name: Cấp quyền thực thi cho Gradle Wrapper (Fix Lỗi 1)
        run: chmod +x gradlew

      - name: Thực thi JUnit Test
        run: ./gradlew test

      - name: Đóng gói tệp JAR ứng dụng
        run: ./gradlew bootJar

      # =======================================================================
      # [KHẮC PHỤC LỖI 2] Sửa lỗi chính tả biến Secret đăng nhập Docker Registry
      # =======================================================================
      - name: Đăng nhập Docker Registry (Fix Lỗi 2)
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ secrets.REGISTRY_USERNAME || github.actor }}
          password: ${{ secrets.REGISTRY_TOKEN || secrets.GITHUB_TOKEN }}

      # Chuẩn hóa tên repository về chữ thường
      - name: Chuyển tên Repository về chữ thường
        run: |
          echo "REPO_LC=${GITHUB_REPOSITORY,,}" >> $GITHUB_ENV

      # =======================================================================
      # [KHẮC PHỤC LỖI 3] Cấu hình Build Context và đường dẫn file Dockerfile
      # =======================================================================
      - name: Đóng gói và đẩy Docker Image lên Registry (Fix Lỗi 3)
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ghcr.io/${{ env.REPO_LC }}:latest
```

---

## 5. Hướng dẫn kiểm tra và nộp bài

### Bước 1: Cam kết mã nguồn và đẩy lên GitHub
Tại thư mục `d:/IT209/Lession12/Bai05`:
```powershell
git init -b main
git add .
git commit -m "fix(ci): fix permission denied on gradlew, secret typo and docker build context"
git remote add origin https://github.com/Vanh48420389/PTIT_CNTT3_IT209_Session12_Bai5.git
git push -u origin main
```

### Bước 2: Quan sát và nghiệm thu kết quả trên GitHub Actions
1. Truy cập tab **Actions** trên repository GitHub.
2. Kiểm tra lần chạy của workflow **Fixed CI/CD Pipeline**:
   - Bước **`Cấp quyền thực thi cho Gradle Wrapper (Fix Lỗi 1)`** chạy thành công, mở đường cho `./gradlew test` chạy đạt kết quả `BUILD SUCCESSFUL`.
   - Bước **`Đăng nhập Docker Registry (Fix Lỗi 2)`** xác thực thành công `Login Succeeded`.
   - Bước **`Đóng gói và đẩy Docker Image lên Registry (Fix Lỗi 3)`** tìm đúng `Dockerfile` tại context gốc, build các stage và đẩy image lên GHCR thành công.
3. Toàn bộ Pipeline chuyển sang trạng thái tích xanh (**Success**).

### Bước 3: Hồ sơ nộp bài
- Tệp kịch bản đã sửa: `.github/workflows/broken-pipeline.yml`
- Tệp báo cáo khắc phục sự cố: `troubleshooting.md`
- Ảnh chụp màn hình kết quả chạy thành công toàn bộ pipeline (tích xanh) trên GitHub Actions.
- Nộp toàn bộ vào thư mục: `homework/session_12/ex5/`.
