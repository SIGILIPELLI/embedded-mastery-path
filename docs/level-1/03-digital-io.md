# 03 · Digital I/O

GPIO — *general-purpose input/output* — is the foundation of everything a
microcontroller does. Each pin can be configured as an **output** (the chip
drives it to 0 V or to the supply voltage) or an **input** (the chip reads
whether an external voltage is high or low). This module covers the three
core functions (`pinMode`, `digitalWrite`, `digitalRead`), how to wire LEDs
and buttons correctly, and debouncing — the first genuinely *embedded* problem
you'll solve.

All circuits here can be built in the [Wokwi](https://wokwi.com/) diagram
pane: click **+**, add an LED, a resistor, and a pushbutton, and drag wires
between pin holes exactly as described.

## Outputs: driving an LED

```cpp
const uint8_t LED_PIN = 5;

void setup() {
  pinMode(LED_PIN, OUTPUT);      // configure once
}

void loop() {
  digitalWrite(LED_PIN, HIGH);   // pin outputs 5V (Uno) / 3.3V (ESP32)
  delay(250);
  digitalWrite(LED_PIN, LOW);    // pin outputs 0V
  delay(250);
}
```

**Wiring:** board pin 5 → resistor (220 Ω) → LED anode (long leg, bent-leg
side in Wokwi); LED cathode → GND.

!!! warning "Always use a current-limiting resistor"
    An LED is not a light bulb — it will draw as much current as the supply
    allows and burn out (possibly damaging the pin, which can only source
    ~20–40 mA). A 220 Ω–330 Ω resistor in series limits the current to a safe
    few milliamps. Wokwi is forgiving about this; real hardware is not, so
    build the habit in the simulator.

`HIGH` and `LOW` are just `1` and `0`. A useful idiom for toggling:

```cpp
digitalWrite(LED_PIN, !digitalRead(LED_PIN));   // flip current state
```

## Inputs: reading a button

A pushbutton connects two points when pressed. The subtlety is what the input
pin sees when the button is **not** pressed: if it's connected to nothing,
the pin *floats*, picking up electrical noise and reading random values. Every
digital input needs a defined idle state, provided by a **pull-up** or
**pull-down** resistor.

The standard solution costs no parts — microcontrollers have built-in pull-up
resistors you can enable in software:

```cpp
const uint8_t BUTTON_PIN = 4;
const uint8_t LED_PIN = 5;

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);  // internal resistor holds pin HIGH when idle
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  // With INPUT_PULLUP the logic is inverted:
  //   not pressed → HIGH, pressed → LOW
  if (digitalRead(BUTTON_PIN) == LOW) {
    digitalWrite(LED_PIN, HIGH);
  } else {
    digitalWrite(LED_PIN, LOW);
  }
}
```

**Wiring:** one side of the button → pin 4, the other side → GND. That's all —
the internal pull-up replaces an external resistor.

| Configuration | Idle reads | Pressed reads | Wiring |
|---------------|-----------|---------------|--------|
| `INPUT_PULLUP` (most common) | `HIGH` | `LOW` | button between pin and **GND** |
| `INPUT` + external pull-down | `LOW` | `HIGH` | button between pin and **VCC**, 10 kΩ resistor pin→GND |
| `INPUT`, nothing attached | random noise | — | never do this |

The inverted logic of `INPUT_PULLUP` (`LOW` = pressed) trips up every
beginner once. Hide it behind a well-named helper:

```cpp
bool buttonPressed() {
  return digitalRead(BUTTON_PIN) == LOW;
}
```

## Debouncing

Mechanical button contacts don't close cleanly — they *bounce*, making and
breaking contact for a few milliseconds. The chip is fast enough to see every
bounce, so naive "count the presses" code counts one press as 5–10:

```cpp
// BUGGY: counts bounces, not presses
if (buttonPressed()) {
  pressCount++;
}
```

The fix: after seeing a change, ignore further changes until the reading has
been stable for ~20–50 ms. Here is the classic non-blocking debounce, which
also demonstrates **edge detection** (acting once per press, not continuously
while held):

```cpp
const uint8_t BUTTON_PIN = 4;
const uint8_t LED_PIN = 5;
const uint32_t DEBOUNCE_MS = 30;

bool ledOn = false;
int lastReading = HIGH;          // raw reading last time we looked
int stableState = HIGH;          // debounced, "official" button state
uint32_t lastChangeMs = 0;

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int reading = digitalRead(BUTTON_PIN);

  if (reading != lastReading) {
    lastChangeMs = millis();     // signal changed — restart the stability timer
    lastReading = reading;
  }

  if (millis() - lastChangeMs > DEBOUNCE_MS && reading != stableState) {
    stableState = reading;       // reading has been stable long enough — accept it
    if (stableState == LOW) {    // falling edge = the moment of the press
      ledOn = !ledOn;            // toggle the LED once per press
      digitalWrite(LED_PIN, ledOn);
    }
  }
}
```

Press the button: the LED toggles on. Press again: off. Note there is no
`delay()` anywhere — `loop()` spins freely, which matters as soon as a sketch
does more than one thing (module 7 builds this into a general technique).

Wokwi's virtual button bounces realistically if you enable it: click the
button, and in its properties set `"bounce": "1"` — then try the buggy counter
version and watch it miscount, exactly like real hardware.

## Cheat sheet

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

## Exercise

Build a two-button counter in Wokwi (ESP32 or Uno): one button increments a
counter, the other resets it to zero, and the count is shown by blinking the
LED that many times (e.g. count 3 → three quick blinks, pause, repeat). Both
buttons must be debounced and edge-detected — holding a button must not keep
incrementing. Print the count over serial on every change. Then enable
`"bounce": "1"` on both virtual buttons and verify your counter still counts
correctly.
