# 07 · Timers & Interrupts

Every sketch so far had one job, so `delay()` was fine. Real devices do many
things at once — blink a status LED, poll a sensor every 2 s, answer serial
commands instantly — and `delay()` makes that impossible, because it freezes
*everything*. This module teaches the two standard escapes: **cooperative
scheduling with `millis()`** (the workhorse pattern of Arduino programming)
and **hardware interrupts** (for events too fast or too important to poll).

## Why `delay()` blocks

`delay(2000)` is a busy-wait: the CPU sits in a loop checking the clock for
two seconds. Nothing else in `loop()` runs. Combine "read sensor every 2 s"
and "respond to serial commands" with delays, and commands take up to 2 s to
answer. Add "blink LED at 5 Hz" and it's simply impossible — the timings
fight.

## `millis()`-based scheduling

`millis()` returns the number of milliseconds since boot as a `uint32_t`,
ticking in the background regardless of what your code does. Instead of
*waiting* for time to pass, you *check whether enough time has passed*:

```cpp
uint32_t lastBlink = 0;

void loop() {
  if (millis() - lastBlink >= 500) {   // 500 ms elapsed since last toggle?
    lastBlink = millis();
    digitalWrite(LED_PIN, !digitalRead(LED_PIN));
  }
  // falls straight through — loop() keeps spinning
}
```

Because nothing blocks, you can stack as many of these as you like. Three
independent "tasks", one `loop()`:

```cpp
const uint8_t LED_PIN = 2;

uint32_t lastBlink = 0;
uint32_t lastReport = 0;

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  uint32_t now = millis();             // sample the clock ONCE per pass

  // Task 1: heartbeat LED at 2 Hz
  if (now - lastBlink >= 250) {
    lastBlink = now;
    digitalWrite(LED_PIN, !digitalRead(LED_PIN));
  }

  // Task 2: status report every 2 seconds
  if (now - lastReport >= 2000) {
    lastReport = now;
    Serial.print(F("uptime_s="));
    Serial.println(now / 1000);
  }

  // Task 3: serial commands — answered within microseconds, always
  if (Serial.available() > 0) {
    String cmd = Serial.readStringUntil('\n');
    cmd.trim();
    if (cmd == "ping") Serial.println(F("pong"));
  }
}
```

This is **cooperative multitasking**: each task runs briefly and returns.
The one rule: **no task may block** — no `delay()`, no `while` loops waiting
on hardware. One slow task stalls all the others.

!!! note "Why `now - lastBlink` and not `now >= lastBlink + 500`"
    `millis()` wraps to 0 after ~49.7 days. Written as a *difference* of
    unsigned numbers, `now - lastBlink` stays correct straight through the
    wraparound (unsigned arithmetic wraps too, cancelling out). Written as a
    comparison of absolute times, it breaks on day 49. Devices are expected
    to run for months — always use the subtraction form.

## Hardware interrupts

Polling has a blind spot: an event shorter than one trip around `loop()` can
be missed entirely. An **interrupt** flips the model — the hardware pauses
your code the instant a pin changes, runs a special function (an **ISR**,
interrupt service routine), then resumes your code exactly where it was.

```cpp
const uint8_t BUTTON_PIN = 4;
const uint8_t LED_PIN = 2;

volatile uint32_t pressCount = 0;          // shared with the ISR → volatile!

void IRAM_ATTR onButtonPress() {           // the ISR: runs on the falling edge
  pressCount++;                            // short and simple — nothing else
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(BUTTON_PIN), onButtonPress, FALLING);
}

void loop() {
  static uint32_t lastReported = 0;
  if (pressCount != lastReported) {        // notice changes from the ISR
    lastReported = pressCount;
    Serial.print(F("presses: "));
    Serial.println(lastReported);
  }
}
```

- `digitalPinToInterrupt(pin)` maps the pin to its interrupt number (on the
  Uno only pins 2 and 3 can interrupt; on the ESP32, any GPIO).
- The trigger mode is `FALLING` here (HIGH→LOW, i.e. the press with a
  pull-up); other options: `RISING`, `CHANGE`, `LOW`.
- `IRAM_ATTR` is ESP32-specific: it places the ISR in RAM, required so it's
  runnable even during flash operations. On AVR it's unnecessary — define it
  as a no-op or omit for Uno-only code.

### The ISR rules

An ISR runs in a delicate context — normal code is frozen, and on most cores
other interrupts are masked. The rules are strict:

1. **Keep it short.** Set a flag, bump a counter, capture `millis()` — get
   out. Microseconds, not milliseconds.
2. **No `Serial.print`, no `delay()`, no I2C/SPI calls** inside an ISR — they
   rely on interrupts themselves and can deadlock or crash.
3. **Every variable shared with an ISR must be `volatile`** — it tells the
   compiler the value can change at any moment, so it must actually read
   memory each time instead of caching it in a register. Without it, `loop()`
   may *never see* the ISR's updates (the compiler "optimized" the read
   away).
4. **Multi-byte shared values need atomic access** on 8-bit chips: an Uno
   reads a `uint32_t` in four steps, and an interrupt can strike mid-read.
   Guard with `noInterrupts(); copy = pressCount; interrupts();` — a
   *critical section*.

The standard division of labor, exactly as in the example: **ISR records the
event, `loop()` does the work.**

## Cheat sheet

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

## Exercise

Rebuild module 3's two-button counter (Wokwi, ESP32) the professional way:
button presses captured by a `FALLING` interrupt that only records
`millis()` into a `volatile` variable; `loop()` notices the new timestamp,
applies a 200 ms software lockout for debouncing (ignore a press within
200 ms of the previous accepted one), and updates the count. Meanwhile a
heartbeat LED must blink at a steady 2 Hz via `millis()` scheduling — verify
it never stutters no matter how fast you hammer the button, then explain (in
a comment) why the `delay()`-based debounce from module 3 could not make that
guarantee.
