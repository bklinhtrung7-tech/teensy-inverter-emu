# 04 – Làm việc với AI: Claude Code + Claude desktop (Cowork)

Mục tiêu: dùng AI để **học nhanh hơn**, không phải để né phần cần học. Kỹ năng đáng luyện là *dẫn dắt và kiểm chứng* AI, giống cách một FAE kiểm chứng câu trả lời trước khi đưa cho khách hàng.

## 1. Chia việc

| Việc | Công cụ | Vì sao |
|---|---|---|
| Đọc RM/datasheet PDF, tóm tắt chương thành `docs/notes/` | **Cowork** (Claude desktop, cấp quyền thư mục repo) | Làm việc tốt với tài liệu dài, file, ghi chú |
| Lên kế hoạch tuần, review `LEARNING_LOG.md`, soạn ADR, vẽ sơ đồ | **Cowork** | Việc tài liệu, không cần terminal |
| Soạn test spec cho CANoe, bảng ma trận CAN | **Cowork** | |
| Viết/review code, build, đọc lỗi compiler, chạy test | **Claude Code** (terminal hoặc VS Code) | Có shell, đọc toàn repo, chạy lệnh |
| Giải thích register, review diff, kiểm tra milestone | **Claude Code** + slash command trong `.claude/commands/` | Ngữ cảnh code |
| Git | **Bạn tự gõ**; Claude Code giải thích và đề xuất | Mục tiêu học |

Cả hai cùng trỏ vào **một thư mục repo**: ghi chú Cowork viết ra (`docs/notes/*.md`) trở thành ngữ cảnh cho Claude Code ở phiên sau. Cũng commit các file đó.

## 2. Thang mức can thiệp (đã ghi trong `CLAUDE.md`)

| Mức | Tên | Bạn | Claude |
|---|---|---|---|
| L1 | Explain | viết toàn bộ | giải thích, chỉ chỗ trong RM, đặt câu hỏi |
| L2 | Review | viết toàn bộ | review, chỉ lỗi, không sửa |
| L3 | Pair | viết phần lõi | viết khung/boilerplate |
| L4 | Delegate | review | viết toàn bộ |

Quy tắc ngón tay cái: thứ bạn muốn **kể trong phỏng vấn** thì L1–L2. Thứ chỉ là công cụ (script Python, YAML CI, CAPL panel) thì L4.

## 3. Tính năng Claude Code nên luyện (theo phase)

| Phase | Tính năng | Bài tập |
|---|---|---|
| P0 | `CLAUDE.md` | Đọc file có sẵn, chỉnh mục "Người dùng" cho đúng ý bạn. Hỏi Claude "tóm tắt dự án này" để xem nó hiểu đúng chưa |
| P0 | Permissions (`.claude/settings.json`) | Thử bảo Claude `git push` → thấy bị chặn. Hiểu allow/deny |
| P1 | **Plan mode** | Trước khi làm startup/linker: yêu cầu Claude lập kế hoạch, bạn sửa kế hoạch rồi mới làm |
| P1 | Slash command `/explain-reg` | `/explain-reg CCM clock root LPSPI` |
| P2 | `/review` + subagent `embedded-reviewer` | Viết driver LPSPI xong, stage, chạy `/review` |
| P2 | Nhắc file bằng `@` | `@docs/00-hardware.md @mcal/Spi/Spi.c kiểm tra mode SPI có khớp DAC không` |
| P3 | Quản lý ngữ cảnh: `/clear` giữa hai chủ đề, `/compact` khi phiên dài | Quan sát chất lượng câu trả lời thay đổi thế nào |
| P3 | Giao việc L4 | "Viết `tools/gen_lut.py` sinh LUT sin 1024 điểm int16 ra `gen/SigGen_Lut.c`, có test" |
| P4 | Đưa bằng chứng thay vì mô tả | Dán log UART, ảnh logic analyzer (kéo ảnh vào), output `arm-none-eabi-objdump -h` |
| P5 | Tự viết slash command mới | Ví dụ `/rtos-stats` diễn giải output lệnh `stats` |
| P5 | **Hook** | Thêm hook `PostToolUse` chạy `clang-format -i` sau mỗi lần Claude sửa file `.c/.h` (đọc tài liệu hooks của Claude Code, tự viết vào `.claude/settings.json`) |
| P6 | Subagent thứ hai | Ví dụ `can-test-writer` chuyên viết test pytest từ DBC |
| P6 | `git worktree` + 2 phiên Claude Code | Một phiên làm FlexCAN, một phiên viết tool Python |
| P6+ | Chế độ headless (`claude -p "..."`) | Gọi trong script để sinh changelog từ `git log` |

> Tính năng của Claude Code thay đổi theo phiên bản. Khi một cú pháp ở đây không chạy, xem `claude --help` và tài liệu tại docs.claude.com.

## 4. Mẫu prompt hiệu quả

**Học register (L1):**
> Mình đang viết `Spi_Init` cho LPSPI4. Đừng viết code. Liệt kê các bước init theo RM, và với mỗi bước hỏi mình một câu để mình tự tìm field trong RM.

**Review (L2):**
> /review — chú ý thứ tự disable module trước khi ghi TCR/CCR, và chỗ mình tính SCKDIV.

**Debug khi không có SWD:**
> Triệu chứng: <…>. Bằng chứng: log UART <…>, ảnh SYNC/SCLK đính kèm. Liệt kê 3 giả thuyết theo khả năng, mỗi giả thuyết cho một phép đo GPIO/log để loại trừ.

**Kiểm chứng (thói quen FAE):**
> Câu trả lời vừa rồi dựa vào đâu? Field nào mình cần mở RM để xác nhận?

**Giao việc (L4):**
> Viết `tools/can_ctrl.py` dùng python-can + cantools với `can/inverter_emu.dbc`: CLI `--freq --amp --enable`, gửi cyclic 20 ms có alive counter, in status nhận được. Có `--interface` để chọn socketcan/slcan/vector.

## 5. Cách dùng Cowork cho tài liệu

- Cấp cho Claude desktop quyền vào thư mục repo. Đặt PDF ở `docs/datasheets/` (bị `.gitignore`, không đẩy lên GitHub vì vấn đề bản quyền).
- Prompt mẫu: "Đọc chương LPSPI trong `docs/datasheets/IMXRT1060RM.pdf`, viết `docs/notes/lpspi.md`: mục đích, sơ đồ khối bằng lời, các thanh ghi dự án dùng, trình tự init, lỗi thường gặp. Ghi số mục/bảng của RM cạnh mỗi ý để mình tra lại. Không chép nguyên văn."
- Chủ nhật hằng tuần: "Đọc `LEARNING_LOG.md` và `docs/01-roadmap.md`, cho mình biết đang chậm/nhanh bao nhiêu và đề xuất kế hoạch 2 buổi tuần tới."

## 6. Cạm bẫy

- AI rất tự tin khi nói về bit register và có thể sai. **Mọi thứ đụng phần cứng phải đối chiếu RM/datasheet.** Ghi lại các lần AI sai vào `docs/notes/ai-mistakes.md`; danh sách này là bài học quý (và là chuyện hay để kể khi phỏng vấn).
- Đừng để phiên Claude Code kéo quá dài với nhiều chủ đề; `/clear` khi đổi việc.
- Không dán thông tin mật của công ty (code, DBC thật, CDD) vào bất kỳ AI nào. DBC trong repo này là tự thiết kế.
