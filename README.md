# NesCat1.0 ESP32 — skywalker0.5 Fork

> Forked from [nathalislight/NesCat1.0_ESP32](https://github.com/nathalislight/NesCat1.0_ESP32)

This fork adapts the original NesCat 1.0 project to run on an **ESP32-WROVER-E** with PSRAM enabled.  
It has been tested and confirmed working with the hardware and software environment described below.

---

## ✅ What Works

- NES Emulator — ROM selection, gameplay, and audio output
- MP3 Player — file browsing and audio playback
- SD card file browsing
- Hardware reset button

---

## 🛠 Hardware

| Component | Details |
|-----------|---------|
| MCU | ESP32-WROVER-E (with PSRAM) |
| Display | 2.4 inch IPS LCD 240x320 SPI, driver ST7789 |
| Audio Amplifier | MAX98357 I2S |
| Storage | MicroSD card module (SPI) |
| Buttons | Tactile micro switches with 10kΩ pull-down resistors to GND |

---

## 📌 Pin Assignment

### Buttons

| Function | GPIO |
|----------|------|
| UP       | 2    |
| DOWN     | 15   |
| LEFT     | 39   |
| RIGHT    | 34   |
| A        | 21   |
| B        | 26   |
| START    | 4    |
| SELECT   | 35   |

> ⚠️ All buttons use 10kΩ pull-down resistors to GND. Button pressed = HIGH (3.3V).  
> GPIO 34, 35, 39 are input-only pins — internal pull-down is not available; physical resistors are required.

### Display (ST7789, SPI)

| Signal | GPIO |
|--------|------|
| DC     | 5    |
| RST    | 19   |
| MOSI   | 23   |
| SCLK   | 18   |
| CS     | -1 (not used) |

### SD Card (SPI)

| Signal | GPIO |
|--------|------|
| MOSI   | 33   |
| MISO   | 13   |
| SCK    | 14   |
| CS     | 22   |

> ⚠️ SD MOSI was moved from GPIO 12 to GPIO 33.  
> GPIO 12 is a strapping pin on ESP32 — using it causes boot and upload failures.

### Audio (MAX98357, I2S)

| Signal | GPIO |
|--------|------|
| BCK    | 27   |
| LRC    | 32   |
| DIN    | 25   |
| SD     | 3.3V (always enabled) |
| VIN    | 5V   |

---

## 💾 SD Card Folder Structure

```
/
├── NES/        ← Place .nes ROM files here
└── AUDIO/      ← Place .mp3 files here
```

---

## 🖥 Software Environment

| Tool / Library | Version |
|----------------|---------|
| Arduino IDE | 2.3.3 |
| ESP32 by Espressif Systems | 2.0.17 |
| Adafruit GFX Library | 1.11.5 |
| SdFat by Bill Greiman | 1.0.5 |

### Board Settings (Arduino IDE)

| Setting | Value |
|---------|-------|
| Board | ESP32 Wrover Module |
| Flash Frequency | 80MHz |
| Flash Mode | QIO |
| Partition Scheme | Huge APP (3MB No OTA / 1MB SPIFFS) |
| Upload Speed | 921600 |

---

## 🔧 Key Modifications from Original

### 1. GPIO 12 → 33 (SD Card MOSI)
GPIO 12 is a strapping pin on ESP32. When a pin is inserted into GPIO 12 at boot, the ESP32 fails to start and cannot be programmed. SD Card MOSI was reassigned to GPIO 33.

### 2. ESP32 Core Version: 2.0.17
The original project recommends ESP32 core 1.0.5 or 1.0.6, but PSRAM support and stable I2S behavior were achieved using **2.0.17**. Version 3.x is not compatible due to API changes in WiFiClient and SPI flash functions.

### 3. I2S Driver Conflict Fix
On ESP32 core 2.0.x, the `Audio` object (used for MP3 playback) occupies the I2S peripheral before `init_sound()` is called for NES audio. This causes:
```
E I2S: register I2S object to platform failed
```
**Fix:** Call `i2s_driver_uninstall()` before re-initializing I2S for NES audio:
```cpp
// In NESemulator_part1.h — init_sound()
i2s_driver_uninstall(I2S_NUM_1); // Release I2S before re-init
i2s_driver_install(I2S_NUM_1, &audio_cfg, 0, NULL);
```
The same fix is applied before `audio.setPinout()` in the MP3 player section:
```cpp
i2s_driver_uninstall((i2s_port_t)I2S_NUM);
audio.setPinout(I2S_BCLK, I2S_LRC, I2S_DOUT);
```

### 4. KEYBOARD_ENABLED → false
`KEYBOARD_DATA` and `PIN_START` were both assigned to GPIO 4, causing a conflict. Since a keyboard is not used in this build, `KEYBOARD_ENABLED` was set to `false`.

### 5. NES Audio Volume Adjustment
The NES audio output level can be adjusted in `NESemulator_part1.h`:
```cpp
uint16_t a = (audio_frame[i] >> 0) * 7 / 10; // Adjust multiplier for volume
```
Using bitshift values greater than `>> 1` introduces quantization noise. Use multiplication/division for smooth volume control.

---

## ⚠️ Known Issues

- I2S initialization error messages appear at startup — these are non-critical and do not affect functionality.
- Audio volume cannot be adjusted in real time during NES gameplay.

---

## 📝 License

GPL-3.0 — see [LICENSE](LICENSE)