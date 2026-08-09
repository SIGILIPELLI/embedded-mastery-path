# ESP-IDF Development

Arduino got you productive fast by hiding the build system, the FreeRTOS
configuration, and the chip's boot process behind `setup()`/`loop()`.
**ESP-IDF** (Espressif IoT Development Framework) is what's underneath —
Espressif's own SDK, built on the same FreeRTOS you met last module, and the
toolchain every production ESP32 firmware is actually built with. This
module covers the project layout, the `idf.py` build tool, and `menuconfig`
— the three things you need before any of the rest of Level 2 makes sense,
since modules 3 onward lean on ESP-IDF components directly.

## Why professionals leave Arduino

The Arduino core is itself a *component* built on top of ESP-IDF — it's not
a different chip, just a friendlier, more restrictive layer. Dropping to
ESP-IDF trades some of that friendliness for:

- **Full API surface.** Every driver, every configuration knob Espressif
  ships is reachable directly, not just what the Arduino wrapper chose to
  expose.
- **Fine-grained configuration** via `menuconfig` — partition layout, log
  verbosity, FreeRTOS tick rate, stack sizes, security features — all
  without editing library source.
- **A real build system.** CMake-based, dependency-aware, incremental, and
  the same one Espressif's own examples and the component registry use.
- **Smaller, more predictable binaries** — no Arduino runtime glue you
  aren't using.

The trade-off is verbosity: what was one `#include <WiFi.h>` line becomes
explicit initialization calls. That's the right trade for firmware that
ships.

## Installing the toolchain

The full install (compiler, `idf.py`, Python environment) is documented on
[Espressif's site](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/get-started/) —
either the **VS Code ESP-IDF extension** (easiest) or the command-line
`install.sh`/`install.ps1` + `export.sh`/`export.ps1` scripts. Either way you
end up with an `idf.py` command on your `PATH` inside an "ESP-IDF terminal".
This module assumes that's done; it focuses on what you do once it is.

## Project layout

An ESP-IDF project is a directory tree, not a single `.ino` file:

```
my_project/
├── CMakeLists.txt          # top-level: names the project, includes IDF's build logic
├── sdkconfig                # generated: your menuconfig choices, one line per option
├── partitions.csv            # optional: custom flash layout (module 2-05 needs this)
└── main/
    ├── CMakeLists.txt      # registers this folder as a component
    └── main.c               # entry point: app_main()
```

Top-level `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(my_project)
```

`main/CMakeLists.txt`:

```cmake
idf_component_register(SRCS "main.c"
                        INCLUDE_DIRS ".")
```

And `main/main.c` — no `setup()`/`loop()`, just a single entry point:

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"

static const char *TAG = "main";

void app_main(void) {
  ESP_LOGI(TAG, "boot: %s %s", __DATE__, __TIME__);

  for (;;) {
    ESP_LOGI(TAG, "tick");
    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}
```

`app_main()` runs as an ordinary FreeRTOS task (named `main`, priority 1) —
exactly the same kind of task you created explicitly last module. It's
allowed to return (the task is then deleted and its stack freed) or, as
here, loop forever. `ESP_LOGI`/`W`/`E`/`D`/`V` replace `Serial.print` —
tagged, leveled logging built into every ESP-IDF component, controllable at
runtime with `esp_log_level_set(TAG, ESP_LOG_WARN)`.

## The `idf.py` command line

| Command | Effect |
|---------|--------|
| `idf.py set-target esp32` | Pick the chip target once, before the first build |
| `idf.py menuconfig` | Open the interactive configuration UI |
| `idf.py build` | Compile — incremental, CMake-driven |
| `idf.py -p /dev/ttyUSB0 flash` | Flash the built binary over serial |
| `idf.py monitor` | Open the serial monitor (`Ctrl+]` to exit) |
| `idf.py -p /dev/ttyUSB0 flash monitor` | The one you'll actually type — flash then watch |
| `idf.py fullclean` | Wipe the build directory (fixes stale-config weirdness) |
| `idf.py size` | Print flash/RAM usage, per component |

`idf.py monitor` is worth using over a plain serial terminal: it
symbolicates crash backtraces automatically, turning a raw address dump
into function names and line numbers.

## `menuconfig`

`idf.py menuconfig` opens a curses-based settings tree (arrow keys, `Enter`
to open a submenu, `Space` to toggle, `?` for help text on the highlighted
item). Settings you'll actually touch in this level:

- **Serial flasher config → Flash size** — must match your board or the
  filesystem modules (2-07) misbehave.
- **Partition Table → Partition Table** — switch to "Custom partition table
  CSV" when OTA (2-05) needs two app slots.
- **Component config → Log output → Default log verbosity** — turn down
  `ESP_LOGD`/`V` noise in release builds.
- **Component config → FreeRTOS → Tick rate (Hz)** — default 100 Hz;
  raising it (e.g. 1000 Hz) gives `vTaskDelay()` finer granularity at the
  cost of more timer-interrupt overhead.

Every choice is written to `sdkconfig` in the project root — check that file
into version control; it's your build's configuration, not a cache.

## Components

A **component** is any folder with its own `CMakeLists.txt` calling
`idf_component_register()` — `main/` is just the component ESP-IDF always
creates for you. Reusable code (a sensor driver, a shared utility) becomes
its own component folder with public headers under `include/`, referenced
from other components via `REQUIRES` in the register call:

```cmake
idf_component_register(SRCS "bme280.c"
                        INCLUDE_DIRS "include"
                        REQUIRES driver)
```

Third-party components — Espressif's own drivers and community libraries —
come from the [ESP Component Registry](https://components.espressif.com/)
via `idf_component.yml`:

```yaml
dependencies:
  espressif/led_strip: "^2.4.0"
```

Run `idf.py reconfigure` after editing it; the manager fetches and builds
the dependency alongside your own code, no manual library installation
step.

## Cheat sheet

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

## Exercise

Create a new ESP-IDF project (`idf.py create-project blinky`), set the
target to your chip, and reimplement module 1-07's two-task heartbeat idea
natively: an `app_main()` that creates one task blinking the onboard LED via
`gpio_set_level()` (from `driver/gpio.h`) at 2 Hz, and a second task that
logs free heap every 2 s with `ESP_LOGI` and `esp_get_free_heap_size()`.
Build, flash, and monitor it; then open `menuconfig`, raise the default log
verbosity to `Debug`, rebuild, and confirm you see additional framework log
lines you didn't before — proof the setting actually took effect.
