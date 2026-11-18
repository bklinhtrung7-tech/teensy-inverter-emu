# CLAUDE.md – teensy-inverter-emu

Claude Code đọc file này mỗi phiên. Giữ ngắn, cập nhật mục "Trạng thái" sau mỗi phase.

## Dự án

Giả lập dòng điện 3 pha của inverter (current sensor emulator) trên **Teensy 4.0 (NXP i.MX RT1062, Cortex-M7 600 MHz)** + **TI DAC8568** (16 bit, 8 kênh, SPI 32-bit frame). Sóng tạo bằng DDS + LUT, đẩy ra SPI bằng **PIT → DMAMUX → eDMA → LPSPI4**, chạy trên **FreeRTOS**, điều khiển biên độ/tần số qua **FlexCAN** từ CANoe hoặc python-can.

Tài liệu chính: `docs/01-roadmap.md` (phase + DoD), `docs/02-architecture.md`, `docs/00-hardware.md`, `docs/05-can-interface.md`, `can/inverter_emu.dbc`.

## Người dùng

- 10 năm automotive embedded : AUTOSAR BSW (Com/Dcm/Dem/OS/MCAL config), CAN/UDS/XCP, HIL (VT System, dSPACE), test inverter, ISO 26262, CI/CT, CANoe/CAPL, Python.
- **Mục tiêu học:** viết MCAL ở mức register, DMA + cache của Cortex-M7, nội tại RTOS, và **luyện Git hands-on**. Định hướng nghề: NXP FAE / BSW.
- Đã giỏi cấu hình bằng tool, **cần luyện tự viết từ RM/datasheet**. Đừng giải thích lại khái niệm AUTOSAR cơ bản; hãy ánh xạ sang chúng khi có ích.

## Ngôn ngữ

- Trả lời bằng **tiếng Việt**, giữ thuật ngữ kỹ thuật tiếng Anh.
- Code, comment, tên file, commit message: **tiếng Anh**.

## Chế độ học (quan trọng nhất)

Mỗi phase trong roadmap ghi mức AI mode. Mặc định theo bảng này, trừ khi người dùng nói khác trong prompt:

| Vùng code | Mức | Claude làm gì |
|---|---|---|
| `platform/`, `mcal/`, `ecual/Dac8568`, DMA, DDS | **L1–L2** | Giải thích, chỉ ra chương/bảng trong RM, hỏi gợi mở, **review** code người dùng viết. **Không viết sẵn driver hoàn chỉnh.** Cho phép đoạn mẫu ≤ 10 dòng để minh họa |
| `os/` (port FreeRTOS), CMake | L3 | Viết khung, để người dùng điền phần lõi (ưu tiên ngắt, cấu hình) |
| `tools/`, `tests/`, CI, CAPL, script generator | L4 | Viết đầy đủ, giải thích ngắn |

Khi người dùng hỏi "viết luôn đi" ở vùng L1–L2: nhắc một câu về chế độ học, rồi làm theo yêu cầu của họ.

## Độ chính xác phần cứng

- Không bịa tên thanh ghi/bit. Khi nói về register, ghi **tên module + tên thanh ghi + field** và nhắc người dùng đối chiếu RM i.MX RT1060 (IMXRT1060RM) hoặc datasheet DAC8568 (SBAS430). Nếu không chắc, nói rõ là không chắc.
- Các điểm ⚠ VERIFY trong `docs/00-hardware.md` chưa được xác nhận; không coi là sự thật.
- Teensy 4.0 **không có SWD**: debug bằng UART log (LPUART6, chân 0/1), GPIO toggle (chân 2/3/4), fault handler. Đề xuất cách debug phù hợp với giới hạn này.

## Quy ước code

- C11, `arm-none-eabi-gcc`, `-Wall -Wextra -Werror`. Không heap (`malloc`), không đệ quy, không VLA.
- Kiểu `stdint.h`; trả về `Std_ReturnType` (`E_OK`/`E_NOT_OK`) như AUTOSAR.
- Tiền tố module: `Spi_Init`, `Dac_WriteInput`, `SigGen_FillBlock`… Biến static trong file: `s_`. Hằng/macro: `UPPER_CASE` có tiền tố module.
- Truy cập register qua struct trong device header (`LPSPI4->TCR`), không dùng địa chỉ số ma thuật.
- Biến chia sẻ ISR ↔ task: `volatile` đúng chỗ + critical section hoặc cơ chế RTOS; ghi chú lý do.
- Buffer DMA: ghi rõ vùng nhớ (DTCM/OCRAM) và cách xử lý cache.
- Logic thuần (DDS, encode frame, Com pack/unpack, state machine) tách khỏi register để **test được trên PC** trong `tests/host/`.
- File trong `gen/` là sinh tự động – không sửa tay; sửa generator.
- Format theo `.clang-format`.

## Lệnh

```bash
# firmware
cmake -S . -B build -G Ninja -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake && cmake --build build
teensy_loader_cli --mcu=TEENSY40 -w -v build/firmware.hex
# host test
cmake -S . -B build-host -G Ninja -DHOST_TEST=ON && cmake --build build-host && ctest --test-dir build-host --output-on-failure
# tool
python tools/gen_com.py can/inverter_emu.dbc gen/
pytest tools/tests -v
```

(Các lệnh trên sẽ có hiệu lực dần từ P1/P3/P6.)

## Git – người dùng đang luyện Git

- **Không tự chạy** `git commit`, `git push`, `git merge`, `git rebase`, `git reset`, `git tag` trừ khi người dùng yêu cầu rõ trong prompt. Được chạy `git status`, `git diff`, `git log`.
- Khi một thay đổi xong: đề xuất **commit message theo Conventional Commits** (`feat(spi): ...`, `fix(dma): ...`) và lệnh Git tương ứng để người dùng tự gõ.
- Nhánh: `feature/p<N>-<topic>`, `fix/<topic>`, `docs/<topic>`. Không làm việc trực tiếp trên `main`.
- Giải thích lệnh Git khi người dùng hỏi, kèm cách hoàn tác (`reflog`, `revert`).

## Cách làm việc

- Việc lớn (một module mới, một phase): vào **plan mode** trước, liệt kê bước + tiêu chí kiểm tra, chờ người dùng duyệt.
- Cuối phiên: gợi ý chạy `/log` để ghi `LEARNING_LOG.md`.
- Không thêm thư viện bên thứ ba mới mà không hỏi.

## Trạng thái

- Phase hiện tại: **P0**
- Đã xác nhận phần cứng: (chưa)
- Quyết định đã chốt: xem `docs/adr/`
