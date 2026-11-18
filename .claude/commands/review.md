---
description: Review thay đổi đang staged (hoặc so với main) theo quy tắc của dự án
argument-hint: "[staged|branch]  (mặc định: staged)"
---

Phạm vi: nếu `$ARGUMENTS` là `branch` thì dùng `git diff main...HEAD`, ngược lại dùng `git diff --staged`. Nếu diff rỗng, báo và dừng.

Dùng subagent `embedded-reviewer` để review, rồi tổng hợp kết quả cho người dùng:

- **Phải sửa** (bug, race ISR/task, thiếu `volatile`, cache/DMA coherency, sai register, tràn số trong DDS/Q15, vi phạm quy ước trong `CLAUDE.md`)
- **Nên sửa** (đọc khó, đặt tên, thiếu test cho logic thuần)
- **Câu hỏi** cho người dùng (chỗ chưa rõ ý đồ)

Mỗi mục ghi `file:dòng`. Không tự sửa code. Cuối cùng đề xuất 1 commit message Conventional Commits cho thay đổi này.
