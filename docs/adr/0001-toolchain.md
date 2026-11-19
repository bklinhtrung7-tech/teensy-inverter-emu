# ADR-0001: Toolchain và cách build firmware

- **Trạng thái:** Accepted
- **Ngày đề xuất:** 2026-11-17
- **Ngày chấp nhận:** 2026-11-19 (P0)

## Bối cảnh

Mục tiêu là học MCU ở mức register (MCAL, DMA, cache) và làm được một sản phẩm chạy ổn định. Teensy 4.0 không lộ cổng SWD/JTAG, chỉ nạp được qua bootloader HalfKay bằng file `.hex`.

## Các lựa chọn

| Lựa chọn | Ưu | Nhược |
|---|---|---|
| A. Arduino / Teensyduino | Chạy ngay, thư viện SPI/CAN có sẵn | Che hết phần cứng → học được ít; khó kiểm soát DMA/ưu tiên ngắt |
| B. PlatformIO + framework Teensy | Build bằng CLI, vẫn dễ | Vẫn là core Arduino bên dưới |
| C. **CMake + arm-none-eabi-gcc + device header NXP (CMSIS) + driver tự viết** | Học đúng thứ cần học; gần với dự án MCAL thật và với MCUXpresso của NXP | Phải tự làm startup, linker, boot header (FCB/IVT); bring-up khó hơn |
| D. Zephyr (board `teensy40`) | Device tree, driver CAN/SPI có sẵn | Học Zephyr nhiều hơn học chip; trừu tượng dày |

## Quyết định (Accepted)

- **C** cho firmware chính.
- **A** chỉ dùng cho smoke test phần cứng ở P0 (`tools/smoke_test/`), để tách bạch "phần cứng lỗi" với "firmware lỗi".
- Boot header và linker script tham khảo core Teensy 4 của PJRC; ghi rõ nguồn và license trong file.
- Debug: UART log + GPIO + logic analyzer + fault handler. Nếu cần step-debug thì mua thêm board MIMXRT1060-EVK (cùng họ chip, có debugger onboard).
- Output build: ELF → `arm-none-eabi-objcopy -O ihex` → `build/firmware.hex`, nạp bằng `teensy_loader_cli --mcu=TEENSY40`.

## Phiên bản công cụ (đo ngày 2026-11-19, Ubuntu)

| Công cụ | Phiên bản | Dùng cho |
|---|---|---|
| `arm-none-eabi-gcc` (apt `gcc-arm-none-eabi`) | 14.2.1 (`15:14.2.rel1-1`) | Firmware chính |
| CMake | 4.2.3 | Build |
| Ninja | 1.13.2 | Build |
| `teensy_loader_cli` (apt `teensy-loader-cli`) | 2.2-1.1build1 | Nạp `.hex` qua HalfKay |
| Teensyduino `teensy-compile` (gcc đi kèm) | 15.2.1 (Teensyduino 1.62.0, Arduino IDE 2.3.11) | **Chỉ** smoke test trong `tools/smoke_test/` |

- Hai toolchain tách biệt: `cmake/arm-none-eabi.cmake` phải dùng gcc của apt (`/usr/bin`), không dùng gcc trong `~/.arduino15`.
- CI (khi có) phải dùng cùng bản 14.2.rel1 để kết quả build giống máy local.

## Việc còn mở (chốt trong ADR sau)

- **Nguồn device header** (`MIMXRT1062.h` + CMSIS core): lấy từ đâu (MCUXpresso SDK / repo `mcux-sdk`), license, đặt ở thư mục nào (ví dụ `third_party/`). Cần chốt trước khi bắt đầu P1.
- **License core Teensy 4 của PJRC**: TODO – đọc file license trong `PaulStoffregen/cores/teensy4` và ghi lại trước khi tham khảo `bootdata.c` / `imxrt1062.ld`.

## Hệ quả

- P1 dài hơn (3 tuần) nhưng các phase sau không bị "hộp đen".
- Phải tự viết test trên PC cho các phần logic thuần (DDS, encode frame, Com) vì không debug được trên target.
- Nếu sau P1 vẫn chưa boot được: dùng tạm startup của core Teensy (không dùng phần Arduino) để không kẹt, ghi lại vào ADR mới.
