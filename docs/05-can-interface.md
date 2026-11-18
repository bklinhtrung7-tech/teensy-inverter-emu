# 05 – Giao tiếp CAN

## 1. Bus vật lý

```
 Teensy 4.0               SN65HVD230 (3.3 V)          PC
  pin 22 CAN1_TX ───────► D                 CANH ──┬───────────┬── CANH ─┐
  pin 23 CAN1_RX ◄─────── R                 CANL ──┼───────────┼── CANL ─┤ VN16xx (CANoe)
  3V3 ────────────────── VCC                       │           │         │ hoặc CANable 2.0
  GND ────────────────── GND ──────────────────────┼── GND ────┼─────────┘
                          Rs → GND (high-speed)  120 Ω       120 Ω
```

- 500 kbit/s, CAN classic ở P6. CAN-FD (FlexCAN3) để dành cho mở rộng E10.
- Termination: 120 Ω ở **mỗi đầu** bus. Đo CANH–CANL khi tắt nguồn ≈ 60 Ω là đúng.
- Một số module SN65HVD230 có sẵn điện trở 120 Ω trên board → kiểm tra trước khi thêm.
- ⚠ VERIFY chân CAN1 với pinout PJRC.

## 2. Ma trận bản tin

Nguồn chuẩn là `can/inverter_emu.dbc` (Intel byte order). Bảng dưới chỉ để đọc nhanh.

### 0x200 `IEMU_Cmd` – Tester → IEMU, cyclic 20 ms

| Signal | Start | Len | Factor | Đơn vị | Ghi chú |
|---|---|---|---|---|---|
| `IEMU_Cmd_CRC` | 0 | 8 | 1 | | E2E CRC8, gửi 0 cho tới khi làm E3 |
| `IEMU_Cmd_Alive` | 8 | 4 | 1 | | tăng mỗi frame |
| `IEMU_Cmd_Enable` | 12 | 1 | | | |
| `IEMU_Cmd_Mode` | 13 | 3 | | | 0 ABC_Sine, 1 DQ (E2), 2 DC_Test, 3 Freq_Sweep |
| `IEMU_Cmd_Freq` | 16 | 16 | 0.1 | Hz | 0–2000 |
| `IEMU_Cmd_Amp` | 32 | 16 | 0.01 | A | biên độ đỉnh, 0–500 |
| `IEMU_Cmd_Dir` | 48 | 1 | | | 0 ABC, 1 ACB (đảo chiều quay) |
| `IEMU_Cmd_FaultReset` | 49 | 1 | | | cạnh lên → thoát FAULT |
| `IEMU_Cmd_RampRate` | 56 | 8 | 10 | A/s | 0 = không ramp |

### 0x201 `IEMU_FaultCmd` – Tester → IEMU, event (dùng ở E1)

| Signal | Start | Len | Ghi chú |
|---|---|---|---|
| `IEMU_Flt_Type` | 0 | 4 | None, Offset, Gain, PhaseLoss, StuckAt, Noise, Harmonic, Imbalance |
| `IEMU_Flt_Phase` | 4 | 2 | A, B, C, All |
| `IEMU_Flt_Enable` | 6 | 1 | |
| `IEMU_Flt_Value` | 8 | 16 (signed, 0.01) | ý nghĩa theo Type |
| `IEMU_Flt_HarmOrder` | 24 | 8 | bậc sóng hài |

### 0x300 `IEMU_Status` – IEMU → Tester, cyclic 100 ms

`State` (INIT/STANDBY/RUN/RAMP_DOWN/FAULT), các cờ lỗi `CmdTimeout`, `SpiErr`, `DmaErr`, `Underrun`, `E2EErr`, `FaultActive`, giá trị thực `ActFreq`, `ActAmp`, `CpuLoad`, `Alive`.

## 3. Hành vi phía IEMU

| Tình huống | Phản ứng | Thời gian |
|---|---|---|
| Không nhận `IEMU_Cmd` | `CmdTimeout = 1`, RUN → RAMP_DOWN | 200 ms (10 chu kỳ) |
| Alive counter không đổi / nhảy cóc (từ E3) | đếm lỗi, quá ngưỡng → như timeout | |
| CRC sai (từ E3) | bỏ frame, `E2EErr = 1` | |
| Giá trị ngoài dải | clamp về dải, không báo lỗi | |
| Bus-off | recovery tự động, ghi log | |

## 4. Phía PC

### Có CANoe + interface Vector (VN16xx)

- Tạo cấu hình `canoe/iemu.cfg`, thêm `can/inverter_emu.dbc`, node `Tester` là simulation node.
- Panel: slider Freq, Amp, RampRate; checkbox Enable; nút FaultReset; hiển thị State, ActFreq, ActAmp, CpuLoad.
- CAPL `canoe/Tester.can`: timer 20 ms gửi `IEMU_Cmd` có alive counter; phím tắt để **ngưng gửi** (test timeout).
- Test module (CAPL hoặc Python qua CANoe COM) cho các tình huống ở mục 3 – liên hệ trực tiếp với kinh nghiệm VT/CANoe test.

### Không có Vector (ở nhà)

- **CANable 2.0** (firmware candleLight → SocketCAN trên Linux; slcan trên Windows) + `python-can` + `cantools`.
- `tools/can_ctrl.py` (CLI) và `tools/tests/test_can.py` (pytest) – nhờ Claude viết ở mức L4.
- Xem bus: `candump can0` (Linux), hoặc SavvyCAN (có load DBC).

```bash
sudo ip link set can0 up type can bitrate 500000
candump -td can0
python tools/can_ctrl.py --interface socketcan --channel can0 --enable --freq 50 --amp 100
```
