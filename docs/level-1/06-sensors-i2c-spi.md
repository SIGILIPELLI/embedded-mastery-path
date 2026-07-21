# 06 · Sensors, I2C & SPI

Real devices are built from *parts that talk to each other*: sensors,
displays, memory chips, radios. Rather than dedicate dozens of pins to each
part, microcontrollers use shared **buses** — most commonly **I2C** and
**SPI**. This module explains both conceptually, then puts them to work with
the two parts you'll use in the capstone: a **DHT22** temperature/humidity
sensor and an **SSD1306 OLED display** — both available as virtual parts in
[Wokwi](https://wokwi.com/).

## I2C in a nutshell

**I2C** (inter-integrated circuit) uses just **two shared wires** for any
number of devices:

- **SDA** — serial data (both directions)
- **SCL** — serial clock (the controller ticks it; data is valid on each tick)

Every peripheral has a 7-bit **address** (e.g. the SSD1306 OLED is usually
`0x3C`). The controller (your board) starts each exchange by broadcasting an
address; only the matching device responds. Both lines need pull-up resistors
(usually already on the module boards). Typical speed: 100–400 kHz — slow-ish,
but two pins for a whole network of sensors is a great trade.

Default I2C pins: **A4 (SDA) / A5 (SCL)** on Uno; **GPIO 21 (SDA) / GPIO 22
(SCL)** on ESP32.

The `Wire` library is Arduino's I2C interface. You'll rarely call it directly
— device libraries wrap it — but a bus scanner shows what's really happening
and is the first thing to run when a device "doesn't work":

```cpp
#include <Wire.h>

void setup() {
  Serial.begin(115200);
  Wire.begin();                              // join the bus as controller
  Serial.println("Scanning I2C bus...");
  for (uint8_t addr = 1; addr < 127; addr++) {
    Wire.beginTransmission(addr);            // try to address each device
    if (Wire.endTransmission() == 0) {       // 0 = someone acknowledged
      Serial.print("  found device at 0x");
      Serial.println(addr, HEX);             // e.g. "found device at 0x3C"
    }
  }
  Serial.println("Done.");
}

void loop() {}
```

## SPI in a nutshell

**SPI** (serial peripheral interface) trades pins for speed — four wires,
tens of MHz, used by SD cards, displays, and flash chips:

| Wire | Meaning |
|------|---------|
| SCK | Clock, driven by the controller |
| MOSI | Controller out, peripheral in |
| MISO | Controller in, peripheral out |
| CS/SS | Chip select — one **per device**, pulled LOW to talk to it |

No addresses: selecting a device is electrical (its CS line). Multiple
devices share SCK/MOSI/MISO but each needs its own CS pin.

**Rule of thumb:** low-speed sensors → I2C (fewer pins); high-bandwidth
things like SD cards and big/fast displays → SPI. You'll use SPI hands-on in
Level 2; today's parts are I2C and single-wire.

## Using libraries

Nobody bit-bangs display protocols by hand — you install a **library** that
speaks the device's protocol and gives you a friendly API.

- **Arduino IDE:** Sketch → Include Library → **Manage Libraries**, search,
  install.
- **Wokwi:** open the **Library Manager** tab (left of the code editor) →
  **+** → search — or just add the part; Wokwi usually prompts for the
  matching library.

For this module install: **DHT sensor library** (Adafruit) — its dependency
**Adafruit Unified Sensor** installs alongside — plus **Adafruit SSD1306**
and **Adafruit GFX Library**.

## Reading a DHT22 (temperature & humidity)

The DHT22 isn't I2C — it uses its own single-wire protocol, which is exactly
why it's a good library lesson: the library hides some genuinely fiddly
microsecond-level timing.

**Wiring (Wokwi part "DHT22"):** VCC → 3V3/5V, GND → GND, SDA/DATA → GPIO 15.

```cpp
#include "DHT.h"

const uint8_t DHT_PIN = 15;
DHT dht(DHT_PIN, DHT22);           // create driver object: pin + sensor type

void setup() {
  Serial.begin(115200);
  dht.begin();
}

void loop() {
  delay(2000);                     // DHT22 max sample rate: once per 2 s

  float h = dht.readHumidity();
  float t = dht.readTemperature();          // Celsius

  if (isnan(h) || isnan(t)) {               // reads can fail — always check!
    Serial.println(F("DHT read failed"));
    return;
  }

  Serial.print(F("Temp: "));
  Serial.print(t, 1);
  Serial.print(F(" C  Humidity: "));
  Serial.print(h, 1);
  Serial.println(F(" %"));
}
```

The `isnan()` check is the important habit: sensors are physical devices on
wires — reads fail, and robust firmware checks every one. In Wokwi, click the
DHT22 part while running and drag the temperature/humidity sliders; your
serial output follows.

## Driving an SSD1306 OLED over I2C

**Wiring (Wokwi part "SSD1306"):** VCC → 3V3, GND → GND, SCL → GPIO 22,
SDA → GPIO 21 (address `0x3C` — your scanner from earlier will find it).

```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

Adafruit_SSD1306 display(128, 64, &Wire, -1);   // width, height, bus, no reset pin

void setup() {
  Serial.begin(115200);
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("SSD1306 init failed"));
    while (true) delay(1000);                   // no display — halt visibly
  }
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(2);
  display.setCursor(0, 0);
  display.println(F("Hello,"));
  display.println(F("embedded!"));
  display.display();                            // nothing appears until this!
}

void loop() {}
```

The library draws into a RAM **framebuffer**; `display.display()` ships the
whole buffer over I2C to the panel. Forgetting that call — and staring at a
blank screen — is a rite of passage.

### Sensor + display together

```cpp
void loop() {
  delay(2000);
  float t = dht.readTemperature();
  float h = dht.readHumidity();
  if (isnan(t) || isnan(h)) return;

  display.clearDisplay();
  display.setTextSize(2);
  display.setCursor(0, 0);
  display.print(t, 1);
  display.println(F(" C"));
  display.print(h, 1);
  display.println(F(" %"));
  display.display();
}
```

That's a working thermometer with a screen — the heart of the capstone, in
which the `delay(2000)` will be replaced by proper non-blocking scheduling
(module 7).

## Cheat sheet

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

## Exercise

Build the thermometer above in Wokwi (ESP32 + DHT22 + SSD1306), then upgrade
it: show temperature and humidity, plus the **minimum and maximum
temperature seen since boot** on a smaller font line (`setTextSize(1)`), and
a boot counter of failed DHT reads. Verify min/max by dragging the DHT22
slider around, and verify the failure counter stays at 0 in normal operation.
Bonus: run the I2C scanner sketch first and confirm the OLED's address, then
deliberately pass the wrong address to `display.begin()` and observe how the
failure path behaves.
