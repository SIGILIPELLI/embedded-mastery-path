# Debugging & Unit Testing

Module 2-08 gave you the machinery to survive a fault in the field. This
module is about the other half: finding out *why* it happened, and catching
the next one before it ever reaches flash. Embedded debugging is genuinely
harder than debugging a laptop program — you cannot attach a debugger to a
device on a rooftop, the act of printing changes the timing you're trying to
observe, and a memory bug can corrupt something a hundred milliseconds
before the crash you actually see. So you build layers: readable panics,
disciplined logging, core dumps for the crashes nobody watched, JTAG for the
ones you can reproduce, and unit tests for the logic that shouldn't need a
chip at all.

## Read the panic before you guess

The single most-wasted debugging hour in embedded work is spent theorising
about a crash whose cause was already printed on the serial port. An ESP32
panic looks like this:

```
Guru Meditation Error: Core 0 panic'ed (LoadProhibited). Exception was unhandled.
Core 0 register dump:
PC      : 0x400d1a2f  PS      : 0x00060730  A0      : 0x800d1b40  A1      : 0x3ffb5020
...
EXCVADDR: 0x00000000
Backtrace: 0x400d1a2f:0x3ffb5020 0x400d1b3d:0x3ffb5040 0x400d2011:0x3ffb5070
```

Three fields carry almost all the information:

| Field | Meaning |
|-------|---------|
| **Cause** | `LoadProhibited` / `StoreProhibited` = bad pointer read/write; `IllegalInstruction` = executing something that isn't code |
| **`EXCVADDR`** | The address the code tried to touch. `0x00000000` is a null dereference; garbage means a corrupted pointer |
| **Backtrace** | `PC:SP` pairs, innermost frame first — the call chain into the crash |

`IllegalInstruction` has one cause worth memorising because it looks
mysterious: **a task function that returned instead of calling
`vTaskDelete(NULL)`**. The task falls off the end of its function into
whatever bytes follow, and the CPU tries to execute them.

Use `idf.py monitor` rather than a plain serial terminal and the backtrace
arrives already symbolicated — function names, files, line numbers — because
the monitor cross-references your build's ELF file. If you only have a log
someone else captured, decode it by hand:

```bash
xtensa-esp32-elf-addr2line -pfiaC -e build/my_project.elf 0x400d1a2f 0x400d1b3d
```

You can also print a backtrace without crashing, which is the fastest way to
answer "who is calling this?":

```c
#include "esp_debug_helpers.h"

void suspicious_function(void)
{
    esp_backtrace_print(10);        /* up to 10 frames, to the console */
}
```

`esp_backtrace_print_all_tasks()` does the same for every task at once —
useful when the question is "what was everything else doing?" rather than
"how did I get here?".

By default a panic prints and reboots. `menuconfig` → **Component config →
ESP System Settings → Panic handler behaviour** (`CONFIG_ESP_SYSTEM_PANIC`)
also offers **halt** (freeze so you can inspect over JTAG) and **GDBStub**,
which turns the panic into a live GDB session over the same serial cable —
no JTAG hardware needed. For bench work that is the best-value setting in
the whole menu.

## Logging that survives contact with production

`ESP_LOGx` (module 2-02) has two independent level controls, and confusing
them is a common time sink:

- **`CONFIG_LOG_MAXIMUM_LEVEL`** is a *compile-time* ceiling. Anything above
  it is not merely hidden — it is not in the binary, so the format strings
  cost no flash and the calls cost no cycles.
- **`CONFIG_LOG_DEFAULT_LEVEL`** is the runtime default, adjustable per tag:

```c
#include "esp_log.h"

void tune_logging(void)
{
    esp_log_level_set("*", ESP_LOG_WARN);          /* quiet everything ... */
    esp_log_level_set("sampler", ESP_LOG_DEBUG);   /* ... except the suspect */
}
```

To keep debug output *available* in a shipped build, set the compile-time
maximum to `Debug` and the runtime default to `Info` — then a support command
over MQTT (module 2-04) can raise one tag's verbosity on a device in the
field without an OTA. For binary protocol work, dump the bytes rather than
describing them:

```c
ESP_LOG_BUFFER_HEXDUMP(TAG, rx_buf, len, ESP_LOG_DEBUG);
```

!!! warning "Logging changes the timing you're trying to measure"
    A log line is not free: at 115200 baud each character takes roughly
    90 µs to leave the UART, and the call blocks while it does. Drop one
    into a 1 kHz sampling loop and the bug moves, disappears, or is replaced
    by a new one — the classic heisenbug. Two rules follow. **Never log from
    an ISR**: set a flag or push to a queue and log from the task (module
    1-07). And when you must instrument fast code, toggle a GPIO and watch
    it on a logic analyser instead — a pin toggle costs a handful of cycles,
    a log line costs milliseconds.

## Core dumps: the crash nobody was watching

A backtrace only helps if someone had a terminal open. **Core dumps** fix
that: on panic, ESP-IDF writes a snapshot of the task stacks into a flash
partition, where it survives the reboot and waits for you.

Enable it in `menuconfig` → **Component config → Core dump → Data
destination → Flash** (`CONFIG_ESP_COREDUMP_ENABLE_TO_FLASH`), and give it a
partition:

```csv
# Name,     Type, SubType,  Offset,   Size
nvs,        data, nvs,      0x9000,   0x4000,
otadata,    data, ota,      0xd000,   0x2000,
phy_init,   data, phy,      0xf000,   0x1000,
coredump,   data, coredump, ,         0x10000,
ota_0,      app,  ota_0,    0x20000,  0x1A0000,
ota_1,      app,  ota_1,    ,         0x1A0000,
```

After the device reboots, read it back over the same USB cable:

```bash
idf.py coredump-info      # summary: panic reason, faulting task, backtrace
idf.py coredump-debug     # full GDB session against the crashed state
```

`coredump-debug` is the good one. It reconstructs an ELF from the dump and
drops you into GDB, so you can `bt`, switch between tasks, and print locals
from a crash that happened yesterday on a device you weren't watching.

Firmware can also notice a stored dump at boot and report it:

```c
#include "esp_core_dump.h"

void report_last_crash(void)
{
    size_t addr = 0, size = 0;

    if (esp_core_dump_image_get(&addr, &size) != ESP_OK) {
        ESP_LOGI(TAG, "no core dump stored");
        return;
    }
    ESP_LOGW(TAG, "core dump: %u bytes at 0x%08x", (unsigned)size, (unsigned)addr);

    /* publish the fact over MQTT so the dashboard flags this device, then
       clear the partition so the *next* crash has somewhere to go */
    ESP_ERROR_CHECK(esp_core_dump_image_erase());
}
```

Erasing matters: the partition holds exactly one dump. Leave a stale one
there and the crash you actually care about is silently discarded.

## JTAG, when printf has run out

Everything above is post-mortem. JTAG gives you breakpoints, single
stepping, and live variable inspection on a running chip — the same
experience as a desktop debugger. The classic ESP32 needs an external probe
(ESP-Prog, or an ESP-WROVER-KIT with its onboard FT2232H); the ESP32-C3, -S3
and later have a USB-Serial-JTAG peripheral built in, so one USB cable is
enough. `idf.py` drives the whole stack:

| Command | Effect |
|---------|--------|
| `idf.py openocd` | Start the OpenOCD server with your board's configuration |
| `idf.py gdb` | Attach GDB, using the project's ELF automatically |
| `idf.py gdbgui` | Same, with a browser-based front end |
| `idf.py openocd gdb` | Both — `idf.py` orders background and interactive actions for you |

On the classic ESP32, **GPIO12–GPIO15 are the JTAG pins** (MTDI, MTCK, MTMS,
MTDO), so a debug session conflicts with anything you wired there. GPIO12
(MTDI) is worse than a conflict: it is a bootstrapping pin sampled at reset
to select the flash chip's supply voltage, so a probe driving it can stop the
device booting at all. Plan the pinout with JTAG in mind, or accept that this
build is not debuggable that way.

## Heap bugs: leaks and corruption

Two failure modes dominate long-running firmware, and neither one crashes
where the bug actually is.

**A leak** shows up as weeks of uptime followed by allocation failures.
Track the trend, not the instant value: `esp_get_free_heap_size()` fluctuates
constantly, but `esp_get_minimum_free_heap_size()` only ever falls. If it
keeps falling over days, something leaks. To find out what, use the
standalone heap tracer (`CONFIG_HEAP_TRACING_STANDALONE`):

```c
#include "esp_heap_trace.h"

#define NUM_RECORDS 100
static heap_trace_record_t s_records[NUM_RECORDS];   /* must be static */

void find_the_leak(void)
{
    ESP_ERROR_CHECK(heap_trace_init_standalone(s_records, NUM_RECORDS));
    ESP_ERROR_CHECK(heap_trace_start(HEAP_TRACE_LEAKS));

    run_suspect_workload();          /* one full cycle of whatever leaks */

    ESP_ERROR_CHECK(heap_trace_stop());
    heap_trace_dump();               /* every unfreed block, with its caller */
}
```

`HEAP_TRACE_LEAKS` records only allocations that were never freed, and the
dump names the calling address for each. Run one cycle of something that
should be net-neutral, and whatever is left in the list is your leak.

**Corruption** — a write past the end of a buffer — is nastier, because the
symptom appears whenever the damaged neighbour is next used, which can be
anywhere. Turn `CONFIG_HEAP_CORRUPTION_DETECTION` up to **Comprehensive**
while hunting it: the allocator puts canary words around every block, fills
fresh allocations with `0xCE` and freed memory with `0xFE`, so a
use-after-free shows up as suspiciously `0xFEFEFEFE` data instead of
plausible-looking garbage. Then bisect with explicit checks:

```c
#include "esp_heap_caps.h"

if (!heap_caps_check_integrity_all(true)) {     /* true = print what's broken */
    ESP_LOGE(TAG, "heap already corrupted at this point");
}
```

Move that call around until it flips from passing to failing; the overrun is
between the last good position and the first bad one. Comprehensive
poisoning is slow and memory-hungry — it is a debugging setting, not a
shipping one.

The third member of the family, **stack overflow**, was covered in module
2-01. One ESP-IDF-specific detail belongs here though: this port's
`uxTaskGetStackHighWaterMark()` reports the worst-ever free stack **in
bytes**, matching the byte-based stack sizes `xTaskCreate()` takes on this
platform, whereas vanilla FreeRTOS documentation describes it in words. The
factor of four has sent plenty of people chasing a problem they didn't have.
Also leave `CONFIG_FREERTOS_CHECK_STACKOVERFLOW` enabled (Canary is the
default) so an overrun is reported by task name instead of quietly
corrupting a neighbour.

## Unit testing the logic that doesn't need a chip

Most firmware bugs are not in the drivers. They are in a CRC routine, a
ring-buffer index, a parser, a state machine, a unit conversion — plain C
logic that has no business requiring an ESP32 to test. Split that logic into
its own component (module 2-02) with no hardware calls in it, and its tests
run in seconds, on every commit.

ESP-IDF ships **Unity**. Test files live in a `test` subdirectory of the
component under test, must be named starting with `test`, and that directory
is itself a component:

```c
/* components/sample_buf/test/test_sample_buf.c */
#include "unity.h"
#include "sample_buf.h"

TEST_CASE("a fresh buffer reports empty", "[sample_buf]")
{
    sample_buf_t b;
    sample_buf_init(&b);
    TEST_ASSERT_TRUE(sample_buf_is_empty(&b));
    TEST_ASSERT_EQUAL_INT(0, sample_buf_count(&b));
}

TEST_CASE("pushing past capacity drops the oldest sample", "[sample_buf]")
{
    sample_buf_t b;
    sample_buf_init(&b);
    for (int i = 0; i < SAMPLE_BUF_CAP + 3; i++) {
        sample_buf_push(&b, (float)i);
    }
    TEST_ASSERT_EQUAL_INT(SAMPLE_BUF_CAP, sample_buf_count(&b));
    TEST_ASSERT_EQUAL_FLOAT(3.0f, sample_buf_peek(&b, 0));   /* 0,1,2 evicted */
}

TEST_CASE("a null buffer is rejected, not dereferenced", "[sample_buf]")
{
    TEST_ASSERT_EQUAL(ESP_ERR_INVALID_ARG, sample_buf_push(NULL, 1.0f));
}
```

```cmake
# components/sample_buf/test/CMakeLists.txt
idf_component_register(SRC_DIRS "."
                       INCLUDE_DIRS "."
                       REQUIRES unity sample_buf)
```

There is no `main()` — Unity's platform layer provides one. Build and flash
the test app, then press Enter in the monitor to get an interactive menu:
type a test's number, its name in quotes, `[sample_buf]` to run one tag, or
`*` to run everything. For CI you don't want a human pressing keys, so
ESP-IDF integrates **`pytest-embedded`**, which drives that menu
automatically and reports pass/fail per case.

The assertions worth knowing: `TEST_ASSERT_TRUE`, `TEST_ASSERT_EQUAL_INT`,
`TEST_ASSERT_EQUAL_FLOAT` (tolerance-based — never compare floats with the
integer macro), `TEST_ASSERT_EQUAL_STRING`, `TEST_ASSERT_EQUAL_HEX8_ARRAY`
for buffers, and `TEST_FAIL_MESSAGE` for an explicit give-up.

Test the ugly cases, not the happy path: empty input, one element, exactly
full, one past full, null pointers, integer wraparound. That is where the
field failures actually live.

## Cheat sheet

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

## Exercise

Take one piece of pure logic out of an earlier project — a ring buffer, a
CSV line formatter, or the command parser from module 2-04 — and move it
into its own component with no ESP-IDF hardware calls in it. Write at least
six Unity `TEST_CASE`s covering empty, single-element, exactly-full,
one-past-full, null-pointer and wraparound inputs. Run them, then
deliberately break one boundary condition in the implementation (change a
`<=` to a `<`) and confirm that exactly the test you predicted goes red.

Then build a crash-forensics loop. Add a `coredump` partition, enable core
dump to flash, and add three serial commands that crash the device three
different ways: a null dereference, a task function that `return`s instead
of calling `vTaskDelete(NULL)`, and a 64-byte write into a 16-byte
`malloc()`. Reboot after each and run `idf.py coredump-info` — note in a
comment which of the three the dump pinpoints and which it doesn't, then
re-run the third with `CONFIG_HEAP_CORRUPTION_DETECTION` set to
Comprehensive and record how much closer that gets you to the real bug.
Finally, have `app_main()` call `esp_core_dump_image_get()` at startup, log
that a dump is waiting, and erase it — proving the device can report its own
crashes with nobody watching the cable.
