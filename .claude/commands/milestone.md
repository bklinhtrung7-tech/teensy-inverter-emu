---
description: Kiểm tra DoD của một phase trước khi tag release
argument-hint: <phase, ví dụ P2>
---

1. Đọc mục **$ARGUMENTS** trong `docs/01-roadmap.md`.
2. Với từng checkbox và từng ý trong **DoD**: tìm bằng chứng trong repo (file, test, ảnh trong `docs/img/`, ghi chú trong `docs/notes/`, `LEARNING_LOG.md`). Đánh dấu ✅ có bằng chứng / ❌ thiếu / ❓ cần người dùng xác nhận (đo đạc thực tế).
3. Chạy build và host test nếu đã có (`cmake --build build`, `ctest --test-dir build-host`), báo kết quả.
4. Nếu đủ: soạn sẵn nội dung **annotated tag** (tóm tắt phase, số liệu đo được) và lệnh để người dùng tự chạy:
   `git tag -a vX.Y.Z -m "..."` rồi `git push origin vX.Y.Z`. Không tự tag.
5. Đề xuất cập nhật mục "Trạng thái" trong `CLAUDE.md` và `README.md`.
