Firmware Modification Plan: Event Logging & Audio Recording Updates                                                                                                 │
     │                                                                                                                                                                     │
     │ Overview                                                                                                                                                            │
     │                                                                                                                                                                     │
     │ Modify OMI firmware to remove BLE audio services, add triple-tap detection, enable always-on SD card recording, and implement button event logging with timestamps. │
     │                                                                                                                                                                     │
     │ Requirements Summary                                                                                                                                                │
     │                                                                                                                                                                     │
     │ 1. ✅ Remove all audio BLE services (live streaming + storage transfer)                                                                                             │
     │ 2. ✅ Add triple button press detection (3 taps within 900ms)                                                                                                       │
     │ 3. ✅ Always record audio to SD card (remove conditional logic)                                                                                                     │
     │ 4. ✅ Create CSV event log on SD card (/SD:/events.csv)                                                                                                             │
     │ 5. ✅ Update button functionality:                                                                                                                                  │
     │   - Single tap: Power on/off (unchanged)                                                                                                                            │
     │   - Double tap: Log "MARKER1" timestamp                                                                                                                             │
     │   - Triple tap: Log "MARKER2" timestamp                                                                                                                             │
     │   - Long press: Toggle LEDs on/off (changed)                                                                                                                        │
     │                                                                                                                                                                     │
     │ User Specifications                                                                                                                                                 │
     │                                                                                                                                                                     │
     │ - Log Format: CSV with timestamp,event_type                                                                                                                         │
     │ - Timestamp: Audio sample count at 16kHz (samples since boot)                                                                                                       │
     │ - Log Location: /SD:/events.csv                                                                                                                                     │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Implementation Phases                                                                                                                                               │
     │                                                                                                                                                                     │
     │ Phase 1: Create Event Logging System (NEW MODULE)                                                                                                                   │
     │                                                                                                                                                                     │
     │ Create Files:                                                                                                                                                       │
     │ - src/event_logger.c - Event logging implementation                                                                                                                 │
     │ - src/event_logger.h - Public API                                                                                                                                   │
     │                                                                                                                                                                     │
     │ Key Components:                                                                                                                                                     │
     │                                                                                                                                                                     │
     │ // Event types                                                                                                                                                      │
     │ typedef enum {                                                                                                                                                      │
     │     EVENT_POWER_ON = 0,                                                                                                                                             │
     │     EVENT_POWER_OFF = 1,                                                                                                                                            │
     │     EVENT_MARKER1 = 2,      // Double tap                                                                                                                           │
     │     EVENT_MARKER2 = 3,      // Triple tap                                                                                                                           │
     │     EVENT_LED_TOGGLE = 4    // Long press                                                                                                                           │
     │ } ButtonEventType;                                                                                                                                                  │
     │                                                                                                                                                                     │
     │ // Core functions                                                                                                                                                   │
     │ int event_logger_init()                                    // Initialize, create/open events.csv                                                                    │
     │ int log_button_event(ButtonEventType type, uint64_t ts)   // Append to CSV                                                                                          │
     │ uint64_t get_audio_sample_timestamp()                     // Get current sample count                                                                               │
     │ void increment_audio_samples(uint32_t samples)            // Called from mic handler                                                                                │
     │                                                                                                                                                                     │
     │ Implementation Details:                                                                                                                                             │
     │ - Keep events.csv file handle open for performance                                                                                                                  │
     │ - Use mutex for thread-safe access (write_sdcard_mutex)                                                                                                             │
     │ - Buffer writes (256 bytes) to reduce SD wear                                                                                                                       │
     │ - CSV format: timestamp,event_type\n (e.g., 80000,2)                                                                                                                │
     │ - Queue up to 100 events in RAM if SD unavailable                                                                                                                   │
     │                                                                                                                                                                     │
     │ File: src/event_logger.c:1-250 (new file, ~250 lines)                                                                                                               │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Phase 2: Add Triple-Tap Detection (MODIFY BUTTON)                                                                                                                   │
     │                                                                                                                                                                     │
     │ File: src/button.c                                                                                                                                                  │
     │                                                                                                                                                                     │
     │ Changes:                                                                                                                                                            │
     │                                                                                                                                                                     │
     │ 1. Add constants (after line 88):                                                                                                                                   │
     │ #define TRIPLE_TAP 6                                                                                                                                                │
     │ #define TRIPLE_TAP_WINDOW 900  // 900ms for three taps                                                                                                              │
     │                                                                                                                                                                     │
     │ 2. Add state variables (after line 176):                                                                                                                            │
     │ static uint8_t btn_tap_count = 0;                                                                                                                                   │
     │ static uint32_t btn_first_tap_time = 0;                                                                                                                             │
     │                                                                                                                                                                     │
     │ 3. Extend ButtonEvent enum (line 162):                                                                                                                              │
     │ BUTTON_EVENT_TRIPLE_TAP,  // Add after BUTTON_EVENT_DOUBLE_TAP                                                                                                      │
     │                                                                                                                                                                     │
     │ 4. Modify check_button_level() detection logic (lines 190-222):                                                                                                     │
     │   - On first tap: Set btn_tap_count = 1, record btn_first_tap_time                                                                                                  │
     │   - On subsequent tap within 900ms: Increment btn_tap_count                                                                                                         │
     │   - If btn_tap_count == 3: Trigger BUTTON_EVENT_TRIPLE_TAP                                                                                                          │
     │   - Handle timeout: If 2 taps and no 3rd within window → DOUBLE_TAP                                                                                                 │
     │ 5. Add triple-tap handler (after line 241):                                                                                                                         │
     │ if (event == BUTTON_EVENT_TRIPLE_TAP) {                                                                                                                             │
     │     LOG_PRINTK("triple tap detected\n");                                                                                                                            │
     │     btn_last_event = event;                                                                                                                                         │
     │     notify_triple_tap();                                                                                                                                            │
     │     log_button_event(EVENT_MARKER2, get_audio_sample_timestamp());                                                                                                  │
     │ }                                                                                                                                                                   │
     │                                                                                                                                                                     │
     │ 6. Add notification function (after line 153):                                                                                                                      │
     │ static inline void notify_triple_tap() {                                                                                                                            │
     │     final_button_state[0] = TRIPLE_TAP;                                                                                                                             │
     │     LOG_INF("Button triple tap");                                                                                                                                   │
     │     struct bt_conn *conn = get_current_connection();                                                                                                                │
     │     if (conn != NULL) {                                                                                                                                             │
     │         bt_gatt_notify(conn, &button_service.attrs[1],                                                                                                              │
     │                       &final_button_state, sizeof(final_button_state));                                                                                             │
     │     }                                                                                                                                                               │
     │ }                                                                                                                                                                   │
     │                                                                                                                                                                     │
     │ Files Modified:                                                                                                                                                     │
     │ - src/button.c:88,153,162,176,190-222,241 (~80 lines modified)                                                                                                      │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Phase 3: Update Button Event Handlers (MODIFY BUTTON)                                                                                                               │
     │                                                                                                                                                                     │
     │ File: src/button.c                                                                                                                                                  │
     │                                                                                                                                                                     │
     │ Changes:                                                                                                                                                            │
     │                                                                                                                                                                     │
     │ 1. Single tap handler (line 225-234) - Add event logging before power-off:                                                                                          │
     │ log_button_event(EVENT_POWER_OFF, get_audio_sample_timestamp());                                                                                                    │
     │ // Then existing: is_off = true; bt_off(); turnoff_all();                                                                                                           │
     │                                                                                                                                                                     │
     │ 2. Double tap handler (line 237-241) - Add event logging:                                                                                                           │
     │ log_button_event(EVENT_MARKER1, get_audio_sample_timestamp());                                                                                                      │
     │                                                                                                                                                                     │
     │ 3. Long press handler (line 244-248) - CHANGE BEHAVIOR to LED toggle:                                                                                               │
     │ static bool leds_enabled = true;  // Add global variable                                                                                                            │
     │                                                                                                                                                                     │
     │ if (event == BUTTON_EVENT_LONG_PRESS && btn_last_event != BUTTON_EVENT_LONG_PRESS) {                                                                                │
     │     LOG_PRINTK("long press - toggling LEDs\n");                                                                                                                     │
     │     btn_last_event = event;                                                                                                                                         │
     │     notify_long_tap();                                                                                                                                              │
     │                                                                                                                                                                     │
     │     leds_enabled = !leds_enabled;                                                                                                                                   │
     │     if (!leds_enabled) {                                                                                                                                            │
     │         set_led_red(false);                                                                                                                                         │
     │         set_led_green(false);                                                                                                                                       │
     │         set_led_blue(false);                                                                                                                                        │
     │     }                                                                                                                                                               │
     │     log_button_event(EVENT_LED_TOGGLE, get_audio_sample_timestamp());                                                                                               │
     │ }                                                                                                                                                                   │
     │                                                                                                                                                                     │
     │ 4. Export LED state in src/button.h:                                                                                                                                │
     │ extern bool leds_enabled;                                                                                                                                           │
     │                                                                                                                                                                     │
     │ 5. Update main LED control in src/main.c (line 75):                                                                                                                 │
     │ void set_led_state() {                                                                                                                                              │
     │     if (!leds_enabled) return;  // Add this check first                                                                                                             │
     │     // ... existing LED logic                                                                                                                                       │
     │ }                                                                                                                                                                   │
     │                                                                                                                                                                     │
     │ Files Modified:                                                                                                                                                     │
     │ - src/button.c:225,237,244 (~30 lines modified)                                                                                                                     │
     │ - src/button.h:14 (1 line added)                                                                                                                                    │
     │ - src/main.c:75 (2 lines added)                                                                                                                                     │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Phase 4: Always Record Audio to SD Card (MODIFY TRANSPORT)                                                                                                          │
     │                                                                                                                                                                     │
     │ File: src/transport.c                                                                                                                                               │
     │                                                                                                                                                                     │
     │ Current Logic (lines 729-747): Audio writes to SD ONLY when !valid && !storage_is_on                                                                                │
     │                                                                                                                                                                     │
     │ NEW Logic: Audio ALWAYS writes to SD, regardless of BLE connection state                                                                                            │
     │                                                                                                                                                                     │
     │ Changes at line 729:                                                                                                                                                │
     │                                                                                                                                                                     │
     │ // ALWAYS write to SD card                                                                                                                                          │
     │ bool sd_write_result = false;                                                                                                                                       │
     │ if (file_num_array[1] < MAX_STORAGE_BYTES) {                                                                                                                        │
     │     k_mutex_lock(&write_sdcard_mutex, K_FOREVER);                                                                                                                   │
     │     if (is_sd_on()) {                                                                                                                                               │
     │         sd_write_result = write_to_storage();                                                                                                                       │
     │     }                                                                                                                                                               │
     │     k_mutex_unlock(&write_sdcard_mutex);                                                                                                                            │
     │                                                                                                                                                                     │
     │     if (sd_write_result) {                                                                                                                                          │
     │         heartbeat_count++;                                                                                                                                          │
     │         if (heartbeat_count == 255) {                                                                                                                               │
     │             update_file_size();                                                                                                                                     │
     │             heartbeat_count = 0;                                                                                                                                    │
     │         }                                                                                                                                                           │
     │     }                                                                                                                                                               │
     │ }                                                                                                                                                                   │
     │                                                                                                                                                                     │
     │ // SEPARATELY: Send via BLE if connected                                                                                                                            │
     │ if (valid) {                                                                                                                                                        │
     │     bool sent = push_to_gatt(conn);                                                                                                                                 │
     │     // ... existing BLE transmission logic                                                                                                                          │
     │ }                                                                                                                                                                   │
     │                                                                                                                                                                     │
     │ Key Change: Separate SD writing from BLE transmission. Both can happen simultaneously.                                                                              │
     │                                                                                                                                                                     │
     │ Impact: Audio is continuously written to SD card, AND streamed via BLE when connected.                                                                              │
     │                                                                                                                                                                     │
     │ Files Modified:                                                                                                                                                     │
     │ - src/transport.c:729-747 (~20 lines modified)                                                                                                                      │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Phase 5: Integrate Sample Counter (MODIFY MAIN)                                                                                                                     │
     │                                                                                                                                                                     │
     │ File: src/main.c                                                                                                                                                    │
     │                                                                                                                                                                     │
     │ Modify mic_handler() (line 30-36):                                                                                                                                  │
     │                                                                                                                                                                     │
     │ static void mic_handler(int16_t *buffer)                                                                                                                            │
     │ {                                                                                                                                                                   │
     │     // Track audio samples for timestamp                                                                                                                            │
     │     increment_audio_samples(MIC_BUFFER_SAMPLES);  // Add this line                                                                                                  │
     │                                                                                                                                                                     │
     │     int err = codec_receive_pcm(buffer, MIC_BUFFER_SAMPLES);                                                                                                        │
     │     if (err) {                                                                                                                                                      │
     │         LOG_ERR("Failed to process PCM data: %d", err);                                                                                                             │
     │     }                                                                                                                                                               │
     │ }                                                                                                                                                                   │
     │                                                                                                                                                                     │
     │ Add includes (top of file):                                                                                                                                         │
     │ #include "event_logger.h"                                                                                                                                           │
     │                                                                                                                                                                     │
     │ Files Modified:                                                                                                                                                     │
     │ - src/main.c:1,30 (~2 lines added)                                                                                                                                  │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Phase 6: Remove BLE Audio Services (CLEANUP)                                                                                                                        │
     │                                                                                                                                                                     │
     │ 6.1 Remove Audio Streaming Service                                                                                                                                  │
     │                                                                                                                                                                     │
     │ File: src/transport.c                                                                                                                                               │
     │                                                                                                                                                                     │
     │ Remove/Comment Out:                                                                                                                                                 │
     │ - Audio service UUID (lines 80-87)                                                                                                                                  │
     │ - Audio service characteristics array (lines 89-114)                                                                                                                │
     │ - Audio service struct (line 116)                                                                                                                                   │
     │ - Audio CCC handlers (lines 282-327)                                                                                                                                │
     │ - Audio service registration (line 838): bt_gatt_service_register(&audio_service);                                                                                  │
     │ - Audio service from advertisement (line 270)                                                                                                                       │
     │                                                                                                                                                                     │
     │ Keep:                                                                                                                                                               │
     │ - broadcast_audio_packets() function (needed internally for SD writing)                                                                                             │
     │ - Ring buffer system (needed for audio buffering)                                                                                                                   │
     │ - Pusher thread (modify to handle only SD writes, not BLE)                                                                                                          │
     │                                                                                                                                                                     │
     │ Files Modified:                                                                                                                                                     │
     │ - src/transport.c:80-116,270,282-327,838 (~120 lines removed/commented)                                                                                             │
     │                                                                                                                                                                     │
     │ 6.2 Remove Storage Transfer Service                                                                                                                                 │
     │                                                                                                                                                                     │
     │ File: src/storage.c                                                                                                                                                 │
     │                                                                                                                                                                     │
     │ Remove/Comment Out:                                                                                                                                                 │
     │ - Storage service UUIDs (lines 43-48)                                                                                                                               │
     │ - Storage service characteristics (lines 62-79)                                                                                                                     │
     │ - Storage service struct (line 81)                                                                                                                                  │
     │ - Storage thread (lines 288-360)                                                                                                                                    │
     │ - Command parsing (lines 162-224)                                                                                                                                   │
     │                                                                                                                                                                     │
     │ File: src/transport.c                                                                                                                                               │
     │ - Remove storage service registration (line 837)                                                                                                                    │
     │                                                                                                                                                                     │
     │ File: src/main.c                                                                                                                                                    │
     │ - Remove storage init call (lines 210-214): Keep mount_sd_card() but remove storage_init()                                                                          │
     │                                                                                                                                                                     │
     │ Files Modified:                                                                                                                                                     │
     │ - src/storage.c:43-81,162-224,288-360 (~200 lines removed/commented)                                                                                                │
     │ - src/transport.c:837 (1 line removed)                                                                                                                              │
     │ - src/main.c:213 (1 line removed)                                                                                                                                   │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Phase 7: Build System Updates                                                                                                                                       │
     │                                                                                                                                                                     │
     │ File: CMakeLists.txt                                                                                                                                                │
     │                                                                                                                                                                     │
     │ Add event_logger.c (line 18):                                                                                                                                       │
     │ target_sources(app PRIVATE                                                                                                                                          │
     │     src/main.c                                                                                                                                                      │
     │     src/button.c                                                                                                                                                    │
     │     src/event_logger.c  # ADD THIS                                                                                                                                  │
     │     # ... rest of sources                                                                                                                                           │
     │ )                                                                                                                                                                   │
     │                                                                                                                                                                     │
     │ File: src/config.h                                                                                                                                                  │
     │                                                                                                                                                                     │
     │ Add configuration (end of file):                                                                                                                                    │
     │ // Event logging configuration                                                                                                                                      │
     │ #define CONFIG_OMI_ENABLE_EVENT_LOGGING 1                                                                                                                           │
     │ #define EVENT_LOG_FILE_PATH "/SD:/events.csv"                                                                                                                       │
     │ #define EVENT_LOG_BUFFER_SIZE 256                                                                                                                                   │
     │ #define EVENT_LOG_QUEUE_SIZE 100                                                                                                                                    │
     │                                                                                                                                                                     │
     │ Files Modified:                                                                                                                                                     │
     │ - CMakeLists.txt:18 (1 line added)                                                                                                                                  │
     │ - src/config.h:42 (5 lines added)                                                                                                                                   │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Critical Files Summary                                                                                                                                              │
     │                                                                                                                                                                     │
     │ | File               | Action | Lines Changed         | Priority |                                                                                                  │
     │ |--------------------|--------|-----------------------|----------|                                                                                                  │
     │ | src/event_logger.c | CREATE | +250                  | High     |                                                                                                  │
     │ | src/event_logger.h | CREATE | +20                   | High     |                                                                                                  │
     │ | src/button.c       | MODIFY | ~110 modified         | High     |                                                                                                  │
     │ | src/button.h       | MODIFY | +1                    | Medium   |                                                                                                  │
     │ | src/transport.c    | MODIFY | ~140 modified/removed | High     |                                                                                                  │
     │ | src/storage.c      | MODIFY | ~200 removed          | Medium   |                                                                                                  │
     │ | src/main.c         | MODIFY | ~5 modified           | High     |                                                                                                  │
     │ | src/config.h       | MODIFY | +5                    | Low      |                                                                                                  │
     │ | CMakeLists.txt     | MODIFY | +1                    | Low      |                                                                                                  │
     │                                                                                                                                                                     │
     │ Total Changes: ~270 new lines, ~140 modified lines, ~200 removed lines                                                                                              │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Testing Checklist                                                                                                                                                   │
     │                                                                                                                                                                     │
     │ Unit Tests                                                                                                                                                          │
     │                                                                                                                                                                     │
     │ Event Logger:                                                                                                                                                       │
     │ - events.csv created on boot                                                                                                                                        │
     │ - CSV format correct: timestamp,event_type                                                                                                                          │
     │ - File handle persists across writes                                                                                                                                │
     │ - Mutex prevents race conditions                                                                                                                                    │
     │                                                                                                                                                                     │
     │ Triple-Tap Detection:                                                                                                                                               │
     │ - 3 taps within 900ms → TRIPLE_TAP                                                                                                                                  │
     │ - 2 taps within 600ms → DOUBLE_TAP (unchanged)                                                                                                                      │
     │ - Single tap → SINGLE_TAP (unchanged)                                                                                                                               │
     │ - Edge cases: tap at 899ms, 901ms                                                                                                                                   │
     │                                                                                                                                                                     │
     │ Button Functionality:                                                                                                                                               │
     │ - Single tap powers off (unchanged)                                                                                                                                 │
     │ - Double tap logs MARKER1                                                                                                                                           │
     │ - Triple tap logs MARKER2                                                                                                                                           │
     │ - Long press toggles LEDs                                                                                                                                           │
     │ - All events logged to CSV                                                                                                                                          │
     │                                                                                                                                                                     │
     │ Always-On Recording:                                                                                                                                                │
     │ - Audio writes to SD when BLE disconnected                                                                                                                          │
     │ - Audio writes to SD when BLE connected                                                                                                                             │
     │ - No data loss during BLE state changes                                                                                                                             │
     │ - File size grows continuously                                                                                                                                      │
     │                                                                                                                                                                     │
     │ BLE Service Removal:                                                                                                                                                │
     │ - No audio service in BLE advertisement                                                                                                                             │
     │ - No storage service visible                                                                                                                                        │
     │ - Button/battery/accelerometer services still work                                                                                                                  │
     │                                                                                                                                                                     │
     │ Integration Tests                                                                                                                                                   │
     │                                                                                                                                                                     │
     │ 1. Power Cycle Test: Boot → check POWER_ON in CSV → single tap → check POWER_OFF                                                                                    │
     │ 2. Event Logging Test: Perform all button actions → verify CSV contains all events                                                                                  │
     │ 3. LED Toggle Test: Long press → LEDs off → long press → LEDs on                                                                                                    │
     │ 4. SD Recording Test: Record for 10 minutes → verify continuous audio file growth                                                                                   │
     │ 5. Timestamp Test: Record for 10s → button at 5s → verify timestamp ~80,000 samples                                                                                 │
     │                                                                                                                                                                     │
     │ Long-Term Tests                                                                                                                                                     │
     │                                                                                                                                                                     │
     │ - 1-hour continuous recording (check for buffer overruns)                                                                                                           │
     │ - 100 button events (check CSV file integrity)                                                                                                                      │
     │ - Power-off recovery (ungraceful shutdown, verify no corruption)                                                                                                    │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Implementation Order                                                                                                                                                │
     │                                                                                                                                                                     │
     │ 1. Phase 1: Create event logger module (no dependencies)                                                                                                            │
     │ 2. Phase 5: Add sample counter integration (needed by logger)                                                                                                       │
     │ 3. Phase 2: Add triple-tap detection (uses logger)                                                                                                                  │
     │ 4. Phase 3: Update button handlers (uses logger + triple-tap)                                                                                                       │
     │ 5. Phase 4: Enable always-on SD recording (independent)                                                                                                             │
     │ 6. Phase 6: Remove BLE services (cleanup, safest last)                                                                                                              │
     │ 7. Phase 7: Build system updates                                                                                                                                    │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Risk Mitigation                                                                                                                                                     │
     │                                                                                                                                                                     │
     │ SD Card Performance: Always-on recording may overload SD card                                                                                                       │
     │ - Mitigation: Keep event log file handle open, buffer writes, test 1+ hour recording                                                                                │
     │                                                                                                                                                                     │
     │ Button Timing Conflicts: Triple-tap may interfere with double-tap                                                                                                   │
     │ - Mitigation: Extensive logging, test edge cases, make timing configurable                                                                                          │
     │                                                                                                                                                                     │
     │ File System Corruption: Simultaneous audio + event writes                                                                                                           │
     │ - Mitigation: Use write_sdcard_mutex for ALL SD operations, test power-off scenarios                                                                                │
     │                                                                                                                                                                     │
     │ Power-Off Event Loss: Device powers off before event written                                                                                                        │
     │ - Mitigation: Add 100ms delay after logging POWER_OFF, ensure flush completes                                                                                       │
     │                                                                                                                                                                     │
     │ Memory Usage: Event queue (100 events) uses RAM                                                                                                                     │
     │ - Mitigation: Monitor heap usage, reduce queue size if needed                                                                                                       │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ CSV Event Log Format                                                                                                                                                │
     │                                                                                                                                                                     │
     │ File: /SD:/events.csv                                                                                                                                               │
     │                                                                                                                                                                     │
     │ Format: timestamp,event_type                                                                                                                                        │
     │                                                                                                                                                                     │
     │ Example:                                                                                                                                                            │
     │ 0,0                                                                                                                                                                 │
     │ 80000,2                                                                                                                                                             │
     │ 160000,3                                                                                                                                                            │
     │ 240000,4                                                                                                                                                            │
     │ 320000,1                                                                                                                                                            │
     │                                                                                                                                                                     │
     │ Event Type Codes:                                                                                                                                                   │
     │ - 0 = POWER_ON                                                                                                                                                      │
     │ - 1 = POWER_OFF                                                                                                                                                     │
     │ - 2 = MARKER1 (double tap)                                                                                                                                          │
     │ - 3 = MARKER2 (triple tap)                                                                                                                                          │
     │ - 4 = LED_TOGGLE (long press)                                                                                                                                       │
     │                                                                                                                                                                     │
     │ Timestamp: Audio samples at 16kHz                                                                                                                                   │
     │ - Convert to time: time_ms = (timestamp * 1000) / 16000                                                                                                             │
     │ - Example: 80000 samples = 5000ms = 5 seconds                                                                                                                       │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Rollback Plan                                                                                                                                                       │
     │                                                                                                                                                                     │
     │ Each phase is independently committable:                                                                                                                            │
     │ 1. Comment out event logger calls to disable logging                                                                                                                │
     │ 2. Remove triple-tap case to revert to double-tap-only                                                                                                              │
     │ 3. Restore BLE services by uncommenting code                                                                                                                        │
     │ 4. Change always-on SD logic back to conditional                                                                                                                    │
     │                                                                                                                                                                     │
     │ Git Strategy: One commit per phase for easy rollback                                                                                                                │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Notes                                                                                                                                                               │
     │                                                                                                                                                                     │
     │ - No RTC hardware detected → using audio sample counter for timestamps                                                                                              │
     │ - Battery life impact: Always-on SD writing will increase power consumption (measure and document)                                                                  │
     │ - Event log file grows indefinitely (consider rotation in future)                                                                                                   │
     │ - Triple-tap window (900ms) is configurable via TRIPLE_TAP_WINDOW                                                                                                   │
     │ - All SD operations protected by existing write_sdcard_mutex                                                                                                        │
     │                                                                                                                                                                     │
     │ ---                                                                                                                                                                 │
     │ Success Criteria                                                                                                                                                    │
     │                                                                                                                                                                     │
     │ ✅ Device records audio continuously to SD card regardless of BLE state                                                                                             │
     │ ✅ Triple-tap detection works reliably (3 taps within 900ms)                                                                                                        │
     │ ✅ Button events logged to CSV with audio sample timestamps                                                                                                         │
     │ ✅ Long press toggles LEDs on/off                                                                                                                                   │
     │ ✅ No audio BLE services visible in BLE scanner                                                                                                                     │
     │ ✅ No data loss or file corruption during 1-hour test                                                                                                               │
     │ ✅ CSV log can be used to find button press timestamps in audio files 