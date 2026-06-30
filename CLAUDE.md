# Project: evDash-derived CAN/GPS Logging Board

## Overview

A simplified, headless CAN/GPS logging device derived from the [evDash](https://github.com/nickn17/evDash) open-source EV dashboard project, developed in collaboration with the evDash maintainer. Unlike the original M5Stack CoreS3-based display device, this board has **no LCD, no touch, no menu UI** — it's a logging-only device: capture CAN data, capture GPS position, buffer to SD card, and upload logs to the cloud over WiFi.

---

## Hardware

### MCU

- **ESP32-S3, N16R2** — 16MB flash, 2MB Quad-SPI PSRAM, dual-core Xtensa LX7 @ 240MHz, full **85°C** operating temperature rating (important: this lives in a car cabin, where Octal-PSRAM variants' 65°C rating would be a real risk).
- **Prototype build**: ESP32-S3-WROOM-1-N16R2 module (larger footprint, easier hand-assembly/probing).
- **Production build**: ESP32-S3-MINI-1-N16R2 module (smaller footprint).
- Both modules expose **identical GPIO numbers** for every signal this design uses — confirmed against both official Espressif datasheets. **Firmware requires zero changes between prototype and production hardware.**
- A third-party module, **DOIT ESPS3-32**, was also evaluated and is pin-compatible with this design (same GPIO numbers), though it's a third-party (non-Espressif) module — worth independent certification/sourcing diligence if used in production.
- PSRAM is **Quad SPI**, not Octal — this matters because Octal-PSRAM variants (R8/R16V) permanently consume IO35/36/37 for the PSRAM bus and carry a reduced 65°C temperature rating. N16R2 avoids both issues.
- Rationale for PSRAM at all (despite dropping the display, which was the original justification): the planned web-config UI + HTTPS + MQTT-over-TLS networking stack can plausibly need several hundred KB of RAM when run concurrently; 2MB PSRAM gives comfortable headroom without the Octal-PSRAM downsides.
- 16MB flash sizing rationale: OTA updates need ~2× firmware image size (dual app partitions for safe rollback) plus filesystem space for web-config assets — 16MB gives breathing room 8MB would not.

### Pin Map (v5 — current/locked)

| Function | Pin(s) | Notes |
|---|---|---|
| USB-UART bridge (flash + console) | TXD0 (GPIO43), RXD0 (GPIO44) | External bridge chip (not native USB — see rationale below) |
| Auto-reset circuit | DTR/RTS → EN, IO0 | Standard transistor-pair auto-reset, same as any ESP32 devkit reference design |
| GPS UART1 | TX: IO17, RX: IO18 | NMEA0183 @ 115200bps, ATGM336H-7N module |
| GPS ON/OFF | IO6 | Active-low shutdown control (power off GPS while parked) |
| GPS nRESET | IO14 | Active-low module reset (recover hung GPS without full power cycle) |
| TWAI (CAN) | TX: IO4, RX: IO5 | Native ESP32-S3 CAN peripheral — no MCP2515, no SPI for CAN |
| SD card SPI2 | SCLK: IO12, MOSI: IO11, MISO: IO13, CS: IO10 | Dedicated bus, not shared with anything (no LCD, no CAN controller) |
| SD card power control | IO7 | Active-high load switch on SD 3.3V rail — gate OFF only after buffer flush + `SD.end()` |
| I2C bus | SDA: IO8, SCL: IO9 | Shared bus: BMI270 (accel/gyro) + small OLED, distinguished by I2C address |
| BMI270 INT1 | IO19 | Interrupt-driven motion wake (avoids continuous I2C polling while parked) |
| Status LED 1 | IO15 | General-purpose, meaning defined in firmware |
| Status LED 2 | IO16 | General-purpose, meaning defined in firmware |
| Strapping — leave alone | IO0 (shared w/ auto-reset), IO3 | IO0 needs defined boot-mode behavior; IO3 needs a defined level, not floating |
| **Free / spare** | IO1, IO2, IO20, IO21, IO35–37, IO38–42 (JTAG), IO46, IO47, IO48 | Available for expansion (AUX-voltage ADC, wake button, expansion header, etc.) |

### Key hardware decisions and rationale

- **CAN: native TWAI, not MCP2515/SPI.** The original evDash hardware uses an MCP2515 SPI CAN controller. Since this board's MCU (ESP32-S3) has a built-in TWAI (CAN 2.0) peripheral, the MCP2515 is eliminated entirely — frees the SPI bus for the SD card alone, removes a BOM component, and only needs a CAN transceiver chip (e.g. TJA1050/SN65HVD230) wired to two GPIOs (TWAI is GPIO-matrix-routed, no fixed pins).
- **USB: external UART bridge, not native USB-Serial/JTAG.** ESP32-S3 has a native USB-Serial/JTAG controller (would have used IO19/IO20) capable of auto-reset-into-bootloader without external circuitry. This was evaluated and **rejected** because the native USB CDC console **disconnects during deep sleep** (and on boot loops before the driver starts) — given this device's planned sleep/wake power-management behavior while parked, a console that vanishes on every sleep cycle is a real field-debugging annoyance. An external USB-UART bridge on GPIO43/44 (the IO-MUX-default UART0 pins) avoids this — it's a passive electrical path that doesn't disconnect on sleep, at the cost of needing a standard DTR/RTS auto-reset circuit around EN/IO0.
- **SD card: kept as standard SD-over-SPI** (Arduino `SD.h`, FAT32, 40MHz), same as the original evDash design — not SDIO/SD_MMC 4-bit mode, not raw SPI NAND/NOR flash. Industrial-grade SDHC cards (SanDisk Industrial, Swissbit, Transcend Industrial) recommended for endurance given continuous logging workload. SPI NAND and FRAM/EEPROM were evaluated as alternatives and rejected — SD keeps the existing `SD.h` firmware pipeline intact and preserves the "human can pull the card and read JSON logs" workflow.
- **No external RTC chip.** The ESP32-S3's own internal RTC domain (powered through deep sleep, does not survive full power loss) is synced from NTP (primary, when WiFi available) or GPS NMEA time sentences (`$GPRMC`/`$GPZDA`, fallback) — this mirrors evDash's existing `BoardInterface::setTime()` pattern exactly (`settimeofday()`). The GPS module's own internal RTC (VBAT-backed) was considered but has no host-accessible register interface — its only practical use is faster GPS hot-start via VBAT backup, not a queryable clock for the ESP32.
- **GPS VBAT backup**: decision deferred between a coin cell (long life, replaceable, years of backup) or a supercapacitor (rechargeable from 3.3V, no replacement, but only minutes–hours of backup after power loss). Electrically trivial either way (GPS backup draw is ~15µA typical).
- **INA3221 voltmeter**: kept as-is from evDash. Confirmed board-agnostic in the original source — not gated by any `BOARD_M5STACK_*` `#ifdef`, runs identically regardless of board. Monitors the car's 12V AUX battery (not anything internal to the MCU board) for two purposes: low-voltage shutdown protection, and detecting AUX voltage crossing ~14V as a signal the car's DC-DC converter is active (car woke up → resume CAN polling).
- **GPIO numbering, not physical pin numbers, is the portable abstraction** — confirmed across WROOM-1, MINI-1, and ESPS3-32 datasheets. This is why prototype (WROOM-1) and production (MINI-1) hardware can share firmware unchanged.

---

## Firmware Architecture (relative to upstream evDash)

### What's being replaced/removed

- **`Board320_240`** (display/sprite/menu engine) — bypassed entirely. It's tightly coupled to the M5Unified/M5CoreS3/M5Core2 libraries at the class-member level (`M5GFX &tft`, `M5Canvas spr`, etc.), gated by `#ifdef BOARD_M5STACK_CORE2` / `BOARD_M5STACK_CORES3`. A new board class implements `BoardInterface` directly instead of subclassing `Board320_240`.
- **M5 library dependency** (`M5Unified`, `M5Core2.h`, `M5CoreS3.h`) — eliminated along with `Board320_240`, since that's the only place it's referenced.
- **AW9523 I2C GPIO extender** — present on original M5Stack CoreS3 hardware (per schematics) for LCD/touch signal routing, but **never referenced directly in evDash source** — all touch/display I/O goes through the M5 library's high-level API. Since this design drops the LCD/touch entirely, the AW9523 and its routing are simply not present on this board; no firmware-level dependency to account for.
- **`CommObd2Can`** (MCP2515-based CAN driver) — being replaced by a new `CommObd2Twai` implementation. The `CommInterface` abstract base class is otherwise unchanged.

### What's being kept/ported largely as-is

- **`CommInterface`** abstraction — `connectDevice()`, `disconnectDevice()`, `mainLoop()`, `executeCommand()`, `sendPID()`, `receivePID()`, plus the ISO-TP multi-frame reassembly logic (`processFrameBytes()`, `processFrame()`, `processMergedResponse()`, `getFrameType()`) — this layer operates on bytes already pulled from the CAN driver and has **no MCP2515-specific code inside it**, so it ports to TWAI essentially unchanged. Only the byte-acquisition layer (`connectDevice`, `sendPID`, `receivePID`, suspend/resume) needs rewriting against ESP-IDF `twai_*` calls instead of the `mcp_can` library.
- **`CarInterface`** and all car-specific parsers (`CarKiaEniro.cpp`, `CarHyundaiEgmp.cpp`, `CarVWID3.cpp`, etc.) — zero display coupling, zero CAN-driver coupling; untouched.
- **`LiveData`** (params/settings shared state) — untouched.
- **SD card mount/record/upload pipeline** (`sdcardMount()`, `sdcardToggleRecording()`, the JSON Lines telemetry log writer, the 256KB-rollover "contribute v2" format) — untouched, board-agnostic already.
- **INA3221 voltmeter logic** — untouched, board-agnostic already (see hardware section above).
- **OTA update logic** (`Board320_240::otaUpdate()`, HTTPS download via `Update.h`, TLS-pinned to `ISRG_ROOT_X1`) — the only board-coupling is calls to `displayMessage()` for progress feedback, which is a required `BoardInterface` method regardless (can be a no-op or routed to syslog/LED/web-status on a headless board). Otherwise portable with minimal change.

### What needs new code

- **A new `BoardInterface` implementation** for this hardware — handles `initBoard()`, `boardLoop()`, sleep/wake, and stub/no-op versions of the display-related pure-virtuals (`displayMessage()`, `confirmMessage()`, `showMenu()`, etc., since `BoardInterface` requires them even on a headless device).
- **`CommObd2Twai`** — new `CommInterface` implementation using ESP-IDF `twai_driver_install()`/`twai_transmit()`/`twai_receive()` in place of `MCP_CAN`/SPI calls. Per-car filter/mask logic (BMW i3, Peugeot e208 special cases in the original) maps to `twai_filter_config_t`.
- **Web-config interface** — not yet started; will need its own memory/flash budget consideration (see PSRAM/flash sizing rationale above).
- **MQTT/HTTPS cloud upload client** — extends the existing OTA-adjacent HTTPS/TLS patterns already in the codebase; not yet started.
- **GPS NMEA parsing → time sync** — wire ATGM336H-7N's NMEA stream into the existing `setTime()`/NTP-fallback pattern.
- **BMI270 motion-wake logic** — new, but conceptually mirrors the **existing** evDash gyro-based Sentry wake logic on CoreS3/Core2 (`CoreS3.Imu.getGyroData()`/`getAccelData()`, threshold-based `liveData->params.gyroSensorMotion` flag) — same intent (wake from parked/sleep on motion), different chip/driver (BMI270 over I2C with a hardware interrupt on INT1, vs. the M5 IMU library's polled gyro read).

### evDash telemetry log format (for reference — what the original device logs)

Two competing JSON Lines formats exist, selected by `liveData->settings.contributeJsonType`:
- **Format A (legacy, `serializeParamsToJson`)**: flat per-tick snapshot of nearly all `LiveData::params` fields — SoC/SoH, battery V/A/kWh, cell voltages, thermal (inverter/motor/battery/cabin), tire pressure/temp ×4, charging cumulative energy, GPS lat/lon/speed/heading, etc.
- **Format B ("contribute v2", `buildContributePayloadV2`)**: structured, upload-oriented — header (device/car identity), rolling `motion` array (GPS + min/max cell voltage samples over time), `charging` array, `chargingStart`/`chargingEnd` event objects, optional raw CAN frame snapshots. Includes 256KB-cap file rollover for manageable upload chunks.

Format B's structure (event timeline + periodic GPS/battery motion samples) is likely the better starting point for this logging-focused device, though Format A's breadth (every parsed CAN field, every tick) may matter if full fidelity is wanted.

---

## Working Principles (how this project is being run)

- **GPIO-number-based pin assignments**, verified against multiple module datasheets — never physical pin numbers — is the standard for all hardware decisions, since it's what makes prototype/production module portability possible.
- **No speculation about hardware behavior from code comments or indirect inference.** Only verified source files (project knowledge) or datasheets provided directly are treated as evidence. When something can't be verified this way, it's flagged explicitly as unconfirmed rather than asserted.
- **Design decisions are reached iteratively**, with explicit confirmation at each stage before moving to the next.
- Illia is working in collaboration with the evDash developer/maintainer directly.

---

## Open Items / Not Yet Decided

- GPS VBAT backup: coin cell vs. supercapacitor (electrically trivial either way; deferred to a later decision point).
- Web-config UI scope and asset size (affects flash/filesystem partition sizing).
- MQTT broker / cloud upload backend details (self-hosted vs. piggybacking on evDash's existing `api.evdash.eu` infrastructure).
- Final partition table (ota_0/ota_1/spiffs/nvs sizing) — deferred until firmware size is better known.
- CAN transceiver chip selection (TJA1050, SN65HVD230, or similar — needs to sit between the TWAI GPIO pins and the physical CAN bus).
- Power supply design: 12V automotive input → 3.3V regulation, reverse-polarity/transient protection.
- BMI270 INT1 polarity/configuration (active-high/low, push-pull/open-drain) and I2C address vs. OLED address collision check — not yet verified against the BMI270 datasheet directly.
