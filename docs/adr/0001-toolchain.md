# ADR-0001: Toolchain và cách build firmware

- **Trạng thái:** Đề xuất (chốt ở P0)
- **Ngày:** 2026-11-17

## Bối cảnh

Mục tiêu là học MCU ở mức register (MCAL, DMA, cache) và làm được một sản phẩm chạy ổn định. Teensy 4.0 không lộ cổng SWD/JTAG, chỉ nạp được qua bootloader HalfKay bằng file `.hex`.

## Các lựa chọn

| Lựa chọn | Ưu | Nhược |
|---|---|---|
| A. Arduino / Teensyduino | Chạy ngay, thư viện SPI/CAN có sẵn | Che hết phần cứng → học được ít; khó kiểm soát DMA/ưu tiên ngắt |
| B. PlatformIO + framework Teensy | Build bằng CLI, vẫn dễ | Vẫn là core Arduino bên dưới |
| C. **CMake + arm-none-eabi-gcc + device header NXP (CMSIS) + driver tự viết** | Học đúng thứ cần học; gần với dự án MCAL thật và với MCUXpresso của NXP | Phải tự làm startup, linker, boot header (FCB/IVT); bring-up khó hơn |
| D. Zephyr (board `teensy40`) | Device tree, driver CAN/SPI có sẵn | Học Zephyr nhiều hơn học chip; trừu tượng dày |

## Quyết định (đề xuất)

- **C** cho firmware chính.
- **A** chỉ dùng cho smoke test phần cứng ở P0 (`tools/smoke_test/`), để tách bạch "phần cứng lỗi" với "firmware lỗi".
- Boot header và linker script tham khảo core Teensy 4 của PJRC; ghi rõ nguồn và license trong file.
- Debug: UART log + GPIO + logic analyzer + fault handler. Nếu cần step-debug thì mua thêm board MIMXRT1060-EVK (cùng họ chip, có debugger onboard).

## Hệ quả

- P1 dài hơn (3 tuần) nhưng các phase sau không bị "hộp đen".
- Phải tự viết test trên PC cho các phần logic thuần (DDS, encode frame, Com) vì không debug được trên target.
- Nếu sau P1 vẫn chưa boot được: dùng tạm startup của core Teensy (không dùng phần Arduino) để không kẹt, ghi lại vào ADR mới.
