# 01 – Lộ trình

**Nhịp học giả định:** 4–5 giờ/tuần (2 buổi × 2 giờ). Lõi P0–P6 ≈ **20 tuần**. Bận tuần nào thì dời lịch, không bỏ bước.

Mỗi phase có:
- **DoD** (Definition of Done): xong hết mới tag.
- **Git**: bài tập Git gắn với phase (chi tiết trong `docs/03-git-workflow.md`).
- **AI mode**: mức để Claude can thiệp (chi tiết trong `docs/04-ai-workflow.md`). L1 = chỉ giải thích, L2 = bạn viết rồi Claude review, L3 = Claude viết khung, bạn viết lõi, L4 = giao Claude viết hết.

---

## P0 – Chuẩn bị (tuần 1–2) → `v0.1.0`

**Mục tiêu:** môi trường chạy được, phần cứng được xác nhận là sống.

- [ ] Cài: Git, GitHub CLI (`gh`), VS Code, CMake, Ninja, `arm-none-eabi-gcc` (Arm GNU Toolchain), `teensy_loader_cli`, Python 3 + `cantools` + `python-can`, Claude Code, Claude desktop app
- [ ] Tạo repo GitHub `teensy-inverter-emu`, push bộ file khởi tạo này (Git bài 1–2)
- [ ] Đọc schematic board DAC, điền bảng ⚠ trong `docs/00-hardware.md`
- [ ] Smoke test bằng **Arduino/Teensyduino** (để riêng trong `tools/smoke_test/`): bật ref, ghi 0 / mid / full-scale, đo điện áp
- [ ] Chụp frame SPI bằng logic analyzer, giải mã tay từng bit
- [ ] Viết `docs/adr/0001-toolchain.md` (đã có bản nháp) và chốt quyết định
- [ ] Dùng Cowork: tóm tắt chương LPSPI, eDMA, DMAMUX, PIT, FlexCAN của RM i.MX RT1060 thành `docs/notes/*.md` (chỉ ghi chú, không copy nguyên văn)

**DoD:** đo được 3 mức điện áp đúng; ảnh chụp frame SPI trong `docs/img/`; bảng phần cứng đã điền.
**Git:** bài 1 (init, config), 2 (commit đầu, push). **AI mode:** L4 cho smoke test, L1 cho đọc datasheet.

---

## P1 – Bare-metal nền móng (tuần 3–5) → `v0.2.0`

**Mục tiêu:** firmware của chính bạn, không Arduino, boot được trên Teensy.

- [ ] `cmake/arm-none-eabi.cmake`, CMakeLists, cờ Cortex-M7 (`-mcpu=cortex-m7 -mfpu=fpv5-d16 -mfloat-abi=hard`)
- [ ] Linker script: FlexSPI (0x60000000), ITCM, DTCM, OCRAM. Hiểu **FlexRAM bank config** (chia 512 KB giữa ITCM/DTCM/OCRAM)
- [ ] Boot header: **FlexSPI Configuration Block (FCB)** + **IVT** + boot data. Tham khảo `bootdata.c` và `imxrt1062.ld` trong repo `PaulStoffregen/cores` (teensy4) – đọc hiểu, ghi nguồn và license
- [ ] Startup: copy `.data`, zero `.bss`, bật FPU, cấu hình VTOR, bật I/D-cache, MPU tối thiểu
- [ ] Clock: ARM PLL → 600 MHz, root clock cho IPG/PERCLK/LPSPI/LPUART (module CCM)
- [ ] MCAL đầu tiên: `Port` (IOMUXC), `Dio` (GPIO, dùng GPIO6–9 fast GPIO), `Uart` (LPUART6 polling, `printf` tối giản)
- [ ] `DWT->CYCCNT` để đo thời gian theo chu kỳ CPU
- [ ] Gây HardFault có chủ ý (đọc địa chỉ không hợp lệ), viết fault handler in ra PC/LR/CFSR qua UART

**DoD:** blink + log "boot ok, SystemCoreClock=600000000" qua UART; fault handler in đúng địa chỉ lỗi.
**Học được (AUTOSAR):** Mcu, Port, Dio, startup/EcuM giai đoạn sớm. **Git:** bài 3 (branch + PR), 4 (`.gitignore` cho build). **AI mode:** L1–L2 (đây là phần quan trọng nhất để tự viết).

---

## P2 – Driver SPI + DAC8568, polling (tuần 6–7) → `v0.3.0`

- [ ] `Spi` (LPSPI4): clock, TCR (FRAMESZ = 31, CPOL/CPHA, PCS0, PRESCALE), CCR (SCKDIV, delay PCS↔SCK), FIFO, xử lý lỗi
- [ ] `Dac8568` (ECU abstraction): `Dac_Init` (reset, bật ref), `Dac_WriteInput(ch, code)`, `Dac_WriteAndUpdateAll(ch, code)`, `Dac_EncodeFrame()` (hàm thuần, test được trên PC)
- [ ] Tăng dần SCK: 1 → 10 → 25 MHz, ghi lại tốc độ cao nhất ổn định trên breadboard
- [ ] Ghi bảng đo: code → điện áp tại 5 điểm, tính sai số offset/gain

**DoD:** đặt 8 kênh 8 mức khác nhau từ UART command; ảnh logic analyzer frame ở 25 MHz; bảng sai số trong `docs/notes/dac-calibration.md`.
**Git:** bài 5 (conventional commits, sửa commit cuối bằng `--amend`). **AI mode:** L2.

---

## P3 – Tạo sóng 3 pha: DDS + LUT, chạy bằng ngắt (tuần 8–9) → `v0.4.0`

- [ ] `Gpt` (PIT): ngắt chu kỳ `Ts = 1/fs` (bắt đầu với fs = 50 kHz)
- [ ] `SigGen`: **DDS** (phase accumulator 32 bit), LUT sin 1024 điểm `int16_t` (sinh bằng script Python lúc build hoặc `const` table), lệch pha ±2³²/3, scale biên độ Q15, offset mid-scale, clamp
- [ ] Mô hình cảm biến dòng: `V = V_offset + I × sensitivity` (tham số: offset 2.5 V, 10 mV/A… cấu hình được)
- [ ] Trong ISR: tính 3 mẫu → gửi 3 frame SPI (A, B ghi input; C ghi và **update all**)
- [ ] Toggle GPIO đầu/cuối ISR → đo **thời gian chạy ISR, jitter, % CPU**
- [ ] **Unit test trên PC** (Unity hoặc CppUTest, build bằng CMake host, chạy `ctest`): DDS tần số đúng, 3 pha lệch 120°, Ia+Ib+Ic ≈ 0, clamp, encode frame

**DoD:** sóng 3 pha 50 Hz và 1 kHz trên scope/ADC; bảng CPU load ở fs 20/50/100 kHz; `ctest` xanh.
**Git:** bài 6 (`git bisect` – xem bài tập), 7 (merge conflict có chủ ý). **AI mode:** L2 cho DDS, L4 cho script sinh LUT và khung test.

---

## P4 – eDMA: tạo sóng không tốn CPU (tuần 10–12) → `v0.5.0`

**Đây là phần khó và đáng giá nhất về MCU.**

- [ ] `Dma`: eDMA TCD (SADDR, SOFF, NBYTES, SLAST/DLAST_SGA, CITER/BITER, INTHALF, INTMAJOR), DMAMUX
- [ ] **PIT kích DMA** qua periodic trigger của DMAMUX (chỉ có ở một số kênh DMA đầu – ⚠ VERIFY chương DMAMUX trong RM). Mỗi trigger = 1 minor loop = 3 word (12 byte) → `LPSPI4->TDR`
- [ ] Buffer vòng ping-pong `uint32_t txbuf[2][N*3]` (N = 64 mẫu/nửa): ngắt half/full → CPU điền nửa vừa phát xong
- [ ] **D-cache:** đặt buffer ở DTCM (không cache) *hoặc* OCRAM + `SCB_CleanDCache_by_Addr`. Thử cả hai, giải thích được vì sao sai thì ra sóng rác
- [ ] Phát hiện lỗi: underrun (CPU điền không kịp), DMA error IRQ, LPSPI FIFO error
- [ ] So sánh với P3: jitter (gần 0 vì phần cứng định thời), CPU load

**DoD:** 3 pha ở fs = 100 kHz, CPU load < 5 %; đo jitter bằng logic analyzer trên SYNC; ghi `docs/notes/dma-design.md` có sơ đồ TCD.
**Git:** bài 8 (interactive rebase dọn commit trước PR). **AI mode:** L1–L2.

---

## P5 – Port sang FreeRTOS (tuần 13–14) → `v0.6.0`

- [ ] FreeRTOS kernel là **git submodule** ghim theo tag (Git bài 9), port `ARM_CM7`
- [ ] Ưu tiên ngắt: hiểu `configMAX_SYSCALL_INTERRUPT_PRIORITY`, NVIC priority grouping, ISR nào được gọi API `FromISR`
- [ ] Task: `SigGen` (cao nhất, chờ task notification từ ISR DMA), `Ctrl` 10 ms, `Status` 100 ms, idle hook (đo CPU load, đá watchdog)
- [ ] Dữ liệu giữa task: setpoint double-buffer hoặc queue, không dùng biến global trần
- [ ] Đo: stack high-water mark mỗi task, CPU load, độ trễ ISR → task
- [ ] Viết bảng so sánh FreeRTOS ↔ AUTOSAR OS (task/ISR cat 2/event/resource/alarm) trong `docs/notes/rtos-vs-autosar-os.md`

**DoD:** hành vi giống P4, thêm lệnh UART `stats` in stack + CPU load mỗi task.
**Git:** bài 9 (submodule). **AI mode:** L3 (khung port), L2 cho phần ưu tiên ngắt.

---

## P6 – CAN + điều khiển từ CANoe (tuần 15–20) → `v1.0.0`

- [ ] Mua transceiver + 120 Ω; đấu bus theo `docs/05-can-interface.md`
- [ ] `Can` (FlexCAN1): bit timing 500 kbit/s (tự tính PRESDIV, PROPSEG, PSEG1, PSEG2, RJW từ clock), message buffer RX có filter, TX, bus-off + recovery
- [ ] `CanIf` mỏng: map ID → PDU, callback RX/TX confirmation
- [ ] `Com`: pack/unpack signal **sinh tự động từ DBC** bằng script Python + `cantools` (`tools/gen_com.py` → `gen/Com_Cfg.c/.h`). Đây là bản thu nhỏ của bộ generator DaVinci
- [ ] `Ctrl`: state machine `INIT → STANDBY → RUN → RAMP_DOWN → FAULT`, giới hạn tốc độ thay đổi (ramp) biên độ/tần số, **đổi tần số liên tục pha** (ưu điểm của DDS)
- [ ] **Timeout** mất lệnh 200 ms → ramp về 0 A (safe state), báo trong status
- [ ] Phía PC: CANoe config + panel (Freq, Amp, Enable) + CAPL gửi lệnh có alive counter; **hoặc** `tools/can_ctrl.py` (python-can) nếu không có Vector HW
- [ ] Test tự động phía PC (`tools/tests/test_can.py`, pytest): enable → đọc status RUN; rút lệnh → FAULT/timeout trong 200 ± 20 ms; ramp đúng tốc độ

**DoD:** đổi biên độ/tần số từ CANoe/python thấy ngay trên scope; test pytest chạy xanh một lệnh; README có GIF/video demo.
**Git:** bài 10 (release + artifact `.hex`), 11 (GitHub Actions). **AI mode:** L2 cho FlexCAN, L4 cho generator và tool Python, L3 cho CAPL.

---

## P7+ – Mở rộng (chọn theo hướng nghề nghiệp)

Xếp theo **tính ứng dụng** (★ = ưu tiên). Mỗi mục là một nhánh `feature/` và một tag minor.

| # | Mở rộng | Ứng dụng thật | Liên hệ profile | Công |
|---|---|---|---|---|
| ★E1 | **Fault injection** qua CAN: offset, gain error, mất pha, stuck-at, nhiễu, sóng hài bậc 5/7, mất cân bằng 3 pha | Test plausibility/diagnostic của ECU inverter (ví dụ Ia+Ib+Ic ≠ 0, sensor out-of-range) đúng như HIL | Đúng công việc test inverter + FuSa verification | 2–3 tuần |
| ★E2 | **Lệnh trong hệ dq**: nhận Id/Iq + tốc độ → inverse Park/Clarke → abc; xuất **góc điện** ra kênh D/E dạng **sin/cos resolver** (hoặc Hall/encoder ABZ qua GPIO/QTimer) | Plant giả cho bộ điều khiển FOC; đồng bộ dòng với vị trí rotor | Motor control | 3 tuần |
| ★E3 | **E2E protection** (CRC8 + alive counter kiểu AUTOSAR E2E Profile 1) cho lệnh điều khiển + error memory kiểu DEM (debounce, status bit, lưu flash) | Bản tin điều khiển an toàn chức năng | ISO 26262, DEM | 2 tuần |
| ★E4 | **UDS trên CAN**: ISO-TP + 0x10, 0x22 (đọc trạng thái, version), 0x2E (ghi calibration), 0x31 (routine bắt đầu fault injection), 0x19/0x14 (DTC) | Cấu hình/chẩn đoán thiết bị test | DCM/DEM đang ôn | 4 tuần |
| ★E5 | **XCP on CAN slave**: đo/hiệu chỉnh amplitude, freq, offset, sensitivity bằng CANape/CANoe hoặc `pyxcp` | Calibration như ECU thật | Từng own XCP stack | 3 tuần |
| ★E6 | **CI/CT**: GitHub Actions build firmware + host test + `cppcheck` (MISRA addon) + clang-format check; tùy chọn self-hosted runner nối board để chạy pytest HIL | Pipeline giống dự án thật | CI/CT 2 ngày → 3 giờ | 1–2 tuần |
| E7 | **Tự giám sát**: ADC đọc lại 3 output qua phân áp → so sánh với giá trị đặt, phát hiện lỗi DAC/dây → safe state | Safety mechanism kiểu "read-back" | FuSa | 2 tuần |
| E8 | **Phát lại sóng thực**: nạp file dòng đo từ xe/HIL (CSV) qua USB/CAN rồi phát lại | Tái hiện lỗi hiện trường | HIL | 2 tuần |
| E9 | **Mô hình PMSM thời gian thực** (closed-loop): bắt PWM từ bộ điều khiển thật bằng FlexPWM/QTimer capture → tính dòng → xuất DAC | Mini-HIL đúng nghĩa | HIL, motor control | 6+ tuần |
| E10 | CAN-FD trên CAN3, **bootloader UDS** (0x34/0x36/0x37) | Flash/FOTA | Secure flash, FOTA | 4 tuần |
| E11 | **Port sang Rust** (`teensy4-rs` + RTIC) giữ nguyên DBC và test pytest | So sánh C ↔ Rust trên cùng bài toán | Lộ trình Rust | 4–6 tuần |

**Gợi ý thứ tự sau P6:** E6 → E1 → E3 → E4 (hoặc E5) → E2. Sau chuỗi này bạn có một thiết bị "current sensor emulator có E2E, DTC, UDS/XCP, CI" – chuyện kể phỏng vấn FAE/BSW rất cụ thể.

---

## Câu nói phỏng vấn sau từng mốc

| Mốc | Câu kể được |
|---|---|
| v0.5 | "I wrote bare-metal drivers on i.MX RT1062 and generated 3-phase signals with PIT-triggered eDMA into LPSPI, zero CPU load, and handled Cortex-M7 cache coherency." |
| v1.0 | "It's a CAN-controlled current-sensor emulator on FreeRTOS, with a Com layer generated from DBC and timeout-to-safe-state, tested by a pytest suite." |
| v1.x | "I use it to inject sensor faults and verify inverter diagnostics, with E2E-protected commands, UDS/XCP access and CI." |
