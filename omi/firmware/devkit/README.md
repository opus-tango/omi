# OMI Firmware - DevKit

## Overview

This is the firmware for the OMI (Open Memory Interface) DevKit, built on the Nordic nRF52840 SoC using the Zephyr RTOS. The firmware implements a wearable audio recording device with Bluetooth Low Energy (BLE) connectivity, audio compression using Opus codec, SD card storage, and various peripherals including LEDs, buttons, microphone, and speaker.

## Hardware Platform

- **Board**: Seeed XIAO nRF52840 Sense
- **SoC**: Nordic nRF52840 (ARM Cortex-M4)
- **Operating System**: Zephyr RTOS

## Project Structure

```
devkit/
├── src/
│   ├── main.c              # Main application entry point
│   ├── button.c/h          # Button handling and state machine
│   ├── led.c/h             # LED control functions
│   ├── sdcard.c/h          # SD card file system operations
│   ├── storage.c/h         # BLE storage service for data transfer
│   ├── mic.c/h             # PDM microphone driver
│   ├── codec.c/h           # Opus audio codec interface
│   ├── transport.c/h       # BLE transport layer
│   ├── speaker.c/h         # Speaker/haptic feedback
│   ├── usb.c/h             # USB charging detection
│   ├── config.h            # System configuration constants
│   └── lib/
│       ├── battery/        # Battery management
│       └── opus-1.2.1/     # Opus audio codec library
├── CMakeLists.txt          # Build configuration
└── prj_*.conf              # Zephyr project configuration files
```

## Key Features

- **Audio Recording**: 16kHz PDM microphone with Opus compression
- **BLE Connectivity**: Audio streaming and storage transfer over BLE
- **SD Card Storage**: FAT filesystem for offline audio storage
- **Button Interface**: Single tap, double tap, and long press detection
- **LED Feedback**: RGB LED status indicators
- **Battery Management**: Battery monitoring and USB charging detection
- **Power Management**: Low power modes and system off capability

---

## LED Control System

### Architecture

The LED control system uses three GPIO pins to control an RGB LED (red, green, and blue channels). The LEDs are accessed through Zephyr's GPIO devicetree API.

### Hardware Configuration

The LEDs are defined using devicetree aliases (src/led.h:7-9):
- `led0` (Red LED) - GPIO alias
- `led1` (Green LED) - GPIO alias
- `led2` (Blue LED) - GPIO alias

### Implementation Details

#### Initialization (src/led.c:10-20)

The `led_start()` function initializes all three LEDs:
1. Checks if each GPIO device is ready using `gpio_is_ready_dt()`
2. Configures each pin as output with inactive state using `gpio_pin_configure_dt()`
3. All LEDs start in the OFF state

#### Control Functions (src/led.c:22-35)

Three simple functions control each LED:
- `set_led_red(bool on)` - Controls red LED
- `set_led_green(bool on)` - Controls green LED
- `set_led_blue(bool on)` - Controls blue LED

Each function uses `gpio_pin_set_dt()` to set the pin state (high/low).

### LED State Machine (src/main.c:75-106)

The firmware implements a sophisticated LED state machine that runs in the main loop every 500ms:

#### State Indicators

1. **USB Charging** (Green LED):
   - Toggles green LED on/off when USB charging is detected
   - Overrides other states when active

2. **Device Off** (All LEDs Off):
   - When `is_off` flag is true, all LEDs turn off
   - Indicates low-power/sleep state

3. **BLE Connected** (Blue LED):
   - Solid blue LED indicates active BLE connection
   - Red LED turns off in this state

4. **BLE Disconnected** (Red LED):
   - Solid red LED indicates recording but no BLE connection
   - Blue LED turns off in this state

#### Boot Sequence (src/main.c:47-73)

On startup, a special LED sequence plays:
1. Red blink (600ms on, 200ms pause)
2. Green blink (600ms on, 200ms pause)
3. Blue blink (600ms on, 200ms pause)
4. All LEDs on together (600ms)
5. All LEDs off

This sequence provides visual feedback that the device has booted successfully and all LED channels are functional.

---

## File Saving System (SD Card Storage)

### Architecture

The file saving system uses a FAT filesystem on an SD card connected via SPI. It implements a FIFO-style audio file management system with support for reading, writing, and transferring audio data over BLE.

### SD Card Hardware Interface

- **Enable Pin**: GPIO pin 19 (P0.19) controls SD card power (src/sdcard.c:23-25)
- **SPI Interface**: SPI2 peripheral (P1.15 MOSI, P1.14 MISO, P1.13 SCK, P0.02 CS)
- **Filesystem**: FAT filesystem using FatFs library

### File System Structure

```
/SD:/
├── audio/
│   ├── a01.txt    # Audio file 1
│   ├── a02.txt    # Audio file 2 (if exists)
│   └── ...
└── info.txt       # Stores read/write offset metadata
```

### Initialization Process (src/sdcard.c:40-136)

The `mount_sd_card()` function performs the following steps:

1. **Power On SD Card**:
   - Configures GPIO pin 19 as output active
   - Powers up the SD card module

2. **Initialize Disk Access**:
   - Calls `disk_access_init("SD")` to initialize the disk driver
   - Retries once after 1 second if initial attempt fails

3. **Mount Filesystem**:
   - Mounts FAT filesystem at `/SD:` mount point
   - Returns error if mount fails

4. **Create Audio Directory**:
   - Creates `/SD:/audio` directory if it doesn't exist
   - Initializes first audio file `a01.txt` if directory is new

5. **Initialize File Pointers**:
   - Sets write pointer to the current file
   - Sets read pointer for BLE data transfer
   - Creates `info.txt` if it doesn't exist (stores offset for resuming transfers)

### File Naming Convention (src/sdcard.c:237-258)

Audio files follow the pattern `audio/aNN.txt` where NN is a two-digit number (01-99):
- The `generate_new_audio_header()` function creates these filenames dynamically
- Files are numbered sequentially starting from `a01.txt`

### Write Operations (src/sdcard.c:215-224)

The `write_to_file()` function:
1. Opens the file at the current write pointer
2. Uses `FS_O_APPEND` flag to append data to the end
3. Writes the data buffer to the file
4. Closes the file

This is used to continuously append encoded audio data to the active file.

### Read Operations (src/sdcard.c:195-213)

The `read_audio_data()` function:
1. Opens the file at the current read pointer
2. Seeks to the specified offset using `fs_seek()`
3. Reads the requested amount of data
4. Closes the file and returns the number of bytes read

This is used by the BLE storage service to retrieve audio data for transfer to a connected device.

### File Management Operations

#### Clear Audio File (src/sdcard.c:285-306)
- Deletes the specified audio file
- Immediately recreates an empty file with the same name
- Preserves the FIFO structure

#### Clear Audio Directory (src/sdcard.c:322-360)
- Deletes all audio files in the directory
- Removes and recreates the `/SD:/audio` directory
- Creates a fresh `a01.txt` file
- Resets file count and pointers
- This is the "nuclear option" for clearing all stored data

#### Offset Management (src/sdcard.c:362-408)
- `save_offset()`: Saves a 32-bit offset value to `info.txt`
- `get_offset()`: Reads the offset from `info.txt`
- Used to resume BLE transfers from where they left off

### Power Management (src/sdcard.c:410-444)

#### SD Card Off (`sd_off()`):
- Suspends the SPI2 peripheral to save power
- Disconnects all SPI pins (MOSI, MISO, SCK, CS)
- Disables SD card power via enable pin
- Critical for battery life during idle periods

#### SD Card On (`sd_on()`):
- Re-enables SD card power
- Reconfigures all SPI pins
- Resumes the SPI2 peripheral
- Prepares SD card for read/write operations

### Storage Service (BLE Transfer)

The `storage.c` module implements a BLE GATT service for transferring audio files to a connected device:

#### Service UUIDs (src/storage.c:43-48)
- **Service**: `30295780-4301-EABD-2904-2849ADFEAE43`
- **Write Characteristic**: `30295781-4301-EABD-2904-2849ADFEAE43`
- **Read Characteristic**: `30295782-4301-EABD-2904-2849ADFEAE43`

#### Commands (src/storage.c:24-27)
- `READ_COMMAND (0)`: Start reading a specific audio file
- `DELETE_COMMAND (1)`: Clear a specific audio file
- `NUKE (2)`: Clear entire audio directory
- `STOP_COMMAND (3)`: Stop current transfer and save progress
- `HEARTBEAT (50)`: Keep connection alive during transfer

#### Transfer Process (src/storage.c:272-286)
1. Client sends READ command with file number and offset
2. Firmware reads 440-byte chunks from SD card
3. Data is sent via BLE notifications
4. Progress is saved to `info.txt` periodically
5. Transfer can resume from saved offset if interrupted

#### Storage Thread (src/storage.c:288-360)
- Runs as a separate Zephyr thread (priority 7)
- Continuously checks for pending operations
- Handles read, delete, and nuke commands
- Manages heartbeat timeout (100 frames max)
- Automatically saves offset when connection is lost

---

## Button Press Detection System

### Architecture

The button system implements a sophisticated finite state machine (FSM) that detects single taps, double taps, long presses, and raw press/release events. It uses GPIO interrupt-driven detection with debouncing and timing logic.

### Hardware Configuration (src/button.c:57-62)

- **D4 Pin (P0.4)**: Configured as output to provide 3.3V to button circuit
- **D5 Pin (P0.5)**: Configured as input with edge detection for button state
- **Interrupt Mode**: `GPIO_INT_EDGE_BOTH` - triggers on both rising and falling edges

### Initialization (src/button.c:453-496)

The `button_init()` function:
1. Configures D4 pin as active output (provides voltage to button)
2. Configures D5 pin as input with interrupt capability
3. Sets up edge detection on D5 (both rising and falling edges)
4. Registers `button_pressed_callback` as the interrupt handler
5. Adds GPIO callback to handle button state changes

### Interrupt Handler (src/button.c:69-78)

The `button_pressed_callback()` runs on each button state change:
- Reads the raw GPIO pin state
- Updates `was_pressed` flag based on pin level
- Low level = button pressed, High level = button released
- This provides the raw input to the state machine

### Button State Machine

#### States (src/button.h:4)
- `IDLE`: No button activity
- `GRACE`: Post-action grace period before returning to IDLE

#### Timing Constants (src/button.c:158-160)
- `TAP_THRESHOLD`: 300ms - Maximum duration for a tap
- `DOUBLE_TAP_WINDOW`: 600ms - Maximum time between two taps for double-tap
- `LONG_PRESS_TIME`: 1000ms - Minimum duration for long press
- `BUTTON_CHECK_INTERVAL`: 40ms - State machine polling rate (25 Hz)

#### Event Types (src/button.c:162-168)
- `BUTTON_EVENT_NONE`: No event
- `BUTTON_EVENT_SINGLE_TAP`: Quick press and release
- `BUTTON_EVENT_DOUBLE_TAP`: Two quick taps in succession
- `BUTTON_EVENT_LONG_PRESS`: Button held for >1 second
- `BUTTON_EVENT_RELEASE`: Button released after press

### State Machine Logic (src/button.c:178-268)

The `check_button_level()` function runs every 40ms as a delayed work item:

#### Press Detection (Lines 187-189)
```c
if (btn_state == BUTTON_PRESSED && !btn_is_pressed) {
    btn_is_pressed = true;
    btn_press_start_time = current_time;
}
```
- Records timestamp when button is first pressed
- Sets flag to track pressed state

#### Release Detection (Lines 190-205)
```c
else if (btn_state == BUTTON_RELEASED && btn_is_pressed) {
    btn_is_pressed = false;
    btn_release_time = current_time;
    // Check for double tap
    uint32_t press_duration = (btn_release_time - btn_press_start_time) * BUTTON_CHECK_INTERVAL;
    if (press_duration < TAP_THRESHOLD) {
        if (btn_last_tap_time > 0 &&
            (current_time - btn_last_tap_time) * BUTTON_CHECK_INTERVAL < DOUBLE_TAP_WINDOW) {
            event = BUTTON_EVENT_DOUBLE_TAP;
            btn_last_tap_time = 0;
        } else {
            btn_last_tap_time = current_time;
        }
    }
}
```
- Records timestamp when button is released
- Checks if press was short enough to be a tap (<300ms)
- If a previous tap occurred within 600ms, triggers double-tap event
- Otherwise, records this as a potential first tap

#### Single Tap Detection (Lines 207-217)
- After release, checks if enough time has passed since press start
- If no second tap occurs, generates single-tap event
- Resets tap timer after detection

#### Long Press Detection (Lines 219-222)
```c
if (btn_is_pressed &&
    (current_time - btn_press_start_time) * BUTTON_CHECK_INTERVAL >= LONG_PRESS_TIME) {
    event = BUTTON_EVENT_LONG_PRESS;
}
```
- Continuously checks while button is held
- Triggers when held for 1000ms or more

### Event Handlers

#### Single Tap (src/button.c:225-234)
- Logs "single tap detected"
- Sends BLE notification with `SINGLE_TAP` event
- **Triggers low power mode**:
  - Sets `is_off = true`
  - Calls `bt_off()` to disable Bluetooth
  - Calls `turnoff_all()` to shut down peripherals

#### Double Tap (src/button.c:237-241)
- Logs "double tap detected"
- Sends BLE notification with `DOUBLE_TAP` event
- Device remains active

#### Long Press (src/button.c:244-248)
- Logs "long press detected"
- Sends BLE notification with `LONG_TAP` event
- Only fires once (checked with `btn_last_event`)
- Device remains active

#### Release (src/button.c:251-264)
- Logs "release detected"
- Sends BLE notification with `BUTTON_RELEASE` event
- Resets all timing variables
- Transitions to `GRACE` state
- Only fires once per press cycle

### Power Down Sequence (src/button.c:513-530)

The `turnoff_all()` function is called on single tap:
1. Turns off microphone (`mic_off()`)
2. Turns off SD card (`sd_off()`)
3. Turns off speaker (`speaker_off()`)
4. Turns off accelerometer (`accel_off()`)
5. Plays 50ms haptic feedback
6. Turns off all LEDs (red, green, blue)
7. Removes button GPIO callback
8. Disables button interrupt
9. Disables USB interrupts
10. Enters Nordic system off mode (`NRF_POWER->SYSTEMOFF = 1`)

The device can only be woken by pressing the button again, which triggers a hardware reset.

### BLE Notifications (src/button.c:105-153)

All button events are sent via BLE GATT notifications:

#### Service UUID (src/button.c:29-30)
- **Service**: `23BA7924-0000-1000-7450-346EAC492E92`
- **Characteristic**: `23BA7925-0000-1000-7450-346EAC492E92`

#### Button States Sent (src/button.c:85-90)
- `SINGLE_TAP (1)`
- `DOUBLE_TAP (2)`
- `LONG_TAP (3)`
- `BUTTON_PRESS (4)`
- `BUTTON_RELEASE (5)`

The notification includes a 2-element array where the first element is the event type.

### Work Queue Management

The button state machine uses Zephyr's delayed work API:
- `K_WORK_DELAYABLE_DEFINE(button_work, check_button_level)` - Defines the work item
- `activate_button_work()` - Called from main to start the state machine
- `k_work_reschedule(&button_work, K_MSEC(BUTTON_CHECK_INTERVAL))` - Reschedules every 40ms

This creates a non-blocking periodic task that doesn't interfere with other system operations.

---

## Audio Pipeline

### Flow

1. **Microphone** (`mic.c`):
   - PDM microphone captures audio at 16kHz
   - Buffers 100ms chunks (1600 samples)
   - Calls `mic_handler()` callback

2. **Codec** (`codec.c`):
   - Receives PCM data from microphone
   - Encodes using Opus codec (32kbps, 160 sample frames)
   - Calls `codec_handler()` callback with compressed data

3. **Transport** (`transport.c`):
   - Broadcasts encoded audio packets over BLE
   - Manages BLE connection and characteristics

4. **Storage** (Optional):
   - Writes encoded audio to SD card
   - Can retrieve and transfer stored audio later

## Configuration

The system uses several configuration files:
- `prj_xiao_ble_sense_devkitv1.conf` - DevKit v1 configuration
- `prj_xiao_ble_sense_devkitv1-spisd.conf` - DevKit v1 with SPI SD card
- `prj_xiao_ble_sense_devkitv2-adafruit.conf` - DevKit v2 configuration

Key configuration options (from `config.h`):
- `MIC_GAIN`: 64
- `MIC_BUFFER_SAMPLES`: 1600 (100ms at 16kHz)
- `AUDIO_BUFFER_SAMPLES`: 16000 (1 second)
- `CODEC_OPUS_BITRATE`: 32000 bps
- `CODEC_OPUS_COMPLEXITY`: 3

## Build System

The project uses CMake with Zephyr SDK:
- Main build file: `CMakeLists.txt`
- Board: `seeed_xiao_nrf52840_sense`
- Key source files compiled into single application
- Conditional Opus codec inclusion based on `CONFIG_OMI_CODEC_OPUS`

## Main Loop

The main function (src/main.c:108-336):
1. Enables DC-DC converters for power efficiency
2. Initializes all peripherals in sequence
3. Runs boot LED sequence
4. Starts transport, codec, and microphone
5. Enters infinite loop updating LED state every 500ms

## Power Management

The firmware implements several power-saving features:
- QSPI flash forced into deep sleep mode
- DC-DC converters enabled for efficiency
- SD card can be powered off when not in use
- System off mode triggered by single tap
- Wake from system off via button press (hardware reset)

## Dependencies

- **Zephyr RTOS**: Real-time operating system
- **Nordic nRF SDK**: BLE stack and hardware drivers
- **FatFs**: FAT filesystem for SD card
- **Opus 1.2.1**: Audio compression codec (fixed-point ARM optimized)

## Development

### Flashing
Use the provided `flash.sh` script to flash the firmware to the device.

### Logging
The firmware uses Zephyr's logging system with module-specific log levels:
- `LOG_MODULE_REGISTER(module_name, CONFIG_LOG_DEFAULT_LEVEL)`
- Log levels can be adjusted in project configuration

### Debugging
Debug output is available via:
- USB CDC ACM serial console
- Segger RTT (if configured)
- UART (if configured)

## License

SPDX-License-Identifier: Apache-2.0