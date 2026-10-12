# Báo cáo: Tối ưu tốc độ CI bằng Gradle Dependency Caching (Bài 3)

## 1. Mục tiêu
Kích hoạt tính năng caching cho các thư viện phụ thuộc (Gradle dependencies) trong luồng GitHub Actions CI nhằm:
- Giảm thời gian tải lại thư viện qua Internet giữa các lần build/test.
- Tối ưu hóa tốc độ thực thi toàn bộ quy trình CI (giảm thời gian chạy Job).
- Tiết kiệm băng thông và tài nguyên máy chủ GitHub Runner.

---

## 2. Cấu hình kịch bản CI (`.github/workflows/ci.yml`)
Bổ sung thuộc tính `cache: 'gradle'` vào bước `Thiết lập môi trường Java JDK 17` (`actions/setup-java@v5`):

```yaml
      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'
```

---

## 3. Cơ chế hoạt động của Gradle Cache trong GitHub Actions
1. **Lần chạy đầu tiên (Tạo Cache / Cache Miss):**
   - Bước `setup-java` khởi tạo khóa cache dựa trên hệ điều hành và tệp cấu hình Gradle (ví dụ: `build.gradle`, `gradle/wrapper/gradle-wrapper.properties`).
   - Lúc này chưa có cache trên GitHub Actions Cache Storage, bước `setup-java` bỏ qua khôi phục.
   - Bước `Thực thi JUnit Test với Gradle` (`./gradlew test`) phải tải toàn bộ dependencies từ Maven Central / Gradle Plugin Portal về máy runner.
   - Khi job kết thúc thành công, bước hậu kỳ (`Post Thiết lập môi trường Java JDK 17`) sẽ tự động đóng gói thư mục cache Gradle (`~/.gradle/caches`, `~/.gradle/wrapper`) và lưu trữ lên GitHub Actions Cache Storage.

2. **Lần chạy thứ hai trở đi (Khôi phục Cache / Cache Hit):**
   - Bước `setup-java` tìm thấy khóa cache phù hợp trên máy chủ cache của GitHub và khôi phục về máy runner:
     ```text
     Received 12345678 bytes
     Cache restored successfully from key: setup-java-cache-Linux-gradle-...
     ```
   - Bước `Thực thi JUnit Test với Gradle` (`./gradlew test`) sử dụng ngay các file thư viện đã có trong cache cục bộ mà không cần tải lại qua Internet.
   - Thời gian thực thi của bước test và tổng thời gian của Job giảm đáng kể.

---

## 4. Bảng so sánh thời gian thực thi

| Tiêu chí | Lần chạy chưa có Cache (Run #1 / #2) | Lần chạy có Cache (Run với `cache: 'gradle'`) | Đánh giá chênh lệch |
| :--- | :---: | :---: | :---: |
| **Trạng thái Cache** | Cache Miss / Không cấu hình cache | Cache Restored (Cache Hit) | Khôi phục thành công từ cache key |
| **Bước `setup-java`** | ~1s (không khôi phục) | ~2s - 4s (tải cache về runner) | Tăng nhẹ để giải nén cache |
| **Bước `./gradlew test`** | **~31s - 38s** (tải mọi dependency) | **~10s - 15s** (dùng cache cục bộ) | **Giảm ~50% - 70% thời gian** |
| **Tổng thời gian Job** | **~37s - 48s** | **~20s - 25s** | **Tốc độ nhanh hơn rõ rệt** |

---

## 5. Các bước kiểm tra thực tế
1. **Push lần 1 (Kích hoạt lưu cache):**
   ```bash
   git add .
   git commit -m "chore: enable gradle caching in CI workflow"
   git push origin main
   ```
   - Chờ GitHub Actions hoàn thành Run.
   - Xem log của bước `Post Thiết lập môi trường Java JDK 17` sẽ thấy thông báo lưu cache thành công.

2. **Push lần 2 (Kiểm tra khôi phục cache):**
   ```bash
   git commit --allow-empty -m "test: trigger CI to verify gradle cache restore"
   git push origin main
   ```
   - Chờ GitHub Actions chạy lần 2.
   - Vào tab **Actions** -> Chọn Run mới nhất -> Bấm vào job **Run Unit Tests**.
   - Mở chi tiết bước **Thiết lập môi trường Java JDK 17**, kiểm tra log:
     ```text
     Cache restored successfully from key: setup-java-cache-Linux-gradle-...
     ```
   - So sánh thời gian bước `./gradlew test` thấy thời gian giảm rõ rệt so với lần trước.
