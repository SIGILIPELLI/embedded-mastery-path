# 10 · Capstone — Environment Monitor

Time to combine every module into one real device: an **environment
monitor** that measures temperature and humidity (DHT22), shows live
readings on an OLED, raises an LED alarm when temperature crosses a
threshold, and takes serial commands to operate it — all scheduled
cooperatively with `millis()`, no blocking anywhere. It builds entirely in
[Wokwi](https://wokwi.com/) on a simulated ESP32, and runs unchanged on a
real board.

**Skills exercised:** sketch structure (1), types/`char` buffers (2),
digital output (3), I2C + libraries (6), serial commands (5), `millis()`
scheduling (7), plus an optional WiFi extension (9).

## The build

Parts (all in Wokwi's + menu): **ESP32 DevKit**, **DHT22**, **SSD1306
OLED**, **LED**, **resistor (220 Ω)**.

| Connection | From | To |
|------------|------|----|
| DHT22 data | DHT22 SDA | ESP32 GPIO 15 |
| DHT22 power | VCC / GND | 3V3 / GND |
| OLED I2C | SCL / SDA | GPIO 22 / GPIO 21 |
| OLED power | VCC / GND | 3V3 / GND |
| Alarm LED | GPIO 5 | 220 Ω resistor → LED anode; cathode → GND |

Equivalent `diagram.json` parts list, if you prefer editing Wokwi's diagram
file directly (the `connections` follow the table above):

```json
"parts": [
  { "type": "board-esp32-devkit-c-v4", "id": "esp" },
  { "type": "wokwi-dht22", "id": "dht1" },
  { "type": "board-ssd1306", "id": "oled1" },
  { "type": "wokwi-led", "id": "led1", "attrs": { "color": "red" } },
  { "type": "wokwi-resistor", "id": "r1", "attrs": { "value": "220" } }
]
```

Libraries (Wokwi Library Manager tab): **DHT sensor library**, **Adafruit
SSD1306**, **Adafruit GFX Library**.

## Design before code

Four cooperative tasks, one shared state:

| Task | Period | Job |
|------|--------|-----|
| Sensor | 2000 ms | Read DHT22 into state; count failures |
| Display | 1000 ms | Redraw OLED from state |
| Alarm | 250 ms | LED solid when temp ≥ threshold; blink if sensor failing |
| Console | every pass | Parse serial commands, reply OK/ERR |

State is a single struct — one owner for every fact, tasks communicate only
through it. This "shared state + periodic tasks" shape is how most
real-world Arduino-class firmware is organized.

## The full sketch

```cpp
// Environment Monitor — Level 1 capstone
// ESP32 + DHT22 (GPIO 15) + SSD1306 OLED (I2C 21/22) + alarm LED (GPIO 5)
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "DHT.h"

// ---- pins & constants ----
const uint8_t DHT_PIN = 15;
const uint8_t LED_PIN = 5;
const uint32_t SENSOR_MS = 2000;
const uint32_t DISPLAY_MS = 1000;
const uint32_t ALARM_MS = 250;

// ---- devices ----
DHT dht(DHT_PIN, DHT22);
Adafruit_SSD1306 display(128, 64, &Wire, -1);

// ---- shared state ----
struct State {
  float tempC = NAN;         // NAN until first good read
  float humidity = NAN;
  float minC = NAN, maxC = NAN;
  float alarmC = 30.0;       // threshold, adjustable over serial
  bool alarmOn = true;       // alarm armed?
  uint16_t failedReads = 0;
  uint32_t goodReads = 0;
};
State st;

// ---- task timers ----
uint32_t lastSensor = 0, lastDisplay = 0, lastAlarm = 0;

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  dht.begin();

  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("FATAL: OLED not found at 0x3C"));
    while (true) { digitalWrite(LED_PIN, !digitalRead(LED_PIN)); delay(100); }
  }
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println(F("Env Monitor v1.0"));
  display.println(F("waiting for sensor..."));
  display.display();

  Serial.println(F("Env Monitor v1.0"));
  Serial.println(F("Commands: status | alarm <degC> | alarm on | alarm off | reset"));
}

void loop() {
  uint32_t now = millis();
  if (now - lastSensor  >= SENSOR_MS)  { lastSensor  = now; taskSensor();  }
  if (now - lastDisplay >= DISPLAY_MS) { lastDisplay = now; taskDisplay(); }
  if (now - lastAlarm   >= ALARM_MS)   { lastAlarm   = now; taskAlarm();   }
  taskConsole();   // every pass — commands answered instantly
}

// ---- task 1: sensor ----
void taskSensor() {
  float t = dht.readTemperature();
  float h = dht.readHumidity();
  if (isnan(t) || isnan(h)) {
    st.failedReads++;
    return;                            // keep last good values on screen
  }
  st.tempC = t;
  st.humidity = h;
  st.goodReads++;
  if (isnan(st.minC) || t < st.minC) st.minC = t;
  if (isnan(st.maxC) || t > st.maxC) st.maxC = t;
}

// ---- task 2: display ----
void taskDisplay() {
  display.clearDisplay();
  display.setCursor(0, 0);

  display.setTextSize(2);
  if (isnan(st.tempC)) {
    display.println(F("--.- C"));
    display.println(F("--.- %"));
  } else {
    display.print(st.tempC, 1); display.println(F(" C"));
    display.print(st.humidity, 1); display.println(F(" %"));
  }

  display.setTextSize(1);
  display.print(F("min ")); display.print(st.minC, 1);
  display.print(F(" max ")); display.println(st.maxC, 1);
  display.print(F("alarm "));
  if (st.alarmOn) { display.print(st.alarmC, 1); display.println(F("C")); }
  else            { display.println(F("off")); }
  display.display();
}

// ---- task 3: alarm LED ----
void taskAlarm() {
  bool sensorDead = st.failedReads > 3 && st.goodReads == 0;
  if (sensorDead) {
    digitalWrite(LED_PIN, !digitalRead(LED_PIN));   // blink = fault
  } else if (st.alarmOn && !isnan(st.tempC) && st.tempC >= st.alarmC) {
    digitalWrite(LED_PIN, HIGH);                    // solid = over temp
  } else {
    digitalWrite(LED_PIN, LOW);
  }
}

// ---- task 4: serial console ----
void taskConsole() {
  if (Serial.available() == 0) return;
  String cmd = Serial.readStringUntil('\n');
  cmd.trim();
  if (cmd.length() == 0) return;

  if (cmd == "status") {
    char buf[96];
    snprintf(buf, sizeof(buf),
             "temp=%.1fC hum=%.1f%% min=%.1f max=%.1f alarm=%s@%.1fC fails=%u up=%lus",
             st.tempC, st.humidity, st.minC, st.maxC,
             st.alarmOn ? "on" : "off", st.alarmC,
             st.failedReads, millis() / 1000);
    Serial.println(buf);
  } else if (cmd == "alarm on") {
    st.alarmOn = true;  Serial.println(F("OK alarm on"));
  } else if (cmd == "alarm off") {
    st.alarmOn = false; Serial.println(F("OK alarm off"));
  } else if (cmd.startsWith("alarm ")) {
    float v = cmd.substring(6).toFloat();
    if (v < -40 || v > 80) { Serial.println(F("ERR range -40..80")); return; }
    st.alarmC = v;
    Serial.print(F("OK alarm at ")); Serial.println(v, 1);
  } else if (cmd == "reset") {
    st.minC = st.maxC = st.tempC;
    st.failedReads = 0;
    Serial.println(F("OK stats reset"));
  } else {
    Serial.print(F("ERR unknown: ")); Serial.println(cmd);
  }
}
```

## Test plan

Verify like an engineer — behavior by behavior:

1. **Boot**: OLED shows the banner, serial prints the command list.
2. **Readings**: click the DHT22, drag sliders — OLED tracks within ~2 s;
   min/max update correctly.
3. **Alarm**: `alarm 25`, drag temperature above 25 °C → LED solid; below →
   off. `alarm off` → LED stays off regardless.
4. **Console under load**: `status` answers instantly, always — the point of
   the non-blocking design.
5. **Endurance**: leave it running; confirm `status` uptime climbs and
   nothing degrades.

## Optional extension: WiFi reporting

Module 9 turns the monitor into a networked instrument. Additions:

```cpp
#include <WiFi.h>
#include <WebServer.h>
WebServer server(80);
```

- In `setup()`: connect to `Wokwi-GUEST`, then register two routes — `/`
  returning an HTML dashboard built with `snprintf` (temperature, humidity,
  min/max, alarm state, auto-refresh every 5 s), and `/json` returning
  `{"temp":24.3,"hum":41.2,"alarm":false}` for programs.
- In `loop()`: one new line — `server.handleClient();`.
- In the handlers: read from `st` — the shared-state design means WiFi is
  *just another task*, ~30 lines, touching nothing else.

That composability is the real lesson of the capstone.

## Cheat sheet — the Level 1 patterns in one table

| Pattern | Where it came from |
|---------|--------------------|
| `setup()` init, task-based `loop()` | Modules 1, 7 |
| Fixed-width types, `char[]` + `snprintf`, `F()` | Module 2 |
| LED + resistor on GPIO, `digitalWrite` | Module 3 |
| I2C device at address (OLED `0x3C`) | Module 6 |
| Check `isnan()` on every sensor read | Module 6 |
| `if (now - last >= period)` scheduling | Module 7 |
| OK/ERR command console with validation | Module 5 |
| Struct as single source of truth | This module |
| Fault signalling (blink = sensor dead) | This module |
| WiFi/web as an add-on task | Module 9 |

## Exercise

Ship v1.1 with three upgrades: (1) a **humidity alarm** (`halarm <pct>`,
`halarm off`) — LED blinks *fast* (100 ms) for humidity alarm vs. solid for
temperature, and decide (in a comment) what should happen when both trigger;
(2) persist `alarmC`, `halarm`, and the on/off flags in **NVS** (module 8) so
settings survive reboot — verify by restarting the simulation; (3) a serial
`log` command that toggles a CSV line (`millis,temp,hum`) every 2 s, suitable
for pasting into a spreadsheet. Run the full test plan again afterwards —
including the "console answers instantly" check — and confirm free heap is
stable over at least five minutes of logging.
