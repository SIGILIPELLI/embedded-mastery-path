# 04 · Analog I/O & PWM

Digital pins know two values; the real world is continuous. This module
covers both directions of the analog boundary: **reading** continuous
voltages with the ADC (`analogRead`, e.g. a potentiometer or light sensor),
and **producing** apparently-continuous output with PWM (`analogWrite`, e.g.
dimming an LED) — plus the ESP32-specific details that differ from classic
Arduino.

## Reading analog inputs: the ADC

An **ADC** (analog-to-digital converter) measures the voltage on a pin and
returns it as a number. Resolution determines the scale:

- **Arduino Uno**: 10-bit ADC → `analogRead` returns **0–1023** for 0–5 V.
- **ESP32**: 12-bit ADC → `analogRead` returns **0–4095** for 0–3.3 V.

```cpp
// Read a potentiometer and print voltage — works on Uno and ESP32
const uint8_t POT_PIN = 34;        // ESP32: use an ADC-capable pin like 32-36; Uno: A0

void setup() {
  Serial.begin(115200);
}

void loop() {
  int raw = analogRead(POT_PIN);   // 0..4095 on ESP32, 0..1023 on Uno

  // Convert to a voltage (ESP32: 3.3V full scale, 4095 steps)
  float volts = raw * 3.3 / 4095.0;

  Serial.print(raw);
  Serial.print(" -> ");
  Serial.print(volts, 2);
  Serial.println(" V");
  delay(200);
}
```

**Wiring (Wokwi):** add a potentiometer; outer legs → 3V3 and GND, middle leg
(wiper) → GPIO 34. Drag the knob while the sketch runs and watch the numbers
track it.

The `map()` function rescales a reading to any range you actually want:

```cpp
int raw = analogRead(POT_PIN);              // 0..4095
int percent = map(raw, 0, 4095, 0, 100);    // 0..100
int level   = map(raw, 0, 4095, 0, 255);    // 0..255 — ready for PWM below
```

!!! note "ESP32 ADC quirks worth knowing"
    Only some ESP32 pins are ADC-capable (GPIO 32–39 are the safe choices —
    ADC2 pins like 25–27 stop working while WiFi is on). The ESP32's ADC is
    also famously non-linear near the extremes: fine for a knob or light
    sensor, not for precision measurement. Real precision work uses an
    external I2C ADC — a Level 2 topic.

## Producing analog-ish output: PWM

Most pins can't output anything between 0 V and full voltage. **PWM** (pulse
width modulation) fakes it: the pin switches HIGH/LOW hundreds or thousands
of times per second, and the fraction of time spent HIGH — the **duty
cycle** — sets the average power. An LED at 25% duty looks dim; a motor at
50% duty runs at roughly half speed. It's not a true voltage, but for LEDs,
motors, and buzzers, average power is what matters.

### Classic Arduino: `analogWrite`

```cpp
// Breathe an LED: fade up and down forever (Uno: use a PWM pin marked ~, e.g. 5)
const uint8_t LED_PIN = 5;

void setup() {}

void loop() {
  for (int duty = 0; duty <= 255; duty++) {     // fade in
    analogWrite(LED_PIN, duty);                 // 0 = off .. 255 = fully on
    delay(4);
  }
  for (int duty = 255; duty >= 0; duty--) {     // fade out
    analogWrite(LED_PIN, duty);
    delay(4);
  }
}
```

On the Uno, only pins marked `~` (3, 5, 6, 9, 10, 11) support PWM, and the
duty range is fixed at 0–255.

### ESP32: LEDC

The ESP32's PWM hardware is called **LEDC** — 16 channels, any output pin,
configurable frequency and resolution. Since ESP32 Arduino core 3.x,
`analogWrite` works there too, but the native API gives you control:

```cpp
const uint8_t LED_PIN = 5;
const uint32_t PWM_FREQ = 5000;    // 5 kHz — flicker-free for LEDs
const uint8_t PWM_RES = 8;         // 8-bit duty: 0..255

void setup() {
  ledcAttach(LED_PIN, PWM_FREQ, PWM_RES);   // core 3.x API
}

void loop() {
  for (int duty = 0; duty <= 255; duty++) {
    ledcWrite(LED_PIN, duty);
    delay(4);
  }
  for (int duty = 255; duty >= 0; duty--) {
    ledcWrite(LED_PIN, duty);
    delay(4);
  }
}
```

Why frequency matters: 5 kHz is invisible for LEDs; motors often want
~20 kHz (above human hearing, so the windings don't audibly whine); servo
signals are a special 50 Hz case with their own library. Resolution trades
against frequency — 8 bits at 5 kHz is the standard LED setup.

## Putting both together: a knob-controlled dimmer

```cpp
// ESP32: potentiometer on GPIO 34 dims LED on GPIO 5
const uint8_t POT_PIN = 34;
const uint8_t LED_PIN = 5;

void setup() {
  Serial.begin(115200);
  ledcAttach(LED_PIN, 5000, 8);
}

void loop() {
  int raw = analogRead(POT_PIN);            // 0..4095
  int duty = map(raw, 0, 4095, 0, 255);     // rescale to PWM range
  ledcWrite(LED_PIN, duty);

  Serial.print(raw);
  Serial.print(" -> duty ");
  Serial.println(duty);
  delay(50);
}
```

Build it in Wokwi (potentiometer + LED + resistor as wired above) and you
have a working dimmer — the complete sense → compute → act pattern that
every embedded system is built on, in 20 lines.

## Cheat sheet

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

## Exercise

In Wokwi (ESP32), build a "night light": an LDR (photoresistor) on GPIO 34
and an LED on GPIO 5. Read the light level; when it drops below a threshold,
fade the LED **up smoothly** over one second, and when light returns, fade it
back down — no abrupt jumps. Print raw ADC value and current duty over serial
at 5 Hz. Bonus: make the threshold adjustable with a potentiometer on a
second ADC pin, and add ~100 counts of hysteresis so the LED doesn't flicker
when the light level sits exactly at the threshold.
