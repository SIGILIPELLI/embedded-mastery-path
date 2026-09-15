---
description: "Memory & Power Basics — Two resources define what an embedded system can do: memory (a few KB to a few hundred KB, fixed forever at design time) and power…"
---

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

## How It Actually Works

**Why SRAM corruption causes "impossible" bugs**: on both AVR and ESP32, the
**stack** (function call frames, local variables, return addresses) and the
**heap** (dynamic allocations, including every hidden `String` allocation)
grow toward each other from opposite ends of the same physical SRAM address
range — the stack grows downward from high addresses, the heap grows upward
from just above your global variables. There is no hardware guard page in
between on these small chips (unlike an MMU-equipped application processor).
When they collide, a function call's return address or a local variable
literally gets overwritten by heap data (or vice versa) — the CPU doesn't
detect this as an error at all, it just executes whatever garbage address it
finds when the function returns, which is why the symptom is a spontaneous
reset (jumping to an invalid address trips the watchdog or triggers a fault)
rather than a clean error message.

**Why flash "wears out"**: flash memory stores bits by trapping electrical
charge on a floating gate inside each memory cell, insulated by a thin oxide
layer. Writing a `1`→`0` bit requires an erase-then-program cycle that forces
charge across that oxide layer via quantum tunneling; each cycle
physically stresses and gradually degrades the oxide's insulating ability
until, after tens of thousands of cycles, it can no longer reliably hold
charge — a hardware failure mode, not a software one. This is exactly why
this module is careful to say "write on change, not per loop": every
`prefs.putUInt()`/`EEPROM.put()` call that changes the underlying flash sector
consumes part of its finite lifetime, and code that saves every second
instead of every hour can wear out a sector in weeks.

**What actually happens in deep sleep**: an ESP32 in deep sleep powers down
the CPU cores, most SRAM, and most peripherals entirely — literally cutting
their supply rails — leaving only a small, separately-powered **RTC domain**
containing a low-power timer, a handful of RTC GPIOs, and a tiny slice of RTC
memory alive. `RTC_DATA_ATTR` places a variable's storage in that surviving
RTC memory region instead of ordinary SRAM, specifically so it isn't wiped
when the rest of SRAM loses power. Waking is indistinguishable from a power-on
reset from the CPU's point of view — the boot ROM runs, `app_main`/`setup()`
runs from scratch — except the boot ROM first checks
`esp_sleep_get_wakeup_cause()` (a status register the RTC domain sets before
waking the rest of the chip) so your code can tell "fresh power-on" from
"woke from a timer/GPIO." This full power-down of the CPU is exactly what
gets deep sleep down to ~10 µA — there is simply no clocked digital logic
left running to consume current, only quiescent leakage in the RTC domain.

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
