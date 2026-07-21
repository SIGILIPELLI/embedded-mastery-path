# 05 · Serial/UART Communication

A microcontroller has no screen and no keyboard — **serial over USB** is your
window into it. Under the hood this is a **UART** (universal asynchronous
receiver-transmitter): two wires, TX (transmit) and RX (receive), carrying
bytes at an agreed speed (the *baud rate*). You've already used
`Serial.print` for output; this module makes serial a two-way tool — your
debugger, your control console, and your first communication protocol.

## Output: `Serial.begin` and the print family

```cpp
void setup() {
  Serial.begin(115200);      // start UART at 115200 bits/second
  Serial.println("Booted."); // println = print + newline
}

void loop() {
  float tempC = 21.57;
  int raw = 3178;

  Serial.print("raw=");      // print WITHOUT newline — build a line in pieces
  Serial.print(raw);
  Serial.print(" temp=");
  Serial.print(tempC, 1);    // second arg = decimal places → "21.6"
  Serial.println(" C");      // finish the line
  delay(1000);
}
```

The baud rate in `Serial.begin()` and in the serial monitor **must match**,
or you'll see garbage characters. `115200` is the modern default (`9600` in
older tutorials — it works, just slower). Other formatting tricks:

```cpp
Serial.println(255, HEX);    // FF
Serial.println(255, BIN);    // 11111111
Serial.println(millis());    // uptime in ms — the poor man's timestamp
```

In Wokwi the serial monitor pane appears under the diagram automatically; in
the Arduino IDE it's the magnifier icon (set the same baud rate in its
dropdown).

## Serial as a debugging tool

Until you meet real debuggers (Level 2), `Serial.print` **is** your debugger.
Three habits that pay off immediately:

```cpp
// 1. Trace checkpoints — find WHERE it hangs or resets
Serial.println(F("about to init sensor..."));
initSensor();
Serial.println(F("sensor OK"));

// 2. Print VALUES, not just positions — find WHY it misbehaves
Serial.print(F("duty=")); Serial.println(duty);

// 3. Timestamp with millis() — see WHEN things happen and how long they take
uint32_t t0 = millis();
readAllSensors();
Serial.print(F("readAllSensors took "));
Serial.print(millis() - t0);
Serial.println(F(" ms"));
```

!!! tip "Serial Plotter"
    The Arduino IDE's **Serial Plotter** (Tools menu) graphs numeric serial
    output live. Print one number per line (or several separated by spaces)
    and you get an instant oscilloscope-ish view of a sensor — try it with
    module 4's potentiometer sketch.

## Input: reading from the PC

The board can read what you type into the serial monitor. Incoming bytes wait
in a small buffer (64 bytes classically) until you read them:

```cpp
void loop() {
  if (Serial.available() > 0) {        // any bytes waiting?
    int b = Serial.read();             // read ONE byte (-1 if none)
    Serial.print("got byte: ");
    Serial.println((char)b);
  }
}
```

Single characters make fine commands already — `'1'` = LED on, `'0'` = off.
For whole words and arguments, read a full line:

```cpp
void loop() {
  if (Serial.available() > 0) {
    // Read characters until newline (send "Newline" line ending in the monitor)
    String line = Serial.readStringUntil('\n');
    line.trim();                       // strip stray \r and spaces
    handleCommand(line);
  }
}
```

## Parsing simple commands

A command interpreter turns your device into something you can *operate*,
not just observe. This pattern — a name, an optional argument, a dispatch
`if`/`else` chain — scales all the way to the capstone:

```cpp
const uint8_t LED_PIN = 5;
bool ledOn = false;

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  Serial.println(F("Commands: on | off | blink <n> | status"));
}

void loop() {
  if (Serial.available() > 0) {
    String line = Serial.readStringUntil('\n');
    line.trim();
    handleCommand(line);
  }
}

void handleCommand(const String &cmd) {
  if (cmd == "on") {
    ledOn = true;
    digitalWrite(LED_PIN, HIGH);
    Serial.println(F("OK led on"));
  } else if (cmd == "off") {
    ledOn = false;
    digitalWrite(LED_PIN, LOW);
    Serial.println(F("OK led off"));
  } else if (cmd.startsWith("blink ")) {
    int n = cmd.substring(6).toInt();          // parse the argument
    if (n < 1 || n > 20) {
      Serial.println(F("ERR blink count must be 1-20"));
      return;
    }
    for (int i = 0; i < n; i++) {
      digitalWrite(LED_PIN, HIGH); delay(150);
      digitalWrite(LED_PIN, LOW);  delay(150);
    }
    Serial.println(F("OK blinked"));
  } else if (cmd == "status") {
    Serial.print(F("led="));
    Serial.print(ledOn ? F("on") : F("off"));
    Serial.print(F(" uptime_ms="));
    Serial.println(millis());
  } else if (cmd.length() > 0) {
    Serial.print(F("ERR unknown command: "));
    Serial.println(cmd);
  }
}
```

Design details worth copying: every command gets a reply (`OK ...` or
`ERR ...`) so you always know it was heard; arguments are validated; unknown
input produces a helpful error instead of silence. (Yes, this uses `String` —
acceptable for a console fed by a human. The allocation-free `char[]`
version is a good challenge after module 2.)

## Cheat sheet

| Function | Purpose |
|----------|---------|
| `Serial.begin(115200)` | Start serial; baud must match the monitor |
| `Serial.print(x)` / `println(x)` | Write without / with trailing newline |
| `Serial.print(f, 2)` | Float with 2 decimal places |
| `Serial.println(x, HEX)` | Print in hex (also `BIN`, `OCT`) |
| `Serial.available()` | Number of received bytes waiting in the buffer |
| `Serial.read()` | Consume one byte (returns `-1` if none) |
| `Serial.readStringUntil('\n')` | Read a full line (set monitor line ending to Newline) |
| `str.trim()` / `startsWith()` / `substring()` / `toInt()` | Basic parsing toolkit |
| `millis()` in logs | Timestamp events, measure durations |

## Exercise

Extend the command interpreter with a potentiometer on an ADC pin (Wokwi
ESP32 template): add `read` (prints the current raw ADC value once),
`stream <ms>` (prints the value every `<ms>` milliseconds — validate
50–5000), and `stop`. Streaming must not use `delay()` — track the last
print time with `millis()` so the interpreter stays responsive while
streaming (module 7 formalizes this pattern). Verify you can type `stop`
mid-stream and get an immediate `OK`.
