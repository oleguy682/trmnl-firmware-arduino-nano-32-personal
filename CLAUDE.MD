# CLAUDE.MD - TRMNL Firmware Project Guide

## Project Overview

This is the firmware for **TRMNL** - an ESP32-based e-ink display device that connects to WiFi, fetches images from a web server, and displays them on an e-paper screen. The device is optimized for ultra-low power consumption using deep sleep modes to achieve ~120-210 days of battery life on a single charge.

**Key Technologies:**
- Platform: ESP32-C3 (primary), ESP32-S3, and other ESP32 variants
- Framework: Arduino + ESP-IDF
- Build System: PlatformIO
- Language: C++
- Key Libraries: ArduinoJson, PNGdec, JPEGDEC, ESPAsyncWebServer

## Architecture & Key Concepts

### Power Management
- Device uses **deep sleep** between display updates to conserve battery
- GPIO remains active during sleep to detect button presses (~100uA consumption)
- WiFi is the biggest power consumer during active cycles
- Buck converter (not linear regulator) ensures efficient power delivery
- Target: 96 updates/day at 15-min intervals = ~120-140 days battery life

### Device Lifecycle
1. **Wake up** (by timer or button press)
2. **Connect to WiFi** (or start captive portal if not configured)
3. **Check for firmware updates** (OTA capable)
4. **Fetch new image** from TRMNL API
5. **Update e-ink display**
6. **Enter deep sleep** until next scheduled wake

### API Communication
The device communicates with TRMNL's backend via three endpoints:

**GET /api/setup** - Initial device registration (Mac address → API key + Friendly ID)
**GET /api/display** - Fetch image URL and device commands (update firmware, reset, etc.)
**POST /api/log** - Submit device logs/errors

See README.md for detailed API specifications.

## Project Structure

```
trmnl-firmware-arduino-nano-32-personal/
├── src/                          # Main source code
│   ├── main.cpp                  # Entry point
│   ├── bl.cpp                    # Business logic (main state machine)
│   ├── display.cpp               # E-ink display control
│   ├── button.cpp                # Button input handling
│   ├── filesystem.cpp            # SPIFFS file operations
│   ├── preferences_persistence.cpp # NVS storage (WiFi creds, API keys)
│   ├── qa.cpp                    # Quality assurance/testing mode
│   ├── api-client/               # API communication modules
│   │   ├── setup.cpp
│   │   ├── display.cpp
│   │   └── submit_log.cpp
│   └── *.h                       # Headers (logo graphics, QR codes, etc.)
│
├── include/                      # Public headers
│   ├── config.h                  # Build configuration
│   ├── display.h                 # Display interface
│   ├── types.h                   # Common type definitions
│   └── api-client/               # API client headers
│
├── lib/                          # Custom libraries
│   ├── bb_epaper/                # E-paper driver library
│   ├── trmnl/                    # TRMNL-specific utilities
│   └── wificaptive/              # Captive portal for WiFi setup
│
├── test/                         # Unit tests (Unity framework)
│   ├── test_string_utils/
│   ├── test_bmp/
│   ├── test_png_flip/
│   ├── test_parse_api_setup/
│   └── test_parse_api_display/
│
├── scripts/                      # Build scripts
│   └── git_version.py            # Embeds git hash in firmware version
│
├── platformio.ini                # PlatformIO build configuration
└── README.md                     # User-facing documentation
```

## Build Environments

The project supports multiple hardware configurations via PlatformIO environments:

### Primary Targets
- **trmnl** (default) - TRMNL OG hardware (ESP32-C3)
- **local** - Development build with USB serial logging
- **TRMNL_X** - Next-gen hardware (ESP32-S3, larger display)

### Development Boards
- **seeed_xiao_esp32c3** - Seeed Studio XIAO ESP32C3
- **seeed_xiao_esp32s3** - Seeed Studio XIAO ESP32S3
- **waveshare-esp32-driver** - Waveshare ESP32 e-paper driver board
- **seeed_reTerminal_E1001** - Seeed reTerminal E1001

### Testing
- **native** - Run unit tests locally (Linux/Mac)
- **native-windows** - Run unit tests on Windows

## Development Workflow

### Initial Setup
1. Install VS Code
2. Install PlatformIO extension
3. Clone repository
4. Open project in VS Code
5. PlatformIO will auto-install dependencies

### Building
```bash
# Build default environment (trmnl)
pio run

# Build specific environment
pio run -e local

# Build all environments
pio run -a
```

### Flashing Hardware
1. Turn off device
2. Connect via USB-C
3. Hold BOOT button, turn on device, release BOOT (enters flash mode)
4. Select COM port in PlatformIO
5. Click "PlatformIO: Upload"

### Testing
```bash
# Run native tests
pio test -e native

# Run on-device tests
pio test -e trmnl
```

### Code Quality
- Static analysis: `clangtidy` configured for bug detection
- Checks: `bugprone-*`, `portability-*`, `clang-analyzer-*`, `google-*`

## Key Files to Understand

### src/bl.cpp (Business Logic)
The main state machine implementing the algorithm flowchart in README.md:
- WiFi connection management
- Setup/registration flow
- Display update loop
- Sleep/wake cycle
- OTA firmware updates
- Error handling and display

### src/display.cpp
E-ink display driver interface:
- BMP/PNG/JPEG image decoding and rendering
- Display initialization and refresh
- Power management for display
- Image format conversion

### include/config.h
Build-time configuration:
- Hardware pin definitions
- Feature flags
- Network settings
- Display parameters

### src/preferences_persistence.cpp
Non-volatile storage (NVS) for:
- WiFi credentials
- API key and Friendly ID
- Device settings
- Last image filename (for deduplication)

### lib/wificaptive/
Custom WiFi captive portal implementation:
- SSID: "TRMNL Setup"
- User enters WiFi credentials via web interface
- Saves credentials to NVS
- Option to reset device (clear API keys)

## Important Considerations

### Memory Constraints
- ESP32-C3 has limited RAM (~400KB available)
- PNG_MAX_BUFFERED_PIXELS defined per environment to control memory usage
- Avoid large stack allocations
- Be mindful of String vs const char* usage

### Security
- API key stored in NVS (encrypted storage)
- HTTPS for API communication (certificates embedded)
- No credentials in code - always user-provided via captive portal

### Image Handling
- Supports BMP, PNG (up to 32bpp), JPEG
- Images downloaded via HTTP/HTTPS to SPIFFS
- Filename comparison prevents redundant display updates
- Display uses 1-bit (black/white) or multi-bit grayscale depending on hardware

### Error Handling
- Errors displayed as QR codes or text on e-ink display
- Logs submitted to /api/log endpoint
- Device enters sleep after errors (conserve battery)
- User can trigger WiFi reset via long button press (>5 seconds)

### OTA Updates
- Server can trigger firmware update via `update_firmware` flag
- Device downloads .bin file from `firmware_url`
- Update applied on next reboot
- Fallback to previous firmware if update fails

## Common Tasks

### Adding a New Board Configuration
1. Define new `[env:myboard]` in `platformio.ini`
2. Extend appropriate base environment
3. Set board-specific flags (e.g., `-D BOARD_MYBOARD`)
4. Update `include/config.h` with pin mappings
5. Test build and flash

### Modifying API Endpoints
1. Update API client in `src/api-client/`
2. Modify request headers in corresponding .cpp file
3. Update JSON parsing logic for response
4. Test with mock server or staging environment
5. Update README.md API documentation

### Changing Display Behavior
1. Modify `src/display.cpp` for rendering logic
2. Update `src/bl.cpp` for high-level display flow
3. Consider power consumption impact
4. Test with actual e-ink hardware (emulator unavailable)

### Adding New Tests
1. Create `test/test_myfeature/` directory
2. Add `myfeature.test.cpp` with Unity tests
3. Update `platformio.ini` if needed
4. Run `pio test -e native`

### Debugging
- Use `local` environment for serial output
- Set `CORE_DEBUG_LEVEL` in build flags (0-5, higher = more verbose)
- Check `monitor_filters = esp32_exception_decoder` for crash analysis
- Use `debug_tool = esp-builtin` for hardware debugging

## Git Workflow

### Branch Strategy
- **Main branch**: Production-ready code
- **Feature branches**: Start with `claude/` prefix (e.g., `claude/update-claude-md-VDv52`)
- Always develop on designated feature branch
- Create PR to main when ready

### Commits
- Use descriptive commit messages
- Reference issue numbers if applicable
- Ensure tests pass before committing

### Pushing Code
```bash
# Push to feature branch
git push -u origin claude/your-branch-name
```

Note: Branches must start with `claude/` prefix or push will fail with 403.

## Dependencies

### Core Libraries (platformio.ini)
- **ArduinoJson 7.4.2** - JSON parsing/serialization
- **PNGdec 1.1.6** - PNG image decoding
- **JPEGDEC 1.8.4** - JPEG image decoding
- **ESPAsyncWebServer 3.7.7** - Async HTTP server for captive portal
- **Arduino-Log** - Logging framework

### Platform
- **espressif32 6.12.0** - ESP32 platform for PlatformIO
- Arduino framework + ESP-IDF

## Useful Resources

- **Official Documentation**: https://usetrmnl.com
- **Web Flash Tool**: https://usetrmnl.com/flash (no-code firmware updates)
- **Issue Tracker**: https://github.com/usetrmnl/firmware/issues
- **Release Notes**: https://github.com/usetrmnl/firmware/releases

## Version Information

Current version: See recent commits (tags follow semantic versioning)
Latest stable: Check releases page
Git hash embedded in firmware via `scripts/git_version.py`

---

**Last Updated**: 2026-01-09
**Maintained By**: TRMNL Team (https://usetrmnl.com)
**License**: See repository root for license information
