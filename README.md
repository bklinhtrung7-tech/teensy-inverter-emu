# teensy-inverter-emu

**Giả lập dòng điện 3 pha của inverter** trên Teensy 4.0 (NXP i.MX RT1062, Cortex-M7 600 MHz) + DAC8568 (16-bit, 8 kênh, SPI).
Biên độ và tần số dòng được điều khiển qua CAN từ CANoe / python-can.

Sản phẩm cuối là một **"HIL-lite current sensor emulator"**: thay cho cảm biến dòng pha thật, cấp tín hiệu analog cho ECU điều khiển motor để test chức năng và chẩn đoán (offset, mất pha, tổng 3 dòng ≠ 0…).

> Mục đích chính: **học**. Viết lại từ đầu các thứ mà công cụ AUTOSAR thường che đi (MCAL, DMA, OS, Com), luyện Git thực tế và luyện cách làm việc với AI (Claude Code + Claude desktop/Cowork) như một kỹ sư dẫn dắt, không phải người chép code.

---

## Kiến trúc một hình

```
 CANoe / python-can                     Teensy 4.0 (i.MX RT1062)                          DAC8568
 ┌──────────────┐  CAN 500k  ┌──────────────────────────────────────────────┐  LPSPI4   ┌──────────┐
 │ Panel: Freq, │──[XCVR]────┤ FlexCAN1 → CanIf → Com → Ctrl (state, ramp)   │  32-bit   │ VOUT A ──┼─► Ia
 │ Amp, Enable, │            │                        │                     │  frames   │ VOUT B ──┼─► Ib
 │ Fault inject │◄───────────┤ Status 100 ms          ▼                     │──────────►│ VOUT C ──┼─► Ic
 └──────────────┘            │ PIT ─► DMAMUX ─► eDMA ─► LPSPI TDR (0% CPU)  │           │ D..H     │  (mở rộng:
                             │            ▲ half/full IRQ                   │           └──────────┘   resolver, Vdc…)
                             │   SigGen task: DDS + LUT → ping-pong buffer  │
                             │   FreeRTOS                                    │
                             └──────────────────────────────────────────────┘
```

Chi tiết: [`docs/02-architecture.md`](docs/02-architecture.md)

## Lộ trình (tóm tắt)

| Phase | Nội dung | Tag Git |
|---|---|---|
| P0 | Toolchain, repo, smoke test DAC bằng Arduino | `v0.1.0` |
| P1 | Bare-metal: startup, linker, boot header, clock, GPIO, UART log | `v0.2.0` |
| P2 | Driver LPSPI + DAC8568 (polling) | `v0.3.0` |
| P3 | DDS + LUT, 3 pha chạy bằng ngắt PIT, unit test trên PC | `v0.4.0` |
| P4 | **eDMA + DMAMUX + PIT trigger**, ping-pong buffer, D-cache | `v0.5.0` |
| P5 | Port sang **FreeRTOS** | `v0.6.0` |
| P6 | **FlexCAN** + Com từ DBC + điều khiển từ CANoe, timeout → safe state | `v1.0.0` |
| P7+ | Mở rộng: fault injection, dq/resolver, E2E, UDS, XCP, CI, Rust… | `v1.x` |

Chi tiết và tiêu chí hoàn thành: [`docs/01-roadmap.md`](docs/01-roadmap.md)

## Bắt đầu từ bộ file này

1. Giải nén vào thư mục làm việc, ví dụ `~/projects/teensy-inverter-emu`.
2. Làm **Git bài 1–2** trong [`docs/03-git-workflow.md`](docs/03-git-workflow.md): `git init`, commit đầu, tạo repo GitHub, bảo vệ `main`.
3. Mở terminal trong thư mục → `claude` → prompt đầu tiên: *"Đọc CLAUDE.md và docs/01-roadmap.md, tóm tắt lại dự án và liệt kê việc của P0 dưới dạng checklist"* để kiểm tra Claude hiểu đúng bối cảnh.
4. Mở Claude desktop (Cowork), cấp quyền vào cùng thư mục, tải tài liệu theo [`docs/datasheets/README.md`](docs/datasheets/README.md) rồi bắt đầu tóm tắt chương LPSPI.
5. Chỉnh `CLAUDE.md` mục "Người dùng" và "Chế độ học" cho đúng ý bạn. File này là của bạn.

## Tài liệu

| File | Nội dung |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | Bộ nhớ dự án cho Claude Code: bối cảnh, quy tắc, chế độ học |
| [`docs/00-hardware.md`](docs/00-hardware.md) | BOM, đấu dây, các điểm phải tự kiểm tra trên board |
| [`docs/01-roadmap.md`](docs/01-roadmap.md) | Phase, tiêu chí xong, ý tưởng mở rộng |
| [`docs/02-architecture.md`](docs/02-architecture.md) | Tầng phần mềm, pipeline DMA, ngân sách thời gian, RTOS |
| [`docs/03-git-workflow.md`](docs/03-git-workflow.md) | Quy ước Git + 12 bài tập Git gắn vào từng phase |
| [`docs/04-ai-workflow.md`](docs/04-ai-workflow.md) | Dùng Claude Code và Claude desktop (Cowork) thế nào cho việc học |
| [`docs/05-can-interface.md`](docs/05-can-interface.md) | Ma trận CAN, timeout, CANoe/python-can |
| [`can/inverter_emu.dbc`](can/inverter_emu.dbc) | DBC cho CANoe / cantools |
| [`docs/adr/`](docs/adr/) | Architecture Decision Records |
| [`LEARNING_LOG.md`](LEARNING_LOG.md) | Nhật ký mỗi buổi học |

## Build & flash (sẽ hoàn thiện ở P1)

```bash
cmake -S . -B build -G Ninja -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake
cmake --build build
teensy_loader_cli --mcu=TEENSY40 -w -v build/firmware.hex   # nhấn nút trên Teensy nếu cần
ctest --test-dir build-host                                 # unit test chạy trên PC (từ P3)
```

## Trạng thái

- Phase hiện tại: **P0**
- Phần cứng có: Teensy 4.0, board DAC8568 V1.1
- Sẽ mua: USB-UART 3.3V, logic analyzer, transceiver CAN (P6)
