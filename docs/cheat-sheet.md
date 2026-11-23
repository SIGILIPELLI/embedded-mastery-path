---
title: "Embedded Systems Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Embedded Systems cheat sheet: 40 quick-reference tables from every level of the Embedded Systems Mastery Path, each linked to its full lesson."
---

# Embedded Systems Cheat Sheet

Quick-reference tables from **40 lessons** of the free Embedded Systems Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · Setup &amp; Toolchain](level-1/01-setup-toolchain.md)

| Concept | Meaning |
|---------|---------|
| Sketch | An Arduino program (`.ino` file) |
| `setup()` | Runs once at power-on/reset — do initialization here |
| `loop()` | Runs repeatedly forever after `setup()` |
| `delay(ms)` | Pause execution for `ms` milliseconds (blocking) |
| `LED_BUILTIN` | Pin number of the onboard LED (13 on Uno, 2 on most ESP32) |
| Verify / Upload | Compile only / compile + flash to the board |
| Board package ("core") | Adds compiler + definitions for a chip family (e.g. ESP32) |
| Flash vs RAM | Program storage vs working memory — both small and fixed |
| [Wokwi](https://wokwi.com/) | Free browser simulator — runs all course sketches, no hardware |

### [02 · C/C++ for Embedded](level-1/02-c-cpp-for-embedded.md)

| Concept | Guidance |
|---------|----------|
| `int` size | 2 bytes on Uno, 4 on ESP32 — don't assume |
| `uint8_t`/`int16_t`/`uint32_t` | Fixed-width types — prefer for anything size-sensitive |
| Overflow | Silent wraparound — pick types with headroom (`uint32_t` for `millis()`) |
| `const uint8_t X = 5;` | Preferred constant — typed, scoped, zero RAM cost |
| `#define X 5` | Text substitution — common in libraries, avoid in your code |
| `String` | Heap allocations → fragmentation → long-run crashes; avoid in unattended code |
| `char buf[N]` + `snprintf` | Fixed-size, allocation-free text formatting |
| `F("literal")` | Keeps string literals out of SRAM on AVR; harmless on ESP32 |
| Flash vs SRAM | Code lives in flash, variables in SRAM — both fixed budgets |

### [03 · Digital I/O](level-1/03-digital-io.md)

| Function / idiom | Purpose |
|------------------|---------|
| `pinMode(pin, OUTPUT)` | Configure pin to drive HIGH/LOW |
| `pinMode(pin, INPUT_PULLUP)` | Input with internal pull-up — idle HIGH, pressed LOW |
| `digitalWrite(pin, HIGH/LOW)` | Set output level |
| `digitalRead(pin)` | Read input level (returns `HIGH` or `LOW`) |
| LED wiring | pin → 220 Ω resistor → LED anode; cathode → GND |
| Button wiring (pull-up) | pin → button → GND, `INPUT_PULLUP`, pressed = `LOW` |
| Floating input | Unconnected input pin reads noise — always define idle state |
| Bounce | Mechanical contacts chatter for ~1–10 ms; debounce with a ~30 ms stability window |
| Edge detection | Compare current vs previous state to act once per press |

### [04 · Analog I/O &amp; PWM](level-1/04-analog-io-pwm.md)

| Concept | Uno | ESP32 |
|---------|-----|-------|
| `analogRead` range | 0–1023 (10-bit) | 0–4095 (12-bit) |
| Full-scale voltage | 5 V | 3.3 V |
| ADC pins | A0–A5 | GPIO 32–39 safest (ADC2 dies with WiFi on) |
| PWM call | `analogWrite(pin, 0..255)` | `ledcAttach(pin, freq, res)` + `ledcWrite(pin, duty)` |
| PWM pins | Only `~` pins (3,5,6,9,10,11) | Any output pin, 16 channels |
| PWM frequency | Fixed (~490/980 Hz) | Configurable (5 kHz LEDs, ~20 kHz motors) |
| Rescaling | `map(x, inLo, inHi, outLo, outHi)` | same |
| Duty cycle | Fraction of time HIGH — sets average power | same |

### [05 · Serial/UART Communication](level-1/05-serial-uart.md)

| Function | Purpose |
|----------|---------|
| `Serial.begin(115200)` | Start serial; baud must match the monitor |
| `Serial.print(x)` / `println(x)` | Write without / with trailing newline |
| `Serial.print(f, 2)` | Float with 2 decimal places |
| `Serial.println(x, HEX)` | Print in hex (also `BIN`, `OCT`) |
| `Serial.available()` | Number of received bytes waiting in the buffer |
| `Serial.read()` | Consume one byte (returns `-1` if none) |
| `Serial.readStringUntil('\n')` | Read a full line (set monitor line ending to Newline) |
| `str.trim()` / `startsWith()` / `substring()` / `toInt()` | Basic parsing toolkit |
| `millis()` in logs | Timestamp events, measure durations |

### [06 · Sensors, I2C &amp; SPI](level-1/06-sensors-i2c-spi.md)

| Concept | Detail |
|---------|--------|
| I2C wires | SDA + SCL, shared by all devices; pull-ups required |
| I2C addressing | 7-bit address per device (OLED: `0x3C`); scanner finds them |
| I2C pins | Uno: A4/A5 — ESP32: GPIO 21 (SDA) / 22 (SCL) |
| SPI wires | SCK, MOSI, MISO + one CS per device; much faster than I2C |
| Choosing a bus | Sensors → I2C; SD cards, fast displays → SPI |
| DHT22 | Single-wire protocol, one read per 2 s, returns floats, check `isnan()` |
| SSD1306 pattern | draw to buffer → `display.display()` pushes it to the panel |
| Library Manager | IDE: Sketch menu — Wokwi: Library Manager tab |

### [07 · Timers &amp; Interrupts](level-1/07-timers-interrupts.md)

| Concept | Detail |
|---------|--------|
| `millis()` | `uint32_t` ms since boot; wraps after ~49.7 days |
| Schedule pattern | `if (now - last >= interval) { last = now; ...task... }` |
| Wraparound safety | Always compare with subtraction, never absolute times |
| Cooperative rule | No task in `loop()` may block |
| `attachInterrupt(digitalPinToInterrupt(pin), isr, mode)` | Run `isr` on pin event |
| Modes | `RISING`, `FALLING`, `CHANGE`, `LOW` |
| Interrupt pins | Uno: only 2, 3 — ESP32: any GPIO |
| ISR rules | Short; no Serial/delay/I2C; shared vars `volatile` |
| `IRAM_ATTR` | ESP32: keeps the ISR in RAM — required |
| Critical section | `noInterrupts()` / `interrupts()` around multi-byte shared reads |

### [08 · Memory &amp; Power Basics](level-1/08-memory-power.md)

| Concept | Detail |
|---------|--------|
| Flash / SRAM / EEPROM-NVS | Program (persistent) / variables (volatile, scarce) / runtime settings (persistent) |
| Out of SRAM | No error — stack corrupts data → random resets |
| `ESP.getFreeHeap()` / `getMinFreeHeap()` | Current / worst-case free RAM; falling min = leak |
| `EEPROM.get/put(addr, var)` | AVR persistent storage |
| `Preferences` (NVS) | ESP32 key-value persistent store — preferred |
| Flash wear | ~10k–100k erase cycles — write on change, not per loop |
| Deep sleep current | ~10 µA vs 30–250 mA active — the battery-life lever |
| `esp_sleep_enable_timer_wakeup(µs)` + `esp_deep_sleep_start()` | Timed sleep; wake = reboot into `setup()` |
| `RTC_DATA_ATTR` | Variable survives deep sleep (not power loss) |
| `Serial.flush()` before sleep | Otherwise the last log lines are cut off |

### [09 · ESP32 WiFi Intro](level-1/09-esp32-wifi.md)

| Concept | Detail |
|---------|--------|
| `WiFi.mode(WIFI_STA)` + `WiFi.begin(ssid, pass)` | Join a network as a client |
| `WiFi.status() == WL_CONNECTED` | Poll until (and monitor after) connection |
| `WiFi.localIP()` / `WiFi.RSSI()` | Board's IP / signal strength (dBm) |
| Wokwi network | SSID `Wokwi-GUEST`, empty password, has internet |
| 2.4 GHz only | ESP32 cannot see 5 GHz networks |
| `HTTPClient` GET | `begin(url)` → `GET()` → check code → `getString()` → `end()` |
| Status codes | 200 OK; negative = connection failed |
| `WebServer server(80)` | HTTP server on the board |
| `server.on(path, handler)` + `server.begin()` | Routing setup |
| `server.handleClient()` in `loop()` | Actually serves requests — keep loop non-blocking |

### [10 · Capstone — Environment Monitor](level-1/10-capstone-project.md)

| Pattern | Where it came from |
|---------|--------------------|
| `setup()` init, task-based `loop()` | Modules 1, 7 |
| Fixed-width types, `char[]` + `snprintf`, `F()` | Module 2 |
| LED + resistor on GPIO, `digitalWrite` | Module 3 |
| I2C device at address (OLED `0x3C`) | Module 6 |
| Check `isnan()` on every sensor read | Module 6 |
| `if (now - last >= period)` scheduling | Module 7 |
| OK/ERR command console with validation | Module 5 |
| Struct as single source of truth | This module |
| Fault signalling (blink = sensor dead) | This module |
| WiFi/web as an add-on task | Module 9 |

## Level 2 · Intermediate

### [FreeRTOS Tasks &amp; Queues](level-2/01-freertos-tasks-queues.md)

| Concept | Detail |
|---------|--------|
| `xTaskCreatePinnedToCore(fn, name, stack, arg, prio, &handle, core)` | Create + pin a task; returns `pdPASS` |
| Stack size units | **Bytes** on ESP-IDF (vanilla FreeRTOS uses words — a real gotcha) |
| Priority | `0` (idle) .. `configMAX_PRIORITIES-1` (25); higher preempts lower |
| `loopTask` | Arduino's `loop()`, running as a task at priority 1, core 1 |
| `vTaskDelay(pdMS_TO_TICKS(ms))` | Blocks *this* task only, yields CPU to others |
| Priority inversion | Low-prio task holds a lock a high-prio task needs; a mid-prio task starves both |
| Fix: real mutex | `xSemaphoreCreateMutex()` — has priority inheritance; binary semaphores don't |
| `xQueueCreate(len, itemSize)` | Fixed-size FIFO; copies data, not pointers, between tasks |
| `xQueueSend`/`xQueueReceive(q, &item, ticks)` | `0`=don't block, `portMAX_DELAY`=block forever |
| `xSemaphoreTake`/`Give(mutex, ticks)` | Guard any resource shared across tasks |
| `uxTaskGetStackHighWaterMark(NULL)` | Worst-case-ever free stack (words) for the calling task |

### [ESP-IDF Development](level-2/02-esp-idf.md)

| Concept | Detail |
|---------|--------|
| Entry point | `app_main(void)` in `main/main.c` — no `setup()`/`loop()` |
| Logging | `ESP_LOGE/W/I/D/V(TAG, "fmt", ...)` — leveled, tagged |
| `idf.py set-target esp32` | Once per project, before first build |
| `idf.py build` / `flash` / `monitor` | Compile / upload / watch serial |
| `idf.py menuconfig` | Interactive config → written to `sdkconfig` |
| `sdkconfig` | Commit it — it's build configuration, not a cache |
| Component | Folder + `CMakeLists.txt` calling `idf_component_register()` |
| `idf_component.yml` | Declares dependencies from the ESP Component Registry |
| `idf.py size` | Flash/RAM usage breakdown per component |

### [Bluetooth Low Energy (BLE)](level-2/03-ble.md)

| Concept | Detail |
|---------|--------|
| GAP | Discovery/connection layer — roles: peripheral (advertises) / central (scans) |
| GATT | Data layer — services contain characteristics; characteristics have properties |
| Properties | `READ`, `WRITE`, `NOTIFY`, `INDICATE` — what a central may do |
| CCCD (`0x2902`) | Descriptor a central writes to subscribe to notify/indicate |
| `BLEDevice::init(name)` | Sets the advertised device name, starts the BLE stack |
| `createServer()` → `createService()` → `createCharacteristic()` | GATT tree, built top-down |
| `BLE2902` descriptor | Required on any `NOTIFY`/`INDICATE` characteristic |
| `pCharacteristic->notify()` | Push value to subscribed centrals only — no-op if none |
| `onDisconnect()` | Must call `pServer->getAdvertising()->start()` to be reachable again |
| Default MTU | 23 bytes (~20 usable) unless negotiated up |

### [MQTT &amp; IoT Messaging](level-2/04-mqtt-iot-messaging.md)

| Concept | Detail |
|---------|--------|
| Model | Publish/subscribe via a broker; one persistent outbound TCP connection |
| Topic | `site/device/measurement` — hierarchical, implicit, no leading `/` |
| `+` / `#` | One level / all remaining levels (must be last) — subscribe side only |
| QoS 0 / 1 / 2 | At most once / at least once (**dupes**) / exactly once |
| Retain | Broker keeps the last message per topic and replays it — for *state* |
| LWT | Broker publishes it on an unclean drop — with retain, gives presence |
| `.broker.address.uri` | `mqtt://host:1883` or `mqtts://host:8883` |
| `esp_mqtt_client_init()` / `_start()` | Non-blocking; a background task owns the connection |
| `esp_mqtt_client_register_event(c, ESP_EVENT_ANY_ID, h, arg)` | One handler, switch on `event_id` |
| `event->topic_len` / `data_len` | Payload is **not** null-terminated — always use lengths |
| `esp_mqtt_client_enqueue()` | Non-blocking publish — the safe call from inside a handler |
| Reconnection | Automatic — don't write your own retry loop |

### [OTA Firmware Updates](level-2/05-ota-updates.md)

| Concept | Detail |
|---------|--------|
| Why two slots | Can't overwrite the flash you're executing from — alternate `ota_0`/`ota_1` |
| `otadata` | 0x2000 data partition holding the boot selection and per-slot state |
| Partition table | Custom `partitions.csv` + menuconfig; `app` partitions align to 64 KB |
| Cost | Roughly half your flash budget goes to the spare slot |
| `esp_https_ota(&cfg)` | One blocking call: download → write → verify. Needs its own task |
| `esp_ota_get_next_update_partition(NULL)` | The slot you're *not* running from |
| `esp_ota_begin/write/end` | Manual path; `end()` validates before you commit |
| `esp_ota_set_boot_partition()` | Point of no return — only after `esp_ota_end() == ESP_OK` |
| `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE` | New image boots as `ESP_OTA_IMG_PENDING_VERIFY` |
| `esp_ota_mark_app_valid_cancel_rollback()` | "Confirmed good" — call it *late*, after a real self-test |
| `esp_ota_mark_app_invalid_rollback_and_reboot()` | Give up now and return to the old slot |
| Crash during probation | Bootloader falls back to the previous slot automatically |
| `esp_app_get_description()` | Version, build date, IDF version from the image header |

### [Advanced Peripherals (I2S, RMT, CAN)](level-2/06-advanced-peripherals.md)

| Concept | Detail |
|---------|--------|
| I2S | Continuous synchronous audio: BCLK + WS (left/right) + data, DMA-backed |
| `i2s_new_channel(&cfg, &tx, &rx)` | Pass `NULL` for the direction you don't need |
| `I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(bits, mode)` | Standard format for most mics/DACs |
| `i2s_channel_read(h, buf, len, &got, timeout)` | Blocks until a DMA buffer is full |
| 24-bit mic in 32-bit slot | Shift the padding out (`>> 14`) or the numbers look absurd |
| RMT | Hardware (level, duration) pulse generator — IR, WS2812, any tight timing |
| `resolution_hz` | Sets the tick; 10 MHz → 1 tick = 0.1 µs |
| `rmt_symbol_word_t` | `{level0, duration0, level1, duration1}` — one bit's waveform |
| `rmt_transmit()` | Queues and returns; hardware clocks it out, immune to preemption |
| TWAI | ESP32's CAN 2.0 controller — needs an external transceiver + 120 Ω ends |
| Message ID | Describes content, not destination; **lower ID wins arbitration** |
| `TWAI_GENERAL_CONFIG_DEFAULT(tx, rx, mode)` | Plus timing + filter configs into `twai_driver_install()` |
| Bus-off | 256 TX errors (often: no second node to ACK) → controller stops transmitting |
| `TWAI_MODE_NO_ACK` | Self-test mode for a solo node on the bench |

### [Filesystems &amp; Storage (LittleFS, SD)](level-2/07-filesystems-storage.md)

| Concept | Detail |
|---------|--------|
| NVS | Key/value settings; `nvs_open` → `set` → **`nvs_commit`** → `nvs_close` |
| NVS limits | Key and namespace ≤ 15 chars; `ESP_ERR_NVS_NOT_FOUND` on first boot is normal |
| `nvs_flash_init()` | Handle `NO_FREE_PAGES`/`NEW_VERSION_FOUND` by erasing; must run before WiFi |
| LittleFS | Power-fail safe, wear-levelled files in internal flash |
| `esp_vfs_littlefs_register(&conf)` | Mounts at `base_path`; then use plain `fopen`/`fprintf` |
| `esp_littlefs_info(label, &total, &used)` | Space check — act before the partition fills |
| Partition | Custom `partitions.csv` entry, subtype `littlefs` (or `spiffs`) |
| SPIFFS | Legacy, no power-fail guarantee, slows badly when full — prefer LittleFS |
| Flash erase unit | 4 KB sector, ~100k cycles — the write cost is the *erase* |
| Wear rule | **Append, don't rewrite in place**; batch small writes; keep 10–20% free |
| FAT/SD | `esp_vfs_fat_sdspi_mount()` → `/sdcard/...`; unmount before removal |
| `format_if_mount_failed` on SD | Leave `false` — `true` silently wipes a user's card |
| Durability | `fclose()` (or `fflush` + `fsync`) is where the data actually lands |

### [Watchdogs &amp; Robust Firmware](level-2/08-watchdogs-robust-firmware.md)

| Concept | Detail |
|---------|--------|
| Task WDT | Resets the chip if a subscribed task stops checking in |
| `esp_task_wdt_add(NULL)` / `esp_task_wdt_reset()` | Subscribe this task / report in from it |
| `esp_task_wdt_config_t` | `.timeout_ms`, `.idle_core_mask`, `.trigger_panic` |
| `idle_core_mask` | Watch the idle tasks — catches any task hogging a core |
| Timeout sizing | Several × the loop period; too tight and the WDT *is* the bug |
| Anti-pattern | Resetting the WDT from a timer/keepalive — it then proves nothing |
| Interrupt WDT | On by default; firing means an ISR or critical section ran too long |
| `esp_task_wdt_delete(NULL)` | Unsubscribe around a known-long blocking call, deliberately |
| `esp_reset_reason()` | `ESP_RST_POWERON` / `PANIC` / `TASK_WDT` / `INT_WDT` / `BROWNOUT` / ... |
| Crash counter in NVS | Cheapest field telemetry there is — publish it |
| Brownout detector | On by default; `ESP_RST_BROWNOUT` means fix the *hardware* |
| Retry policy | Bounded attempts, exponential backoff, capped, with a give-up branch |
| Heap trend | `esp_get_minimum_free_heap_size()` falling over days = leak |
| `esp_restart()` | A valid recovery path — log first, rate-limit it |

### [Debugging &amp; Unit Testing](level-2/09-debugging-unit-testing.md)

| Concept | Detail |
|---------|--------|
| `LoadProhibited` / `StoreProhibited` | Bad pointer; `EXCVADDR` holds the address, `0x0` = null |
| `IllegalInstruction` | Often a task function that returned without `vTaskDelete(NULL)` |
| Backtrace | `PC:SP` pairs, innermost first; `idf.py monitor` symbolicates them |
| `addr2line -pfiaC -e build/app.elf <pc>` | Decode a backtrace captured elsewhere |
| `esp_backtrace_print(depth)` | Print the current stack without crashing |
| `CONFIG_ESP_SYSTEM_PANIC` | Print+reboot / halt / **GDBStub** (live GDB over plain serial) |
| `CONFIG_LOG_MAXIMUM_LEVEL` | Compile-time ceiling — above it, the code isn't in the binary |
| `esp_log_level_set(tag, level)` | Runtime, per tag; `"*"` for all tags |
| `ESP_LOG_BUFFER_HEXDUMP(tag, buf, len, level)` | Hex + ASCII dump for protocol work |
| Logging trap | Blocks for milliseconds; never from an ISR — toggle a GPIO instead |
| Core dump | `coredump` data partition + `CONFIG_ESP_COREDUMP_ENABLE_TO_FLASH` |
| `idf.py coredump-info` / `coredump-debug` | Summary / full GDB session on a stored crash |
| `esp_core_dump_image_get()` / `_erase()` | Detect a dump at boot; clear it or lose the next one |
| `idf.py openocd gdb` | Live JTAG; ESP32 needs ESP-Prog, C3/S3 have USB-JTAG built in |
| ESP32 JTAG pins | GPIO12–15; GPIO12 (MTDI) also straps the flash supply voltage at reset |
| Leak hunting | `esp_get_minimum_free_heap_size()` trend, then `HEAP_TRACE_LEAKS` |
| `heap_trace_init_standalone(recs, n)` → `start` → `stop` → `dump` | Names the caller of every unfreed block |
| `CONFIG_HEAP_CORRUPTION_DETECTION` | Comprehensive: `0xCE` fresh, `0xFE` freed, canaries around blocks |
| `heap_caps_check_integrity_all(true)` | Bisect to where the overrun happens |
| `uxTaskGetStackHighWaterMark()` | On ESP-IDF this is **bytes** (vanilla FreeRTOS docs say words) |
| Unity test | `TEST_CASE("name", "[tag]")` in `components/<c>/test/test_*.c` |
| Test CMake | `idf_component_register(SRC_DIRS "." INCLUDE_DIRS "." REQUIRES unity <comp>)` |
| Running tests | Flash, press Enter for the menu; `*` runs all; `pytest-embedded` for CI |

### [Project — WiFi Data Logger](level-2/10-project-wifi-data-logger.md)

| Piece | Choice made here | Why |
|-------|------------------|-----|
| Topology | 3 tasks, 2 queues, samples copied by value | No shared mutable state between tasks |
| Priorities | sampler 6 > logger 5 > uplink 4 > ota 3 | Timing-critical first; network last |
| `sample_q` depth 60 | Deep, 50 ms send timeout | A slow flash write must not cost a reading |
| `uplink_q` depth 30 | Shallow, **0 ms** send timeout | A dead network must not stall the logger |
| `vTaskDelayUntil()` | Fixed period, drift-free | `vTaskDelay()` adds the work time to every cycle |
| Batching 30 samples | One `fopen`/`fclose` per batch | Erase cost is per 4 KB sector, not per byte |
| Rotation at 64 KB | `rename()` to `log.1.csv` | Bounded storage regardless of uptime |
| MQTT publish | `esp_mqtt_client_enqueue()`, QoS 1, retain | Non-blocking; dashboards get state on connect |
| Timestamps | SNTP, stored as UTC epoch seconds | Local time in a log breaks twice a year |
| Task WDT | sampler + logger subscribed; uplink not | Only tasks with a duty cycle should be watched |
| OTA trigger | MQTT command → dedicated 8 KB task | Never do the work in the event handler |
| Rollback gate | `self_test_ok()` = IP + broker + filesystem | Proves the device can receive the *next* fix |
| Mutex type | `xSemaphoreCreateMutex()` for the filesystem | Priority inheritance; binary semaphores have none |
| Crash reporting | Reset reason + core dump published at boot | Field telemetry you can't get any other way |

## Level 3 · Advanced

### [Bare-Metal ARM Cortex-M (STM32)](level-3/01-bare-metal-arm-stm32.md)

| Concept | Detail |
|---|---|
| Memory-mapped register | `*(volatile T *)ADDRESS` — `volatile` prevents caching/reordering |
| Enable order | Clock enable (`RCC_*ENR`) before touching any peripheral register |
| Reset entry | CPU loads SP from `0x0`, PC from `0x4` — no bootloader code runs first |
| Weak alias | Unimplemented vectors fall back to `Default_Handler` instead of link errors |
| `BSRR` vs `ODR |=` | `BSRR` is a true atomic set/clear; `ODR` read-modify-write is not |
| Toolchain | `arm-none-eabi-gcc` + OpenOCD/ST-Link, not Arduino/ESP-IDF build system |
| Verification here | Portable bit-logic compiled/run with `gcc`; addresses reviewed against ARM/ST docs only |

### [Registers, Linker Scripts &amp; Startup Code](level-3/02-registers-linker-startup.md)

| Concept | Detail |
|---|---|
| `.text`/`.rodata` | Code and constants — stay in FLASH, never copied |
| `.data` | Non-zero globals — live in RAM, load image in FLASH, copied by startup code |
| `.bss` | Zero-initialized globals — live in RAM only, zeroed by startup code, occupy no flash |
| `AT> FLASH` | Tells linker the *load* address differs from the *run* address |
| `_sdata`/`_edata`/`_sbss`/`_ebss` | Linker-defined symbols marking section boundaries, used by startup code |
| Stack direction | Grows **down** from `_estack` (top of RAM) |
| `KEEP()` | Prevents the linker from discarding a section nothing references directly (e.g. `.isr_vector`) |
| Verification here | Copy/zero logic simulated and run with `gcc`; real flash/RAM layout not testable off-chip |

### [RTOS Internals](level-3/03-rtos-internals.md)

| Concept | Detail |
|---|---|
| TCB | Task Control Block — saved SP + priority + list linkage, not the stack itself |
| Initial fake context | Stack pre-filled so the first switch "resumes" the task at its entry point |
| `PendSV` | Lowest-priority exception, dedicated to context switches, never preempts real IRQs |
| Auto-saved by hardware | R0-R3, R12, LR, PC, xPSR — pushed/popped by exception entry/return |
| Manually saved in handler | R4-R11 — the context-switch code's own job |
| Ready bitmap | One bit per priority; highest set bit = next task to run (O(1) via CLZ) |
| Same-priority tasks | Time-slice via `SysTick`; only strictly higher priority guarantees preemption |
| Verification here | Bitmap scheduling logic compiled/run with `gcc`; PendSV assembly reviewed, not executed |

### [DMA &amp; High-Speed Peripherals](level-3/04-dma-high-speed-peripherals.md)

| Concept | Detail |
|---|---|
| DMA controller | Independent hardware that moves data peripheral<->memory without CPU load/store per byte |
| Circular mode | DMA re-arms automatically at the end of the buffer — true continuous capture |
| D-Cache coherency | Only relevant on cached cores (e.g. M7); invalidate before CPU read, clean before DMA read |
| `SCB_InvalidateDCache_by_Addr` | CMSIS call to drop stale cached copies before reading DMA'd data |
| Double buffering | Two buffers, DMA fills one while code processes the other — avoids torn reads |
| `volatile` on shared flags | Required wherever an ISR sets a flag `main_loop` polls |
| Verification here | Buffer-swap bookkeeping compiled/run with `gcc`; cache/DMA hardware behavior reviewed, not executed |

### [Low-Power Design Deep Dive](level-3/05-low-power-design.md)

| Concept | Detail |
|---|---|
| `WFI` | Halts CPU until any enabled interrupt is pending — the core sleep primitive |
| `SLEEPDEEP` (SCB_SCR) | Selects how deep `WFI` goes: Sleep vs Stop/Standby |
| Sleep | CPU clock off, peripherals/RAM retained, µs wake |
| Stop | Most clocks off, RAM retained, µs-ms wake, reconfigure clocks on resume |
| Standby | RAM lost, behaves like reset on wake, must persist state externally |
| Any pending IRQ wakes | Disable/mask anything not meant to wake the device (e.g. `SysTick`) before deep sleep |
| Wake source setup | Must be enabled in **both** the power controller and the NVIC/EXTI |
| Verification here | Battery-life arithmetic compiled/run with `gcc`; actual sleep-mode current is hardware-measured, not testable here |

### [Interrupts &amp; the NVIC](level-3/06-interrupts-nvic.md)

| Concept | Detail |
|---|---|
| NVIC | Cortex-M's interrupt controller — enable, priority, and pending state per IRQ |
| Priority numbers | **Lower number = higher urgency**; can preempt any strictly higher-numbered priority |
| Equal priority | Never preempts — queues and runs after the current handler returns |
| `EXTI_PR` / pending bit | Must be cleared in the handler, with care about ordering vs. reading data |
| `volatile` | Prevents compiler caching/reordering; does **not** by itself guarantee atomicity |
| `__disable_irq()`/`__enable_irq()` | Critical section for accesses that must be atomic w.r.t. an ISR |
| Verification here | Preemption-rule logic compiled/run with `gcc`; real NVIC register behavior reviewed against ARM docs only |

### [PCB Design Basics (KiCad)](level-3/07-pcb-design-kicad.md)

| Concept | Detail |
|---|---|
| ERC | Checks the **schematic**: unconnected pins, conflicting drivers, missing power symbols |
| DRC | Checks the **physical board**: clearance, drill sizes, fab capability limits |
| Decoupling cap | ~100 nF ceramic per IC power pin, placed as close as physically possible |
| Trace width for current | Use an IPC-2152 calculator; ~0.25 mm/A (1oz Cu, external) is only a rough starting estimate |
| Ground plane | Gives every signal a short return path; avoid routing across plane splits |
| Gerber/drill files | The actual manufacturing output sent to a fab house |
| Reality check | This module is manual technical review — no board from this course was fabricated or tested |

### [Protocol Deep Dive (UART/I2C/SPI/CAN)](level-3/08-protocol-deep-dive.md)

| Concept | Detail |
|---|---|
| UART framing | Start(0), 8 data bits LSB-first, optional parity, stop(1) — no shared clock |
| I2C clock stretching | Slave holds SCL low to pause the master — legitimate, must be handled |
| I2C arbitration | Multi-master: whoever tries to send `1` while another sends `0` backs off |
| SPI CPOL/CPHA | Four modes; mismatch produces plausible garbage, not an obvious failure |
| CAN arbitration | Dominant bit (`0`) always wins; lower numeric ID = higher priority, by design |
| Verification here | UART/CAN logic compiled/run with `gcc`; real bus electrical behavior reviewed against spec only |

### [Bootloaders &amp; Firmware Updates](level-3/09-bootloaders.md)

| Concept | Detail |
|---|---|
| Two-stage boot | Fixed bootloader at reset address, jumps to application elsewhere in flash |
| `SCB_VTOR` | Must be set to the application's vector table address after the jump |
| Dual-bank update | Write+verify the new image in the inactive bank; flip active bank only after |
| Never erase-before-verify | The rule that prevents power-loss bricking |
| Boot counter / rollback | Falls back to the previous bank after N failed boots of a new image |
| CRC vs signature | CRC catches corruption; only cryptographic signing catches tampering (module 4-02) |
| Verification here | Header/CRC validation logic compiled/run with `gcc`; real flash erase/write needs actual hardware |

### [Project — Custom Sensor Board Firmware](level-3/10-project-sensor-board.md)

| Module combined | Role in this project |
|---|---|
| 3-01/3-02 | Register access, linker/startup for the whole firmware image |
| 3-03 | Not directly used — bare-metal main loop chosen over an RTOS for this design's simplicity |
| 3-04 | DMA-driven I2C sampling, freeing the CPU to sleep between reads |
| 3-05 | Stop mode between samples; wake source must be unmasked in the NVIC |
| 3-06 | `volatile` + ISR-safe ring buffer indices shared between DMA ISR and main loop |
| 3-07 | Decoupling placement, ground plane, test points on the physical board |
| 3-08 | UART framing for the debug/report protocol |
| 3-09 | Vector table relocation if deployed behind a bootloader for field updates |

## Level 4 · Master

### [Production Firmware Architecture](level-4/01-production-firmware-architecture.md)

| Concept | Detail |
|---|---|
| HAL | Bottom layer — knows chip registers, exposes a portable function-pointer interface |
| Driver | Knows a specific peripheral's protocol, depends only on the HAL interface |
| Application | Business logic — no register access, no board `#ifdef` |
| Feature flags | `HAS_X` compile-time flags, not board-name checks, in application code |
| Fake HAL | Swap in a canned-response HAL to unit-test drivers on a host machine, no hardware needed |
| Consistent error enum | Decide once, project-wide — expensive to retrofit later |
| Verification here | Driver-against-fake-HAL logic compiled/run with `gcc`; real hardware timing not represented |

### [Secure Boot &amp; Encrypted OTA at Scale](level-4/02-secure-boot-ota-scale.md)

| Concept | Detail |
|---|---|
| CRC | Detects accidental corruption only — trivially recomputable by an attacker |
| Signature (Ed25519/ECDSA) | Proves the image came from whoever holds the private key |
| Public key placement | Must be immutable (OTP fuses/ROM) — a rewritable key defeats the whole scheme |
| Secure boot root of trust | First-stage verification anchored in hardware, not just software checking software |
| Encryption | Confidentiality, separate from and additional to signature-based authenticity |
| Downgrade protection | Version monotonicity check prevents re-installing an old, vulnerable, validly signed image |
| Canary/staged rollout | Procedural mitigation for a legitimately signed but regressive build |
| Verification here | Verify-then-trust decision logic compiled/run with `gcc`; real crypto primitives reviewed, not implemented/executed here |

### [TinyML &amp; Edge AI](level-4/03-tinyml-edge-ai.md)

| Concept | Detail |
|---|---|
| Quantization | Map float weights/activations to int8 via `scale`/`zero_point`, computed offline |
| Why it matters | Often the difference between a model fitting in MCU flash/RAM and not fitting at all |
| Accumulator width | int8 x int8 products summed in int32 to avoid overflow |
| Requantization | Each layer's int32 accumulator is rescaled back to int8 before the next layer |
| No FPU needed | int8 MAC operations run on cores without a floating point unit |
| RAM sizing | Driven by the largest single layer's activation buffer, not total model size |
| Verification here | Quantize/dequantize and one dense layer compiled/run with `gcc`; no real trained model or NN framework used |

### [RF &amp; Wireless Protocol Design](level-4/04-rf-protocol-design.md)

| Concept | Detail |
|---|---|
| Link budget | TX power + antenna gains - path loss - cable loss, compared against RX sensitivity + margin |
| Path loss (free space) | `20*log10(4*pi*d/wavelength)` — a best case, real environments lose more |
| Processing gain | Lower data rate/spreading factor trades throughput for range/noise immunity |
| Sequence number | Detects lost/duplicate packets — expected on RF links, not just a nice-to-have |
| Duty cycle | Legally regulated in many ISM bands — a protocol design constraint, not just politeness |
| Verification here | Link-budget arithmetic and CRC/seq-num logic compiled/run with `gcc`; real RF/antenna/propagation behavior needs lab equipment |

### [HIL Testing &amp; CI for Firmware](level-4/05-hil-testing-ci.md)

| Concept | Detail |
|---|---|
| Testing pyramid | Mostly host-side unit tests (fake HAL), fewer integration/emulator tests, fewest HIL tests |
| HIL | Real target hardware, automated (flash, power, UART, GPIO) — catches timing/electrical bugs simulators can't |
| CI without hardware | Build, static analysis, host unit tests, flash/RAM size gate |
| Size gate | Cheap, automatic protection against silent flash/RAM budget creep |
| HIL scope | Timing margins, real power measurement, end-to-end — not a replacement for fast unit tests |
| Verification here | Size-budget policy logic compiled/run with `gcc`; HIL rig automation is architecture reviewed, not executed (no rig available) |

### [Functional Safety &amp; MISRA](level-4/06-functional-safety-misra.md)

| Concept | Detail |
|---|---|
| Fail-safe | On detected failure, go to a known-safe state — most common design goal |
| Fail-operational | Must keep working correctly through a failure — needs redundancy, reserved for critical systems |
| MISRA C:2012 | Restricts C to eliminate undefined/unspecified/misuse-prone constructs |
| Rule 15.5 | Single point of exit per function (advisory) |
| Rule 8.13 | `const`-correctness for read-only pointer parameters (advisory) |
| Rule 10.x | Essential-type conversion rules — governs signed/unsigned/width mismatches |
| Plausibility check | Validates a reading against recent history/rate limits, not just its static range |
| Verification here | Validation/plausibility logic compiled/run with `gcc`; MISRA rule text and safety-process claims reviewed against real, citable published material only |

### [Ultra-Low-Power &amp; Energy Harvesting](level-4/07-ultra-low-power.md)

| Concept | Detail |
|---|---|
| Power balance | Harvesting designs must satisfy avg(harvest) >= avg(consume), not a fixed total budget |
| Adaptive duty cycling | Sample stored energy, classify state, adjust sleep interval accordingly |
| `E = 0.5 * C * V^2` | Supercapacitor stored energy — voltage is the measurable proxy for energy level |
| Regulator quiescent current | Can dominate a microwatt-scale power budget — choose parts for this, not battery convenience |
| Hysteresis | Needed between power states to avoid oscillation near a threshold |
| Worst-case sizing | Storage capacity and critical-state behavior must survive worst case, not average, harvest conditions |
| Verification here | Power-balance and state-classification logic compiled/run with `gcc`; real harvester/regulator behavior needs lab measurement |

### [Manufacturing, DFM &amp; Factory Test](level-4/08-manufacturing-dfm-test.md)

| Concept | Detail |
|---|---|
| DFM | Layout choices for reliable machine assembly — placement, panelization, stencil design |
| Bed-of-nails / flying probe | Physical test-point contact for electrical test — needs accessible test points |
| Boundary-scan (JTAG) | Pin-level connectivity test through the chip's own scan chain, no physical probe needed |
| Factory test firmware | Separate image, exercises every peripheral once, never ships to customers |
| Run-all-tests policy | Log every result even after a failure — preserves diagnostic information |
| Serial-number traceability | Permanent record linking each unit to its exact factory test results |
| Verification here | Test-aggregation/logging logic compiled/run with `gcc`; real DFM/assembly-line behavior reviewed against manufacturing practice only |

### [Fleet Management &amp; Device Clouds](level-4/09-fleet-management.md)

| Concept | Detail |
|---|---|
| Device identity | Provisioned UUID + fleet-CA-signed certificate — never a MAC address alone |
| Telemetry aggregation | Report summaries (min/max/mean/count) on an interval, not every raw sample |
| Config validation | Validate before applying, same principle as firmware verify-before-install |
| Staged config rollout | Canary/staged pattern from module 4-02 applies to configuration, not just firmware |
| Cardinality limits | Fleet-scale telemetry backends need bounded label/tag cardinality to stay operational |
| Offline behavior | Devices must buffer and retry, not block, during connectivity loss |
| Verification here | Aggregation/validation logic compiled/run with `gcc`; real fleet backend/network behavior not represented |

### [Capstone — Production IoT Product](level-4/10-capstone-production-iot.md)

| Module | Role in the capstone |
|---|---|
| 4-01 | Layering: application depends only on driver/HAL interfaces |
| 4-02 | Bootloader signature verification + staged rollout for updates |
| 4-03 | Quantized on-device anomaly inference on sensor readings |
| 4-04 | (If wireless) link budget and packet design for the report path |
| 4-05 | HIL tier catches what this capstone's host-side tests structurally cannot |
| 4-06 | Plausibility/range validation before any reading is trusted or aggregated |
| 4-07 | Power-state classification driving adaptive duty cycling |
| 4-08 | DFM-reviewed layout and factory test firmware at manufacturing time |
| 4-09 | Telemetry aggregation and device identity for fleet reporting |

---

Go deeper with the full [Embedded Systems Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
