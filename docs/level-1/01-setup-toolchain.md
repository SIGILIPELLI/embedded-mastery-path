# 01 · Setup & Toolchain

An embedded program (a *sketch*, in Arduino terms) doesn't run on your
computer — it's compiled on your computer into machine code for a
*microcontroller* (a small, self-contained chip with its own CPU, RAM, and
flash storage), then uploaded over USB and runs there forever, restarting
every time the board gets power. This module sets up the two ways you'll run
every example in this course: the **Arduino IDE** (for real boards) and the
**Wokwi online simulator** (no hardware needed at all).

## Option A — Wokwi: zero-install, zero-hardware

[Wokwi](https://wokwi.com/) is a free browser-based simulator with virtual
Arduino Uno, ESP32, and Raspberry Pi Pico boards plus virtual parts: LEDs,
buttons, potentiometers, DHT22 temperature sensors, OLED displays, and more.

1. Open [wokwi.com](https://wokwi.com/) and click **Arduino Uno** (or
   **ESP32**) under "Start from a template".
2. You get a code editor on the left and a virtual board on the right.
3. Click the green **▶ play button** — Wokwi compiles your sketch with the
   real Arduino toolchain and runs it on a simulated chip.
4. Add parts with the blue **+** button in the diagram pane, then drag wires
   between pins.

Every sketch in this course runs in Wokwi unmodified. When a lesson uses a
part (say, a DHT22 sensor), the wiring is described so you can recreate it in
the diagram pane. If you later buy a real board, the same code uploads to it
with no changes.

!!! tip "Recommended: simulate ESP32"
    The Uno is the classic beginner board, but this course's later modules
    (WiFi, deep sleep) need an **ESP32** — and Wokwi simulates it well,
    including its WiFi. Using the ESP32 template from day one means you never
    have to switch.

## Option B — Arduino IDE 2.x for real boards

If you do have hardware (or plan to), install the
[Arduino IDE 2.x](https://www.arduino.cc/en/software) for Windows, macOS, or
Linux. It bundles the compiler toolchain, a serial monitor, and a library
manager.

### Installing board support (the ESP32 core)

Out of the box the IDE only knows official Arduino boards (Uno, Mega,
Nano...). Support for other chips comes as a **board package** (a "core"):

1. **File → Preferences → Additional boards manager URLs**, add:
   ```
   https://espressif.github.io/arduino-esp32/package_esp32_index.json
   ```
2. **Tools → Board → Boards Manager**, search **esp32**, install
   **"esp32 by Espressif Systems"**.
3. Plug in the board, then pick it under **Tools → Board** (e.g. *ESP32 Dev
   Module*) and its serial port under **Tools → Port**.

The board package contains the cross-compiler (e.g. `xtensa-esp32-elf-gcc`),
the chip's hardware definitions, and the upload tool — the IDE picks the
right ones automatically based on the selected board.

## Anatomy of a sketch: `setup()` and `loop()`

Every Arduino program has exactly two required functions:

```cpp
void setup() {
  // Runs ONCE when the board powers on or resets.
  // Configure pins, start Serial, initialize sensors here.
  pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
  // Runs over and over, forever, as fast as it can.
  digitalWrite(LED_BUILTIN, HIGH);  // LED on
  delay(500);                       // wait 500 milliseconds
  digitalWrite(LED_BUILTIN, LOW);   // LED off
  delay(500);
}
```

This is **Blink**, the "Hello, World" of embedded systems. Behind the scenes
the framework supplies a hidden `main()` that calls your `setup()` once and
then calls `loop()` in an infinite loop. There is no operating system and no
exit — when `loop()` returns, it is simply called again.

`LED_BUILTIN` is a constant naming the pin wired to the small LED most boards
have onboard (pin 13 on an Uno, pin 2 on most ESP32 dev boards). On a real
board or in Wokwi, running this makes that LED blink once per second.

## Compiling and uploading

In the Arduino IDE:

- **✓ Verify** compiles the sketch without uploading — your fastest check
  that the code is valid.
- **→ Upload** compiles, then flashes the machine code into the board over
  USB. The board resets and starts running your program immediately.

The compile output ends with a line worth reading from day one:

```
Sketch uses 924 bytes (2%) of program storage space. Maximum is 32256 bytes.
Global variables use 9 bytes (0%) of dynamic memory, leaving 2039 bytes for
local variables. Maximum is 2048 bytes.
```

That's your **flash** (program storage) and **RAM** budget — an Uno has just
32 KB of flash and 2 KB of RAM. Module 8 digs into what these numbers mean;
for now, just notice they exist. Embedded programming is programming against
real, small limits.

In Wokwi, the ▶ button does verify + upload + run in one step, and the
simulated serial monitor appears below the diagram automatically.

!!! warning "Upload problems on real ESP32 boards"
    Some ESP32 dev boards need you to hold the **BOOT** button while the IDE
    prints `Connecting....` during upload. If uploads fail with a timeout,
    that's the first thing to try. (Simulator users: this problem doesn't
    exist in Wokwi — one of several reasons it's great for learning.)

## How It Actually Works

"Verify" and "Upload" are not one step wearing two names — they're a whole
toolchain pipeline, and it's worth knowing the stages because every error
message you'll ever see names one of them:

1. **Preprocessor** expands `#include`s and macros, and the Arduino build
   system additionally scans your `.ino` for function definitions and
   auto-generates forward declarations (so you can call `blink()` from
   `loop()` even though it's defined below it — an Arduino-specific
   convenience real C++ doesn't give you).
2. **Compiler** (`avr-gcc` for Uno, `xtensa-esp32-elf-gcc` for ESP32 — the
   board package installed exactly these) turns your C++ into target-specific
   **assembly**, then an **object file** (`.o`) of machine code plus
   unresolved symbol references.
3. **Linker** resolves those references against the Arduino core library and
   your sketch's other object files, and — critically — lays out where every
   function and global variable will physically live in flash and RAM
   according to a **linker script** for that exact chip. This is where the
   "Sketch uses 924 bytes" report comes from: it's literally the size of the
   `.text` (code) and `.data`/`.bss` (variable) sections the linker produced.
4. **Objcopy** converts the linked ELF file into a raw binary image
   (`.hex` for AVR, `.bin` for ESP32) — the exact byte stream that will sit in
   flash.
5. **Upload tool** (`avrdude` for Uno, `esptool.py` for ESP32) opens the
   serial port, and — this is the mechanism behind the BOOT-button warning
   above — talks a bootloader protocol to a tiny piece of firmware already
   burned into the chip. On Uno, the bootloader watches for a reset pulse
   asserted by the IDE toggling the DTR serial line through a capacitor
   ("auto-reset"); on many ESP32 boards, DTR/RTS aren't wired the same way, so
   the chip has to be forced into its ROM bootloader manually by holding BOOT
   (pulls the `GPIO0` strapping pin low) while EN (reset) is toggled — which
   is exactly what holding BOOT during `Connecting....` does.
6. The bootloader then writes your binary into flash at a fixed address, and
   on the next reset the chip's boot ROM jumps to it and your `setup()`/`loop()`
   loop begins.

None of this happens on your machine once uploaded — flash is
non-volatile (it keeps its contents with power off, unlike RAM), which is why
the program "runs there forever, restarting every time the board gets
power": the reset vector always points at the same flashed code.

## Cheat sheet

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

## Exercise

In Wokwi, start from the **ESP32** template and get Blink running with the
onboard LED. Then modify it: make the LED flash **SOS** in Morse code — three
short blinks (150 ms on/off), three long (500 ms), three short, then a 2-second
pause before repeating. Use `Serial.begin(115200);` in `setup()` and
`Serial.println("SOS");` at the start of each cycle, and confirm the message
appears in the serial monitor once per cycle. (Hint: writing a helper function
`void blink(int onMs)` will keep `loop()` readable — a preview of module 2.)
