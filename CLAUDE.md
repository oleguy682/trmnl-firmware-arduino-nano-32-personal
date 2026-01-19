# Arduino Nano ESP32 Configuration

This document describes the configuration and setup for running TRMNL firmware on the Arduino Nano ESP32 board.

## Hardware Specifications

**Board:** Arduino Nano ESP32
- **Module:** NORA-W106-10B (ESP32-S3)
- **Flash:** 16MB
- **PSRAM:** 8MB (ESP32-S3R8)
- **CPU:** Dual-core 240MHz
- **Connectivity:** WiFi, Bluetooth LE

## Pin Configuration

### E-Paper Display (SPI)
```cpp
EPD_SCK_PIN   48   // SPI Clock
EPD_MOSI_PIN  38   // SPI MOSI
EPD_CS_PIN    21   // Chip Select
EPD_RST_PIN   17   // Reset
EPD_DC_PIN    18   // Data/Command
EPD_BUSY_PIN  10   // Busy
```

### External Button
```cpp
PIN_INTERRUPT  9   // D6/GPIO9 - External momentary button
                   // Requires 10kΩ pull-up resistor to 3.3V
```

**Button Wiring:**
```
                     +3.3V
                       |
                      [R]  10kΩ Pull-up Resistor
                       |
                       +-------- D6 / GPIO9
                       |
                      ///  Momentary Switch
                       |
                      GND
```

## Build Configuration

### PlatformIO Environment

```ini
[env:arduino_nano_esp32]
extends = env:esp32_base
board = esp32s3_n16r8
board_build.f_cpu = 240000000L
board_build.f_flash = 80000000L
board_build.flash_mode = qio
board_build.partitions = trmnl_x_partitions.csv
board_build.filesystem = spiffs
build_flags =
    ${env:esp32_base.build_flags}
    -D ARDUINO_USB_MODE=1
    -D ARDUINO_USB_CDC_ON_BOOT=1
    -D WAIT_FOR_SERIAL=1
    -D ARDUINOJSON_ENABLE_ARDUINO_STRING=1
    -D BOARD_ARDUINO_NANO_ESP32
    -D PNG_MAX_BUFFERED_PIXELS=6432
    -D FORCE_FULL_REFRESH=1
lib_deps =
    ${deps_app.lib_deps}
    bitbank2/FastEPD@1.3.0
monitor_speed = 115200
```

### Build Flags Explained

- `BOARD_ARDUINO_NANO_ESP32` - Enables Arduino Nano ESP32 specific code paths
- `FORCE_FULL_REFRESH=1` - Forces full e-ink refresh every update (prevents partial refresh artifacts)
- `PNG_MAX_BUFFERED_PIXELS=6432` - Limits PNG decode buffer size for memory constraints
- `FAKE_BATTERY_VOLTAGE` - Simulates battery voltage (no battery monitoring hardware)

### Library Dependencies

**FastEPD (bitbank2/FastEPD@1.3.0)**
- Used instead of bb_epaper to avoid PSRAM allocation issues
- Better compatibility with ESP32-S3 and Arduino framework
- Supports both 1-bit and 2-bit grayscale rendering

## Firmware Modifications

### 1. API Setup Parser (`lib/trmnl/src/parse_response_api_setup.cpp`)

**Issue:** Terminus BYOS API doesn't include `"status": 200` field in responses (only TRMNL Cloud does)

**Fix:** Made status field optional with default value of 200
```cpp
response.status = doc["status"] | 200;  // Default to 200 if not present

// Only fail if status explicitly set and not 200
if (doc.containsKey("status") && response.status != 200) {
    response.outcome = ApiSetupOutcome::StatusError;
    return response;
}
```

### 2. Filename Persistence (`src/bl.cpp`)

**Issue:** ESP32-S3 `esp_sleep_get_wakeup_cause()` always returns 0 (UNDEFINED) instead of proper timer wakeup value, causing display to continuously re-flash the same image.

**Fix:** Check if filename exists in preferences instead of relying on wakeup reason
```cpp
String savedFilename = preferences.getString(PREFERENCES_FILENAME_KEY, "");
bool isGpioWakeup = (wakeup_reason == ESP_SLEEP_WAKEUP_GPIO ||
                     wakeup_reason == ESP_SLEEP_WAKEUP_EXT0 ||
                     wakeup_reason == ESP_SLEEP_WAKEUP_EXT1);

if (savedFilename.length() == 0 || isGpioWakeup) {
    // Show logo and clear filename only on first boot or button press
    display_show_image(storedLogoOrDefault(1), DEFAULT_IMAGE_SIZE, false);
    preferences.putString(PREFERENCES_FILENAME_KEY, "");
}
```

### 3. GPIO Wakeup Configuration (`src/bl.cpp`)

**Issue:** Floating GPIO0 (boot button) caused spurious wakeups from deep sleep.

**Solution:** Configure external button on GPIO9 with proper pull-up resistor
```cpp
#elif CONFIG_IDF_TARGET_ESP32S3
  esp_sleep_enable_ext0_wakeup((gpio_num_t)PIN_INTERRUPT, 0);
#endif
```

### 4. Full Refresh Mode (`src/display.cpp`)

**Issue:** Partial refresh causes rendering artifacts with emojis and complex layouts on e-ink display.

**Solution:** Added build flag to force full refresh every update
```cpp
if ((iUpdateCount & 7) == 0 || apiDisplayResult.response.maximum_compatibility == true
#ifdef FORCE_FULL_REFRESH
    || true  // Build flag set to always force full refresh
#endif
) {
    iRefreshMode = REFRESH_FULL;
}
```

## Known Issues and Workarounds

### 1. Wakeup Reason Detection
**Problem:** `esp_sleep_get_wakeup_cause()` always returns 0 on ESP32-S3
**Workaround:** Use filename persistence check instead of wakeup reason
**Status:** Working correctly

### 2. Partial Refresh Artifacts
**Problem:** Emojis and complex graphics render poorly with partial refresh
**Workaround:** `FORCE_FULL_REFRESH=1` build flag
**Status:** Resolved - full refresh works perfectly

### 3. PSRAM Heap Issues
**Problem:** bb_epaper library causes heap corruption with Arduino-only framework
**Workaround:** Use FastEPD library instead
**Status:** Resolved

### 4. Git Push Restrictions
**Problem:** Cannot push directly to `arduino_nano_esp32_configuration` branch
**Workaround:** Commits backed up on `claude/*` branches
**Status:** Expected behavior

## Usage Instructions

### Building and Flashing

```bash
# Build firmware
pio run -e arduino_nano_esp32

# Upload to device
pio run -e arduino_nano_esp32 -t upload

# Monitor serial output
pio device monitor -e arduino_nano_esp32
```

### Initial Setup

1. **Connect Hardware:**
   - Connect e-paper display to SPI pins
   - Wire external button to D6/GPIO9 with 10kΩ pull-up

2. **Power On:**
   - Device displays TRMNL logo and setup QR code
   - Connect to WiFi network shown on screen

3. **Configure Device:**
   - Scan QR code or navigate to setup URL
   - Enter API key from Terminus BYOS server
   - Device will register and begin normal operation

### Normal Operation

- **Timer Wake:** Device wakes every 5 minutes (default), fetches new content, displays, then sleeps
- **Button Wake:** Pressing button during sleep wakes device and forces refresh
- **Full Refresh:** Every screen update uses full refresh (prevents ghosting)

## Terminus BYOS Integration

### API Endpoints Used

1. **Setup API** (`/api/setup/`)
   - Used during initial device provisioning
   - Returns API key and friendly ID

2. **Display API** (`/api/display/`)
   - Fetches screen content on each wake cycle
   - Returns BMP image URL and refresh rate

### Response Format Differences

Terminus BYOS differs from TRMNL Cloud:
- Does not include `"status": 200` field in responses
- Firmware defaults to 200 if field absent
- All other fields compatible with TRMNL Cloud API

## Development Notes

### Framework Choice
- **Framework:** Arduino-only (not arduino+espidf hybrid)
- **Reason:** Hybrid framework causes heap allocation mismatches with e-paper libraries
- **Tradeoff:** Fewer ESP-IDF features, but stable operation

### Memory Considerations
- Total heap: ~8MB PSRAM + ~400KB internal RAM
- PNG decode buffer limited to 6432 pixels to conserve memory
- FastEPD manages buffers efficiently

### Power Consumption
- Deep sleep: ~10-20µA (theoretical)
- Active (WiFi): ~100-200mA
- Display refresh: ~50-100mA peak

## Testing Checklist

- [x] Device boots successfully
- [x] WiFi connection stable
- [x] Display renders correctly
- [x] API setup completes
- [x] Filename persists across reboots
- [x] Deep sleep/wake cycle works
- [x] Button wakeup functions
- [x] Full refresh renders cleanly
- [x] No memory leaks or crashes
- [x] Terminus BYOS integration successful

## References

- [Arduino Nano ESP32 Documentation](https://docs.arduino.cc/hardware/nano-esp32)
- [TRMNL Firmware Repository](https://github.com/usetrmnl/trmnl-firmware)
- [Terminus BYOS](https://github.com/carlallen/terminus)
- [FastEPD Library](https://github.com/bitbank2/FastEPD)

## Contributors

Configuration and development by Claude Code AI assistant in collaboration with user oleguy682.

## License

Follows TRMNL firmware licensing (see LICENSE file in repository root).
