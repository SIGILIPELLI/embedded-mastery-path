# 02 · C/C++ for Embedded

Arduino sketches are C++ (with a thin layer of convenience), but embedded
C++ has a different center of gravity than desktop C++: memory is tiny and
fixed, there's no operating system to clean up after you, and the size of
every variable matters. This module covers the language essentials you'll use
in every sketch — types and their real sizes, constants, functions — and the
classic beginner pitfalls (`String`, integer overflow) that behave very
differently on a microcontroller than on a PC.

## Types and their sizes on a microcontroller

On desktop machines you rarely think about whether `int` is 4 or 8 bytes. On
microcontrollers you must: an **`int` is 2 bytes on an Arduino Uno** (AVR,
8-bit) but **4 bytes on an ESP32** (32-bit). Code that assumes one size can
break on the other.

```cpp
void setup() {
  Serial.begin(115200);
  Serial.print("int:       "); Serial.println(sizeof(int));        // 2 on Uno, 4 on ESP32
  Serial.print("long:      "); Serial.println(sizeof(long));       // 4 on both
  Serial.print("float:     "); Serial.println(sizeof(float));      // 4 on both
  Serial.print("double:    "); Serial.println(sizeof(double));     // 4 on Uno(!), 8 on ESP32
  Serial.print("bool/byte: "); Serial.println(sizeof(bool));       // 1 on both
}

void loop() {}
```

Because of this, embedded code prefers the **fixed-width types** from
`<stdint.h>` (always available in Arduino):

```cpp
uint8_t  brightness = 255;        // exactly 1 byte, 0..255
int16_t  temperature10 = -125;    // exactly 2 bytes, -32768..32767 (e.g. -12.5 °C × 10)
uint32_t uptimeMs = 0;            // exactly 4 bytes, 0..4,294,967,295
```

The names encode signedness and bit width: `uint8_t` = unsigned 8-bit,
`int32_t` = signed 32-bit. When a value's range matters — and on hardware it
almost always does — say the size explicitly.

### Overflow is silent

Unsigned types wrap around; small signed types overflow at surprisingly small
values:

```cpp
uint8_t counter = 250;
for (int i = 0; i < 10; i++) {
  counter++;                 // 251, 252, 253, 254, 255, 0, 1, 2, 3, 4
}

int16_t x = 30000;
x = x + 10000;               // overflow! wraps to -25536 — no warning, no crash
```

A classic real-world bug: storing `millis()` (a `uint32_t` millisecond
counter) in an `int` and having timing logic break after ~32 seconds on an
Uno. Match the type to the data.

## `const` vs `#define`

Both create named constants; prefer `const` (or `constexpr`) because it's
typed and scoped:

```cpp
const uint8_t LED_PIN = 5;            // typed constant — compiler checks usage
const float TEMP_ALARM_C = 30.0;

#define BLINK_MS 500                  // text substitution — no type, global forever
```

`#define` does blind text replacement before compilation, which invites
classic traps (`#define WIDTH 2+3` then `WIDTH*2` becomes `2+3*2` = 8, not
10). You'll still see `#define` everywhere in Arduino libraries and example
code — recognize it, but write `const` yourself. Neither costs RAM for simple
integer constants: the compiler folds them into the code.

## Functions

Nothing exotic — standard C++ functions. One Arduino-specific convenience:
the IDE auto-generates forward declarations, so ordering functions after
`loop()` still compiles.

```cpp
const uint8_t LED_PIN = 2;

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(115200);
}

void loop() {
  blinkTimes(3, 200);
  float tempF = celsiusToFahrenheit(21.5);
  Serial.println(tempF);   // 70.70
  delay(2000);
}

// Reusable building blocks keep loop() readable
void blinkTimes(uint8_t times, uint16_t onOffMs) {
  for (uint8_t i = 0; i < times; i++) {
    digitalWrite(LED_PIN, HIGH);
    delay(onOffMs);
    digitalWrite(LED_PIN, LOW);
    delay(onOffMs);
  }
}

float celsiusToFahrenheit(float c) {
  return c * 9.0 / 5.0 + 32.0;
}
```

Break every distinct job into a function from day one — capstone-sized
sketches (module 10) stay manageable only because `loop()` reads as a list of
function calls.

## The `String` pitfall — and what to use instead

Arduino's `String` class is convenient and appears in countless tutorials:

```cpp
String msg = "Temp: " + String(21.5) + " C";   // works... for a while
```

Every `String` concatenation allocates memory on the **heap**. On a chip with
2 KB of RAM, repeated allocate/free cycles **fragment** the heap — free
memory gets chopped into pieces too small to use, and hours or days in, an
allocation fails and the board resets or misbehaves. This is the single most
common cause of "my Arduino project crashes after running all night".

The robust alternative: **fixed-size `char` arrays** and `snprintf`:

```cpp
char msg[32];                                   // fixed buffer, allocated once
float tempC = 21.5;

// snprintf never writes past the buffer size you give it
snprintf(msg, sizeof(msg), "Temp: %d.%d C",
         (int)tempC, (int)(tempC * 10) % 10);   // "Temp: 21.5 C"
Serial.println(msg);
```

(Why the integer gymnastics? On AVR boards, `%f` support is disabled by
default to save flash. `(int)t` and `(int)(t*10)%10` print one decimal place
portably. On ESP32, `snprintf(msg, sizeof(msg), "Temp: %.1f C", tempC)` works
directly.)

Rule of thumb for this course: `String` is acceptable in short-lived demo
sketches; anything meant to run unattended uses `char` buffers.

## Program memory vs RAM

Your compiled code lives in **flash** (plentiful-ish: 32 KB on Uno, 4 MB on
ESP32) and your variables live in **SRAM** (scarce: 2 KB on Uno, ~520 KB on
ESP32). String literals are data, so this innocent line costs RAM:

```cpp
Serial.println("Starting environment monitor v1.0...");   // ~40 bytes of SRAM on Uno
```

On AVR boards the `F()` macro keeps the literal in flash and reads it out on
demand:

```cpp
Serial.println(F("Starting environment monitor v1.0..."));  // ~0 bytes of SRAM
```

On ESP32 literals effectively stay in flash anyway and `F()` is a harmless
no-op — so using `F()` for fixed message strings is a good portable habit.
Module 8 covers the memory map in detail.

## Cheat sheet

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

## Exercise

Write a sketch with a function `formatReading(char *buf, size_t bufSize,
uint16_t id, float tempC)` that uses `snprintf` to produce a line like
`sensor 3: 21.5 C` into the caller's buffer. In `loop()`, call it once per
second with an incrementing `id` (stored in a `uint8_t` — deliberately) and a
temperature that rises 0.5 °C each call, printing the result over serial. Run
it in Wokwi and watch what happens to the id after 255 — then fix the type.
Bonus: print `sizeof(int)` at startup, run the same sketch on both the Uno
and ESP32 templates, and compare.
