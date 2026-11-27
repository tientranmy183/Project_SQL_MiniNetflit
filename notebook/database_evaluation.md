# Đánh giá Cơ sở Dữ liệu Hiện tại — Mini-Netflix

> **Phạm vi đánh giá:** Toàn bộ thiết kế, mã nguồn SQL (`SQL/`), bộ kiểm thử (`SQL/tests/`), sơ đồ (`notebook/`) và tài liệu kỹ thuật (`docs/`) của dự án **Mini-Netflix** (Đề tài #2, môn Cơ sở Dữ liệu `INT1313`) trên **PostgreSQL 18**.

---

## 1. Tổng quan Kiến trúc CSDL

CSDL hiện tại gồm **18 bảng**, **11 chỉ mục bổ sung**, **6 hàm trigger / 12 trigger**, **3 view**, **4 vai trò nhóm + 4 vai trò đăng nhập (RBAC)** và hơn **3.700+ dòng dữ liệu mẫu** (cùng bộ test tự hoàn tác):

- **Nhóm Tài khoản & Hồ sơ**: `account_types`, `users`, `profiles` (thực thể yếu của `users`).
- **Nhóm Gói cước & Thanh toán**: `plans`, `subscriptions`, `payments`.
- **Nhóm Phân cấp Tác phẩm & Nội dung (EER)**: `titles`, `series`, `seasons` (thực thể yếu của `series`), `contents`, `movies` (lớp con chung của `titles` và `contents`), `episodes`.
- **Nhóm Phân loại & Nhân sự**: `genres`, `title_genres`, `people`, `title_credits`.
- **Nhóm Tương tác Người dùng**: `watch_history`, `watchlist_items`.

---

## 2. Điểm mạnh Nổi bật

### 2.1. Mô hình hóa EER & Ánh xạ Lược đồ Chuẩn xác
- **Phân cấp EER kép (Disjoint & Total Specialization)** được cài đặt chặt chẽ ở mức CSDL:
  - *Disjoint* ép bằng khóa ngoại ghép `(title_id, title_type)` / `(content_id, content_type)` kết hợp `CHECK` cố định giá trị trên các bảng con (`series`, `movies`, `episodes`).
  - *Total* ép bằng 6 constraint trigger `DEFERRABLE INITIALLY DEFERRED` (`contents_require_subclass`, `titles_require_subclass` và các trigger giữ lớp con khi xóa/sửa), cho phép chèn cha và con trong cùng một transaction và kiểm tra toàn vẹn tại thời điểm `COMMIT`.
- **Thực thể yếu** (`profiles`, `seasons`) dùng khóa chính ghép và `ON DELETE CASCADE` đúng nguyên tắc Elmasri.

### 2.2. Chuẩn hóa Cao (BCNF / 3NF) & Kiểm soát Dư thừa
- Toàn bộ 18 quan hệ ở mức logic đạt **BCNF** (và do đó đạt 3NF, 2NF, 1NF), không chứa thuộc tính đa trị, phức hợp hay phụ thuộc bắc cầu.
- Hai điểm đánh đổi có chủ đích ở mức vật lý đều được kiểm soát tốt:
  1. 4 cột phân loại cố định ở bảng con (`title_type`, `content_type`) được khóa cứng bằng `CHECK` nên không sinh dị thường cập nhật.
  2. Năm phát hành phim lẻ (`titles.release_year` và `contents.release_date`) được giữ đồng bộ bằng 3 deferred constraint trigger (`check_movie_release_year`).

### 2.3. Ràng buộc Nghiệp vụ & Chống Tranh chấp (Concurrency)
- **Chống trùng ngày gói đăng ký**: `subscriptions` sử dụng `EXCLUDE USING gist (user_id WITH =, daterange(start_date, end_date) WITH &&) WHERE (subscription_status <> 'CANCELLED')`, giải quyết triệt để ở mức CSDL mà không cần logic ứng dụng phức tạp.
- **Chống race condition khi tạo hồ sơ**: Hàm `check_profile_limit()` dùng `SELECT ... FROM users WHERE user_id = NEW.user_id FOR UPDATE` để tuần tự hóa các giao dịch chèn `profiles` đồng thời trên cùng một tài khoản.
- **Lịch sử xem theo ngữ cảnh thời gian**: `watch_history_enforce_access` kiểm tra gói hợp lệ theo `NEW.started_at::DATE` (giữ lịch sử cũ hợp lệ khi gói hết hạn) và chặn nội dung `age_rating > 12` cho hồ sơ trẻ em (`is_kids = TRUE`).

### 2.4. Bảo mật (RBAC) & Kiểm thử Tự động
- **Phân quyền tối thiểu (Least Privilege)** trong `SQL/08_roles.sql`:
  - Thu hồi toàn bộ quyền mặc định của `PUBLIC`, tách 4 vai trò nhóm (`netflix_admin`, `netflix_moderator`, `netflix_app`, `netflix_analyst`) và 4 login role tương ứng.
  - Phân quyền chi tiết tới **mức cột**: `netflix_app` chỉ sửa được các cột thông tin cơ bản của `users` (không tự nâng `account_type_code`), lịch sử `payments` chỉ được thêm (`INSERT`), `netflix_analyst` bị ẩn `email` và `password_hash`.
  - Tất cả 6 hàm trigger được cấu hình `SECURITY DEFINER SET search_path = public, pg_temp` và thu hồi quyền `EXECUTE` từ `PUBLIC`.
- **Bộ test bài bản**: `constraint_tests.sql` (40 ca kiểm tra ràng buộc/trigger), `rbac_tests.sql` (kiểm tra phân quyền), và `performance_tests.sql` (đo `EXPLAIN ANALYZE` trên ~50.000 phim và ~420.000 lượt xem).

### 2.5. Truy vấn & Chỉ mục Phù hợp Thực tế
- 14 truy vấn trong `SQL/07_queries.sql` bao phủ đầy đủ `JOIN` đa bảng qua phân cấp EER, CTE, Window Function (`ROW_NUMBER`, `RANK`, `LAG`), Subquery tương quan và thống kê theo khung giờ.
- Chỉ mục `titles_name_trgm_idx` (`GIN pg_trgm`) giúp tìm kiếm `ILIKE '%...%'` dưới `5 ms` (đạt yêu cầu phi chức năng `< 200 ms`), và `watch_history_recent_idx` tối ưu truy vấn "Tiếp tục xem" nhanh hơn 50–100 lần.

---

## 3. Hạn chế & Điểm có thể Cải thiện

### 3.1. Thiếu tầng Ứng dụng Demo (Yêu cầu Giai đoạn 4)
- Theo kế hoạch đánh giá môn học (`INT1313`), Giai đoạn 4 yêu cầu **giao diện (UI) và backend đơn giản** (Python/Flask, Node.js hoặc Java) kết nối trực tiếp tới CSDL khi bảo vệ đồ án. Hiện tại repository mới hoàn thiện tầng CSDL và tài liệu, **chưa có mã nguồn backend/UI**.

### 3.2. Một số Edge Case nhỏ ở Mức Trigger & Chỉ mục
- **Giảm thời lượng video (`contents.duration_seconds`)**: Trigger `watch_history_enforce_position` chỉ chạy khi `INSERT/UPDATE` trên `watch_history`. Nếu quản trị viên cập nhật giảm `contents.duration_seconds` nhỏ hơn `position_seconds` đã lưu trước đó, hệ thống chưa có trigger kiểm tra ngược từ `contents`.
- **Hồ sơ trẻ em đổi cờ (`profiles.is_kids`)**: Khi chuyển một hồ sơ từ người lớn (`FALSE`) sang trẻ em (`TRUE`), các lượt xem phim 18+ trong quá khứ vẫn giữ nguyên trong `watch_history` (hợp lý về mặt nhật ký lịch sử, nhưng giao diện "Tiếp tục xem" cần lọc lại theo `age_rating <= 12` khi hiển thị).
- **Trạng thái `subscriptions.subscription_status`**: Có thể tồn tại bản ghi vẫn ghi `'ACTIVE'` dù `end_date` đã qua nếu chưa có cron/job cập nhật sang `'EXPIRED'` (hiện tại `current_subscriptions` và `check_profile_limit` đã kiểm tra kèm `CURRENT_DATE < end_date` nên không sai logic).
- **Chỉ mục `watch_history_content_idx`**: Theo kết quả đo trong `docs/4_implementation.md`, chỉ mục này không nhanh hơn rõ rệt so với quét khóa chính ở bài test `COUNT(*)`, nhưng vẫn hữu ích cho kiểm tra khóa ngoại `ON DELETE RESTRICT` khi xóa `contents`.

### 3.3. Đồng bộ Tài liệu Phụ
- Tệp `Requirement/README.md` vẫn giữ nội dung mô tả bản nháp cũ dùng MySQL; nên ghi chú lại hoặc bỏ qua để tránh nhầm lẫn với `README.md` ở thư mục gốc (PostgreSQL 18).

---

## 4. Kết luận Tổng quan

| Tiêu chí | Đánh giá | Ghi chú |
|---|---|---|
| **Mô hình hóa EER & Quan hệ** | Xuất sắc | Thể hiện đầy đủ phân cấp EER kép, shared subclass, thực thể yếu, M:N |
| **Chuẩn hóa (3NF/BCNF)** | Rất tốt | 18/18 bảng đạt BCNF ở mức logic; ngoại lệ vật lý có kiểm soát bằng trigger |
| **Kỹ thuật SQL & Ràng buộc** | Rất tốt | Dùng `EXCLUDE USING gist`, `FOR UPDATE`, `DEFERRABLE` trigger, `pg_trgm` |
| **Bảo mật & Kiểm thử** | Rất tốt | RBAC phân quyền tới mức cột, `SECURITY DEFINER`, 40 ca test tự động |
| **Tích hợp Ứng dụng (Giai đoạn 4)** | Chưa có | Cần bổ sung backend + UI đơn giản kết nối PostgreSQL để hoàn thiện bảo vệ |
