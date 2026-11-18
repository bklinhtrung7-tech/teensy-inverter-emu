# 00 – Phần cứng

> Các dòng có **⚠ VERIFY** là điểm phải tự đo hoặc đối chiếu datasheet/schematic trước khi cấp nguồn. Đừng tin AI (kể cả file này) hơn datasheet.

## 1. BOM

| Món | Có chưa | Dùng ở phase | Ghi chú |
|---|---|---|---|
| Teensy 4.0 | ✅ | P0 | i.MX RT1062, 2 MB flash QSPI (Winbond), 1 MB RAM |
| Board DAC8568 V1.1 (8 kênh) | ✅ | P0 | Xem mục 3, phải đọc schematic của board |
| Cáp micro-USB (có dây data) | ? | P0 | |
| **USB-UART 3.3V** (CP2102 / CH340 / FT232) | ❌ | **P1** | Bare-metal không có USB CDC → log qua LPUART. Rẻ, mua ngay |
| **Logic analyzer** 8 kênh 24 MHz + PulseView/Saleae Logic | ❌ | P1 | Soi SPI frame, đo jitter bằng GPIO toggle. Gần như bắt buộc |
| Oscilloscope (USB scope 2 kênh hoặc mượn) | ❌ | P3 | Xem sóng sine 3 pha. Tạm thời có thể đọc lại bằng ADC của Teensy (mục 5) |
| Breadboard, dây dupont ngắn, điện trở, tụ 100 nF | ? | P0 | Dây SPI **càng ngắn càng tốt** (< 10 cm) |
| Module transceiver CAN 3.3V (SN65HVD230) | ❌ | P6 | CAN classic. Muốn CAN-FD sau này: TJA1462 / MCP2562FD |
| 2 × điện trở 120 Ω | ❌ | P6 | Termination 2 đầu bus |
| Interface CAN phía PC | ❌/? | P6 | **CANoe cần interface Vector (VN16xx)**. Ở nhà không có Vector → CANable 2.0 + python-can. Xem `docs/05` |

## 2. Teensy 4.0 – chân dùng trong dự án

⚠ VERIFY tất cả với **pinout card chính thức của PJRC** và chương IOMUXC trong RM i.MX RT1060.

| Teensy pin | Chức năng i.MX RT1062 | Dùng cho |
|---|---|---|
| 13 | LPSPI4_SCK | DAC SCLK (chân này cũng nối LED onboard → LED sẽ nháy theo SPI, bình thường) |
| 11 | LPSPI4_SDO | DAC DIN |
| 12 | LPSPI4_SDI | Không dùng (DAC8568 không có SDO) |
| 10 | LPSPI4_PCS0 | DAC **SYNC** (chip select, active low) |
| 9 | GPIO | DAC **LDAC** (tùy chọn – hoặc nối GND cố định, xem mục 3) |
| 8 | GPIO | DAC **CLR** (tùy chọn – hoặc kéo lên mức cao cố định) |
| 0 / 1 | LPUART6 RX / TX | Log ra USB-UART (Serial1 trong Arduino) |
| 2, 3, 4 | GPIO | **Chân đo thời gian** (toggle trong ISR/task → logic analyzer) |
| 22 / 23 | FLEXCAN1 TX / RX | CAN tới transceiver (P6) |
| 14–21 (A0–A7) | ADC1/ADC2 | Đọc lại tín hiệu DAC qua cầu phân áp (mở rộng E7) |

## 3. DAC8568 – những gì cần biết

Nguồn: datasheet TI **SBAS430F** (DAC7568/DAC8168/DAC8568). Tải về và để vào `docs/datasheets/` (đã nằm trong `.gitignore`, PDF không commit).

- Giao tiếp: SPI 3 dây (SYNC, SCLK, DIN), **tối đa 50 MHz**. Thanh ghi dịch **32 bit**, dữ liệu lấy mẫu ở **cạnh xuống** SCLK, DAC cập nhật sau xung thứ 32 → thường là **SPI mode 1** (CPOL=0, CPHA=1). ⚠ VERIFY bằng logic analyzer.
- Khung 32 bit (⚠ VERIFY với Table 1 của datasheet):

  | Bit | 31–28 | 27–24 | 23–20 | 19–4 | 3–0 |
  |---|---|---|---|---|---|
  | Trường | Prefix `0000` | Control C3..C0 | Address A..H (0–7), `1111` = tất cả | Data 16 bit | Feature |

- Control hay dùng (⚠ VERIFY bảng lệnh):
  - `0000` ghi input register kênh n (chưa ra output)
  - `0010` ghi input register n **và cập nhật tất cả kênh** (software LDAC) → dùng cho kênh cuối cùng để **3 pha đổi cùng lúc**
  - `0011` ghi và cập nhật riêng kênh n
  - `0111` software reset
  - `1000` cấu hình internal reference – `0x08000001` bật ref 2.5 V (static mode)
- **Internal reference tắt mặc định** khi cấp nguồn → nếu board không có ref ngoài thì output = 0 cho tới khi gửi lệnh bật ref. Lỗi kinh điển ở buổi đầu.
- Full-scale là 2.5 V hoặc 5 V tùy **grade** của chip (đọc ký hiệu in trên chip rồi tra bảng Ordering/Device Comparison). Muốn 5 V full-scale thì AVDD phải đủ 5 V.
- Nguồn AVDD 2.7–5.5 V.

### ⚠ Checklist đọc schematic board V1.1 (làm ở P0, ghi kết quả vào bảng dưới)

| Câu hỏi | Kết quả |
|---|---|
| Chip là grade nào (ký hiệu in trên chip)? Full-scale 2.5 V hay 5 V? | |
| AVDD lấy từ đâu (5 V, 3.3 V, có LDO trên board không)? | |
| Có reference ngoài không, hay dùng internal ref? | |
| LDAC, CLR được kéo lên/xuống sẵn hay để hở? | |
| Ngưỡng logic: nếu AVDD = 5 V thì VIH tối thiểu là bao nhiêu? Có > 3.3 V × biên an toàn không? (tra *Electrical Characteristics*) | |
| → Có cần level shifter (ví dụ 74AHCT125) không? | |
| Output có buffer / op-amp / cầu phân áp sau DAC không? | |

> Quy tắc: **chưa điền xong bảng này thì chưa cắm board DAC vào Teensy.** Teensy 4.0 chịu 3.3 V, **không chịu 5 V** trên chân I/O.

## 4. Đấu dây P0–P5

```
Teensy 4.0                 DAC8568 board
  3V3  ───────────────────  VDD logic / AVDD (nếu board dùng 3.3 V)   ⚠ VERIFY
  VIN (5V USB) ───────────  AVDD (nếu board cần 5 V)                  ⚠ VERIFY
  GND  ───────────────────  GND   (dây GND ngắn, đặt sát dây SCLK)
  13   ───────────────────  SCLK
  11   ───────────────────  DIN
  10   ───────────────────  SYNC
  9    ───────────────────  LDAC  (hoặc LDAC → GND)
  8    ───────────────────  CLR   (hoặc CLR → mức cao)

Teensy 0 (RX1) ◄── TX  USB-UART
Teensy 1 (TX1) ──► RX  USB-UART      GND chung
```

## 5. Khi nối output ra thiết bị khác

- **Đọc lại bằng ADC của Teensy:** ADC chỉ chịu 0–3.3 V → nếu DAC ra 0–5 V phải dùng cầu phân áp (ví dụ 10k/15k → 0–3 V) + tụ 10 nF.
- **Cấp cho ECU thật:** chung GND, thêm điện trở nối tiếp 100–470 Ω, kiểm tra dải điện áp đầu vào ADC của ECU (cảm biến dòng thường ra 0.5–4.5 V quanh offset 2.5 V).
- Không bao giờ nối output DAC vào mạch công suất.

## 6. Khởi động lần đầu (P0)

1. Cấp nguồn Teensy một mình qua USB, chạy Blink (Arduino) → board sống.
2. Đo AVDD và ref trên board DAC bằng đồng hồ (chưa nối SPI).
3. Nối SPI, chạy sketch Arduino: bật internal ref → ghi mid-scale (`0x8000`) vào kênh A → đo ~½ full-scale.
4. Ghi 0x0000 / 0xFFFF → đo 0 V / full-scale. Ghi kết quả vào `LEARNING_LOG.md`.
5. Chụp frame SPI bằng logic analyzer, đối chiếu từng bit với bảng khung 32 bit ở trên.
