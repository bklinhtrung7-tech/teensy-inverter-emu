# 02 – Kiến trúc

## 1. Phân tầng (AUTOSAR-lite)

Không dùng AUTOSAR thật, nhưng giữ **ranh giới tầng** giống AUTOSAR để vừa quen tay vừa dễ kể khi phỏng vấn.

```
┌───────────────────────────────────────────────────────────────┐
│ app/        Ctrl (state machine, ramp, timeout)   SigGen (DDS) │
│             FaultInj (E1)   Model (E9)                         │
├───────────────────────────────────────────────────────────────┤
│ services/   Com (signal, sinh từ DBC)   Log   Diag (E3/E4)     │
├───────────────────────────────────────────────────────────────┤
│ ecual/      Dac8568 (driver thiết bị)   CanIf                  │
├───────────────────────────────────────────────────────────────┤
│ mcal/       Mcu  Port  Dio  Uart  Gpt(PIT)  Spi(LPSPI)          │
│             Dma(eDMA+DMAMUX)  Can(FlexCAN)  Wdg  Adc            │
├───────────────────────────────────────────────────────────────┤
│ os/         bare-metal (P1–P4)  →  FreeRTOS (P5+)              │
│ platform/   startup, linker, boot header (FCB/IVT), CMSIS, device header │
└───────────────────────────────────────────────────────────────┘
```

**Quy tắc phụ thuộc:** chỉ gọi xuống, không gọi lên. Gọi ngược lên (callback, notification) phải qua con trỏ hàm hoặc hàm `*_Cbk` khai báo ở tầng trên, giống AUTOSAR.

## 2. Cây thư mục dự kiến

```
teensy-inverter-emu/
├── CLAUDE.md  README.md  LEARNING_LOG.md
├── CMakeLists.txt
├── cmake/                 arm-none-eabi.cmake, host.cmake
├── platform/teensy40/     startup.c, boot_header.c, imxrt1062.ld, system.c
├── third_party/           CMSIS, device header MIMXRT1062.h, FreeRTOS-Kernel (submodule)
├── mcal/<Module>/         <Module>.c/.h, <Module>_Cfg.h
├── ecual/  services/  app/  os/
├── gen/                   file sinh tự động từ DBC (không sửa tay)
├── tests/host/            unit test chạy trên PC
├── tools/                 gen_lut.py, gen_com.py, can_ctrl.py, tests/ (pytest), smoke_test/ (Arduino)
├── canoe/                 cấu hình CANoe, panel, CAPL
├── can/                   inverter_emu.dbc
└── docs/                  notes/, adr/, img/, datasheets/ (không commit PDF)
```

## 3. Đường đi tín hiệu

### 3.1 DDS (Direct Digital Synthesis)

```
phase_acc (uint32) += phase_inc          phase_inc = f_out × 2^32 / fs
idx_a = (phase_acc              ) >> 22   (LUT 1024 điểm → lấy 10 bit cao)
idx_b = (phase_acc - 0x55555555 ) >> 22   (−120°)
idx_c = (phase_acc - 0xAAAAAAAA ) >> 22   (−240°)
I_x   = amp × LUT[idx_x]                  (Q15)
code  = clamp(V2CODE(V_offset + I_x × sensitivity), 0, 65535)
```

- Đổi `phase_inc` giữa chừng thì **pha liên tục**, không giật sóng.
- Độ phân giải tần số = fs / 2³² (≈ 23 µHz ở 100 kHz).
- Tùy chọn: nội suy tuyến tính giữa 2 điểm LUT để giảm méo (đo THD trước/sau là bài tập hay).

### 3.2 Ba pha đổi cùng lúc

Mỗi mẫu gửi 3 frame:

| Frame | Control | Kênh | Ý nghĩa |
|---|---|---|---|
| 1 | `0000` | A | ghi input register A |
| 2 | `0000` | B | ghi input register B |
| 3 | `0010` | C | ghi input C **và cập nhật tất cả** → A, B, C ra cùng lúc |

Cách khác: dùng chân LDAC phần cứng. Phải hiểu vì sao cần cập nhật đồng thời (nếu lệch nhau vài µs thì tại mỗi thời điểm Ia+Ib+Ic ≠ 0, ECU có thể báo lỗi plausibility).

## 4. Pipeline DMA (P4)

```
 PIT ch0 (mỗi Ts) ──trigger──► DMAMUX ch0 (TRIG=1, nguồn always-on) ──► eDMA ch0
                                                                           │ minor loop: 12 byte (3 × uint32)
                                                                           ▼
        txbuf[0][N×3] │ txbuf[1][N×3]   ──────────────────────────────►  LPSPI4->TDR  ──► DAC
                ▲ half IRQ / major IRQ (vòng tròn: SLAST = −kích thước buffer)
                │
        SigGen điền nửa buffer vừa phát xong (DDS)
```

Các điểm phải hiểu (⚠ VERIFY từng mục trong RM i.MX RT1060):
- Kênh DMA nào hỗ trợ **periodic trigger** từ PIT.
- LPSPI: TCR cấu hình một lần (FRAMESZ = 31, không bật CONT) để mỗi word ghi vào TDR là một frame có SYNC riêng. Độ sâu TX FIFO.
- **Cache:** eDMA không đi qua D-cache của M7. Buffer ở OCRAM có cache thì CPU ghi xong phải **clean** trước khi DMA đọc. Buffer ở DTCM thì không cần.
- **Độ trễ tham số:** setpoint mới có hiệu lực sau tối đa 1 nửa buffer (N/fs = 64/100 kHz = 0.64 ms), chấp nhận được.
- **Underrun:** nếu CPU chưa điền xong nửa buffer khi DMA quay lại → bật cờ lỗi (đếm bằng cờ "đã điền" mỗi nửa).

## 5. Ngân sách thời gian

| Tham số | Giá trị | Ghi chú |
|---|---|---|
| SCK | 25 MHz (bắt đầu 1–10 MHz trên breadboard) | Max DAC 50 MHz |
| 1 frame | 32 bit / 25 MHz = 1.28 µs + delay CS ≈ 1.5 µs | |
| 3 frame/mẫu | ≈ 4.5 µs | |
| fs mặc định | 100 kHz (Ts = 10 µs) | Còn dư ~5.5 µs/chu kỳ |
| Settling DAC | vài µs | ⚠ VERIFY trong datasheet; fs cao hơn không có ích nếu DAC chưa settle |
| f_out tối đa | 2 kHz | ≥ 50 mẫu/chu kỳ ở 100 kHz |
| LUT | 1024 × int16 = 2 KB | đặt ở DTCM |

Motor xe điện tốc độ cao có tần số điện vài trăm Hz đến ~1–2 kHz nên dải trên đủ dùng.

## 6. RTOS (P5+)

| Task / ISR | Ưu tiên | Chu kỳ / kích hoạt | Việc làm |
|---|---|---|---|
| ISR DMA half/full | cao (≥ mức syscall) | mỗi N mẫu | `vTaskNotifyGiveFromISR(SigGen)` |
| ISR FlexCAN RX | cao | sự kiện | copy frame vào queue |
| `T_SigGen` | cao nhất trong các task | notification | điền nửa buffer, kiểm tra underrun |
| `T_Com` | cao | 5 ms | xử lý queue RX, unpack signal, giám sát timeout |
| `T_Ctrl` | trung bình | 10 ms | state machine, ramp → ghi setpoint double-buffer |
| `T_Status` | thấp | 100 ms | gửi status, CPU load |
| Idle hook | thấp nhất | – | đếm idle → CPU load, đá watchdog |

**Ánh xạ sang AUTOSAR OS:** task ↔ task (basic/extended), task notification ↔ event, ISR có gọi API ↔ ISR category 2, ISR vượt mức syscall ↔ ISR category 1, mutex/critical section ↔ resource (priority ceiling), software timer ↔ alarm.

## 7. State machine `Ctrl`

| Từ | Điều kiện | Sang |
|---|---|---|
| (power-on) | – | `INIT` |
| `INIT` | init SPI/DAC/DMA/CAN OK | `STANDBY` |
| `INIT` | init lỗi | `FAULT` |
| `STANDBY` | `Enable = 1` và lệnh hợp lệ | `RUN` |
| `RUN` | `Enable = 0`, timeout lệnh, lỗi E2E | `RAMP_DOWN` |
| `RAMP_DOWN` | biên độ đã về 0 | `STANDBY` |
| bất kỳ | lỗi DMA / SPI / underrun liên tiếp | `FAULT` (output về offset ngay, không ramp) |
| `FAULT` | lệnh reset lỗi qua CAN (hoặc reset MCU) | `INIT` |

Safe state = output về **offset (0 A)**, không phải 0 V (0 V nghĩa là "−max A" với cảm biến có offset).
