# BÀI TẬP 5: CHẨN ĐOÁN VÀ KHẮC PHỤC 3 SỰ CỐ ĐỨT GÃY PIPELINE CI/CD THỰC TẾ

**Học phần:** Điện toán đám mây / CI-CD & DevOps (IT209)  
**Session:** 12 - Tối ưu hóa & Nâng cao CI/CD Pipeline với GitHub Actions  
**Thư mục bài làm:** `Lession12/Bai05`  
**GitHub Submission Path:** `homework/session_12/ex5/`  

---

## 1. Mục tiêu bài thực hành
- Phân tích log lỗi thực tế trên môi trường máy ảo của GitHub Actions Runner khi pipeline bị đứt gãy.
- Xác định nguyên nhân gốc rễ và thực hiện sửa lỗi trực tiếp trên tệp kịch bản `.github/workflows/broken-pipeline.yml` để khôi phục trạng thái tích xanh.
- Nắm vững 3 lỗi kinh điển trong CI/CD:
  1. **Lỗi 1 (Permission Denied):** Gradle Wrapper thiếu cờ thực thi `+x` trên môi trường Linux Runner.
  2. **Lỗi 2 (Authentication Failed):** Lỗi đăng nhập Docker Registry do sai chính tả tên biến bí mật (`secrets`).
  3. **Lỗi 3 (Build Context Error):** Lệnh build Docker Image không tìm thấy file `Dockerfile` do chỉ định sai thư mục ngữ cảnh (context).

---

## 2. Cấu trúc thư mục dự án

```text
Bai05/
├── .github/
│   └── workflows/
│       └── broken-pipeline.yml  # Kịch bản GitHub Actions sau khi được khắc phục hoàn toàn 3 lỗi
├── gradle/
│   └── wrapper/                 # Gradle Wrapper
├── src/
│   ├── main/java/               # Mã nguồn ứng dụng Spring Boot
│   └── test/java/               # Bộ Unit Test JUnit 5
├── Dockerfile                   # Kịch bản Multi-stage Build Docker
├── docker-compose.yml           # Khởi chạy cục bộ
├── build.gradle                 # Cấu hình Gradle build & dependencies
├── settings.gradle              # Tên định danh dự án
├── gradlew                      # Gradle wrapper script cho Linux/macOS
├── gradlew.bat                  # Gradle wrapper script cho Windows
├── troubleshooting.md           # Báo cáo chuyên sâu phân tích nguyên nhân gốc rễ & giải pháp khắc phục
└── README.md                    # Hướng dẫn tổng quan & nộp bài
```

---

## 3. Tổng hợp 3 lỗi và phương pháp xử lý

| STT | Lỗi ban đầu | Nguyên nhân | Giải pháp xử lý |
|:---:|:---|:---|:---|
| **1** | `line 1: ./gradlew: Permission denied` (Exit code 126) | Tệp `gradlew` thiếu quyền `+x` khi runner checkout từ git | Thêm `run: chmod +x gradlew` trước lệnh test |
| **2** | `unauthorized: incorrect username or password` (Exit code 1) | Gõ sai chính tả `${{ secrets.REGISTRY_TOKNE }}` dẫn đến token bị rỗng | Sửa thành `${{ secrets.REGISTRY_TOKEN }}` (hỗ trợ fallback `${{ secrets.GITHUB_TOKEN }}`) |
| **3** | `ERROR: failed to solve: failed to read dockerfile: no such file or directory` | Tham số `context: ./src` hoặc sai đường dẫn `file: ./docker/Dockerfile` | Cấu hình đúng `context: .` và `file: ./Dockerfile` |

Chi tiết phân tích log và so sánh diff được trình bày đầy đủ tại tệp [`troubleshooting.md`](file:///d:/IT209/Lession12/Bai05/troubleshooting.md).

---

## 4. Hướng dẫn kiểm tra và nộp bài

### Bước 1: Khởi tạo Git & Đẩy code lên GitHub
Mở PowerShell tại thư mục `d:/IT209/Lession12/Bai05`:
```powershell
git init -b main
git add .
git commit -m "fix(ci): fix permission denied on gradlew, secret typo and docker build context"
git remote add origin https://github.com/Vanh48420389/PTIT_CNTT3_IT209_Session12_Bai5.git
git push -u origin main
```

### Bước 2: Quan sát tab Actions trên GitHub
- Mở repository trên GitHub, chọn tab **Actions**.
- Lựa chọn workflow **`Fixed CI/CD Pipeline`**.
- Xác nhận cả 3 bước đều vượt qua thành công:
  - Cấp quyền cho gradlew $\rightarrow$ `./gradlew test` thành công (**Success**).
  - Đăng nhập Docker Registry thành công (**Success**).
  - Đóng gói và đẩy Docker Image thành công (**Success**).
- Toàn bộ Pipeline chuyển trạng thái tích xanh (**All checks have passed**).

### Bước 3: Hồ sơ nộp bài
- Tệp kịch bản đã sửa: `.github/workflows/broken-pipeline.yml`
- Báo cáo phân tích sự cố: `troubleshooting.md`
- Ảnh chụp màn hình kết quả chạy tích xanh trên GitHub Actions.
- Nộp vào thư mục quy định: `homework/session_12/ex5/`.
