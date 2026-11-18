---
description: Giải thích một peripheral/register của i.MX RT1062 hoặc DAC8568 theo chế độ học (không viết driver sẵn)
argument-hint: <module hoặc register, ví dụ "LPSPI TCR" hoặc "eDMA TCD">
---

Giải thích **$ARGUMENTS** cho người dùng đang tự viết driver bare-metal.

Trình bày theo thứ tự:
1. Peripheral/register này làm gì trong dự án (liên hệ `docs/02-architecture.md`).
2. Các field quan trọng: tên field, ý nghĩa, giá trị dự án cần. Ghi rõ "đối chiếu IMXRT1060RM chương <tên chương>" – không bịa số trang hay bit nếu không chắc.
3. Trình tự khởi tạo điển hình (các bước, không phải code hoàn chỉnh). Bao gồm clock gating (CCM) và mux chân (IOMUXC) nếu liên quan.
4. 3 lỗi hay gặp và cách phát hiện bằng UART log / logic analyzer (Teensy không có SWD).
5. Ánh xạ sang tham số cấu hình MCAL AUTOSAR tương ứng nếu có.
6. Kết thúc bằng 2–3 câu hỏi kiểm tra để người dùng tự trả lời trước khi code.

Không viết driver hoàn chỉnh. Đoạn mẫu tối đa 10 dòng.
