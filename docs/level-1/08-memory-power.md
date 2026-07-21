# 08 · Memory & Power Basics

Two resources define what an embedded system can do: **memory** (a few KB to
a few hundred KB, fixed forever at design time) and **power** (mains, USB, or
a battery that must last). This module maps out the three kinds of memory in
your chip, shows how to measure what you're using, and introduces the ESP32's
**deep sleep** — the difference between a battery project lasting hours and
lasting months.

## The three memories

| Memory | Uno (ATmega328P) | ESP32 | Survives power-off? | Holds |
|--------|------------------|-------|---------------------|-------|
| **Flash** | 32 KB | 4 MB (typical) | Yes | Your program, constants |
| **SRAM** | 2 KB | ~520 KB | No | Variables, stack, heap |
| **EEPROM / NVS** | 1 KB EEPROM | NVS (in flash) | Yes | Settings, counters — data you write at runtime |

- **Flash** is written when you upload; your code executes from it, and it
  keeps the program through power cycles. Wears out after ~10k–100k *erase*
  cycles — irrelevant for uploads, very relevant if firmware logs data to it
  (Level 2 topic).
- **SRAM** is working memory — every variable, the call stack, the heap. It's
  the scarcest resource on small chips, and running out doesn't give a tidy
  error: the stack silently collides with your data, corrupting either — the
  classic cause of random resets and impossible bugs.
- **EEPROM/NVS** is small persistent storage your *program* can write:
  calibration values, a WiFi password, a boot counter.

The compile summary you saw in module 1 reports the first two. Watch the RAM
number: past ~75% on an Uno, stack collisions become a real risk.

## Measuring free memory at runtime

On the ESP32 it's built in:

```cpp
void loop() {
  Serial.print(F("free heap: "));
  Serial.println(ESP.getFreeHeap());       // bytes currently available
  Serial.print(F("min free ever: "));
  Serial.println(ESP.getMinFreeHeap());    // low-water mark since boot — the number to watch
  delay(5000);
}
```

`getMinFreeHeap()` is the diagnostic gold: a *steadily falling* value is a
**memory leak** (often `String` churn — module 2) and predicts exactly when
the device will die. Log it in anything meant to run for weeks.

On the Uno there's no API, but the classic trick measures the gap between the
heap and the stack:

```cpp
int freeRam() {
  extern int __heap_start, *__brkval;
  int v;
  return (int)&v - (__brkval == 0 ? (int)&__heap_start : (int)__brkval);
}
```

You don't need to understand the internals yet — call it, log it, watch the
trend, exactly as with the ESP32 version.

## Persistent data

**Classic Arduino** — the `EEPROM` library reads/writes single bytes or whole
structs:

```cpp
#include <EEPROM.h>

uint32_t bootCount;
EEPROM.get(0, bootCount);        // read 4 bytes at address 0
bootCount++;
EEPROM.put(0, bootCount);        // write back (put skips unchanged bytes)
```

**ESP32** — EEPROM is emulated; the native mechanism is **NVS**
(non-volatile storage), a key-value store used via `Preferences`:

```cpp
#include <Preferences.h>
Preferences prefs;

void setup() {
  Serial.begin(115200);
  prefs.begin("app", false);                          // namespace, read-write
  uint32_t boots = prefs.getUInt("boots", 0) + 1;     // default 0 if absent
  prefs.putUInt("boots", boots);
  prefs.end();
  Serial.printf("boot number %u\n", boots);
}
```

Remember flash wear: save when a value *changes* (or rarely), never
unconditionally in `loop()`.

## Power: why sleep matters

Rough ESP32 numbers tell the whole story:

| State | Current (order of magnitude) |
|-------|------------------------------|
| Active, WiFi transmitting | 150–250 mA |
| Active, WiFi off | 30–50 mA |
| **Deep sleep** | **~10 µA** |

A 2000 mAh battery lasts about **two days** at 40 mA — or **years** at 10 µA.
Battery devices therefore live a duty-cycled life: wake, measure, transmit,
sleep. The fraction of time awake sets the battery life.

## ESP32 deep sleep

In deep sleep the CPU and RAM power off; only the ultra-low-power RTC domain
keeps running. Waking is a **reboot** — `setup()` runs again — so state you
need across sleeps goes in NVS or in RTC memory:

```cpp
const uint64_t SLEEP_S = 30;

RTC_DATA_ATTR uint32_t wakeCount = 0;    // RTC_DATA_ATTR: survives deep sleep
                                         // (but not power-off — that's what NVS is for)

void setup() {
  Serial.begin(115200);
  wakeCount++;

  Serial.printf("Wake #%u, cause %d\n", wakeCount, esp_sleep_get_wakeup_cause());

  // ... read sensor, log or transmit the value ...

  Serial.printf("Sleeping %llu s...\n", SLEEP_S);
  Serial.flush();                                    // let the UART finish printing!
  esp_sleep_enable_timer_wakeup(SLEEP_S * 1000000ULL);  // microseconds
  esp_deep_sleep_start();                            // never returns
}

void loop() {}                                       // never reached
```

Run it in Wokwi (ESP32 template — the simulator supports deep sleep): the
serial monitor shows a boot banner and an incrementing wake count every 30
simulated seconds. Note the structure — *everything happens in `setup()`*,
because every wake is a fresh boot.

Timer wakeup is one of several sources: `esp_sleep_enable_ext0_wakeup(pin,
level)` wakes on a GPIO (a button, a sensor's alarm output), and sources can
be combined — `esp_sleep_get_wakeup_cause()` tells you which one fired.

## Cheat sheet

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

## Exercise

Build a Wokwi ESP32 sketch simulating a battery sensor node: on each wake,
increment a counter in **NVS** (so it survives even full power-off, unlike
`RTC_DATA_ATTR` — keep both and print both to see the difference), read a
potentiometer as a stand-in sensor, print boot number, both counters, the
reading, and `ESP.getFreeHeap()`, then deep-sleep 15 s. Let it run several
cycles and confirm: NVS counter always climbs, RTC counter climbs per sleep
cycle, heap stays flat. Bonus: estimate battery life at 2000 mAh if the node
is awake 200 ms at 45 mA per cycle and asleep at 10 µA otherwise — first for
a 15 s cycle, then for 10 min.
