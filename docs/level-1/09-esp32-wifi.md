# 09 · ESP32 WiFi Intro

The ESP32's superpower is a full WiFi radio on the same chip as your code —
this is the moment your device becomes an *IoT* device. This module covers
the three foundational moves: **joining a network**, **fetching data from
the internet** (HTTP GET), and **serving a web page from the board** so any
browser on the network can see your sensors. All of it runs in
[Wokwi](https://wokwi.com/), which simulates a WiFi network complete with
internet access.

!!! tip "WiFi in the simulator"
    Wokwi's virtual ESP32 sees an open access point named **`Wokwi-GUEST`**
    (empty password). Real board? Just substitute your own SSID/password —
    everything else is identical. Note: ESP32 WiFi is 2.4 GHz only; 5 GHz
    networks are invisible to it — a top-3 real-hardware gotcha.

## Connecting to WiFi

```cpp
#include <WiFi.h>

const char *WIFI_SSID = "Wokwi-GUEST";   // your SSID on real hardware
const char *WIFI_PASS = "";              // your password on real hardware

void setup() {
  Serial.begin(115200);

  WiFi.mode(WIFI_STA);                   // station mode: join a network
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  Serial.print("Connecting");
  while (WiFi.status() != WL_CONNECTED) {
    delay(250);
    Serial.print(".");
  }
  Serial.println();
  Serial.print("Connected, IP: ");
  Serial.println(WiFi.localIP());        // the board's address on the network
  Serial.print("Signal: ");
  Serial.print(WiFi.RSSI());             // dBm; > -70 is decent
  Serial.println(" dBm");
}

void loop() {}
```

**Station mode (STA)** means the ESP32 joins an existing network like a
laptop would. (It can also *be* an access point — `WIFI_AP` — handy for
first-time device setup; a Level 2 topic.) The `while` loop blocks on
purpose here — there's nothing useful to do before the network is up. The
capstone shows the polite non-blocking version.

WiFi drops happen; production code watches for them:

```cpp
void loop() {
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("WiFi lost, reconnecting...");
    WiFi.reconnect();
    delay(1000);
    return;
  }
  // ... normal work ...
}
```

## Making an HTTP GET request

With a connection up, `HTTPClient` makes web requests look almost like
desktop code:

```cpp
#include <WiFi.h>
#include <HTTPClient.h>

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_STA);
  WiFi.begin("Wokwi-GUEST", "");
  while (WiFi.status() != WL_CONNECTED) delay(250);
  Serial.println("Connected.");

  HTTPClient http;
  // Open time API — returns a small JSON document
  http.begin("http://worldtimeapi.org/api/timezone/Etc/UTC");
  int code = http.GET();                 // performs the request

  if (code == HTTP_CODE_OK) {            // 200
    String body = http.getString();      // response body
    Serial.println(body);                // {"utc_datetime":"2026-07-21T09:00:00Z",...}
  } else {
    Serial.printf("HTTP error: %d\n", code);   // negative = connection failure
  }
  http.end();                            // free the connection — don't forget
}

void loop() {}
```

Always check the status code: `200` is success, `-1` means the connection
itself failed, and anything else is the server talking back. Real projects
parse the JSON with the **ArduinoJson** library rather than string-hacking —
worth exploring after this module. (HTTPS works too via
`WiFiClientSecure`; plain HTTP keeps today simple.)

## Serving a web page from the board

Flip the roles: the ESP32 listens on port 80 and *answers* browsers. The
built-in `WebServer` class maps URL paths to handler functions:

```cpp
#include <WiFi.h>
#include <WebServer.h>

WebServer server(80);                    // standard HTTP port

uint32_t hits = 0;

void handleRoot() {
  hits++;
  char page[400];
  snprintf(page, sizeof(page),
    "<!DOCTYPE html><html><head><title>ESP32</title>"
    "<meta http-equiv='refresh' content='5'></head>"       // auto-reload every 5 s
    "<body style='font-family:sans-serif'>"
    "<h1>Hello from ESP32</h1>"
    "<p>Uptime: %lu s</p><p>Page hits: %lu</p>"
    "<p>Free heap: %u bytes</p>"
    "</body></html>",
    millis() / 1000, (unsigned long)hits, ESP.getFreeHeap());
  server.send(200, "text/html", page);   // status, content type, body
}

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_STA);
  WiFi.begin("Wokwi-GUEST", "");
  while (WiFi.status() != WL_CONNECTED) delay(250);

  Serial.print("Open http://");
  Serial.println(WiFi.localIP());

  server.on("/", handleRoot);            // route: GET / → handleRoot()
  server.on("/api", []() {               // a JSON "API endpoint", lambda style
    char json[80];
    snprintf(json, sizeof(json), "{\"uptime_s\":%lu,\"heap\":%u}",
             millis() / 1000, ESP.getFreeHeap());
    server.send(200, "application/json", json);
  });
  server.begin();
}

void loop() {
  server.handleClient();                 // MUST run often — answers pending requests
}
```

`server.handleClient()` is the whole trick: the server only processes
requests when you call it, which is why it sits in a fast-spinning,
non-blocking `loop()` — your module 7 discipline is what keeps a web server
responsive. In Wokwi, the serial monitor prints the IP and Wokwi offers a
way to open the simulated device's web server right from the browser (via
its IoT gateway); on real hardware, any phone or laptop on the same network
just visits `http://<the-ip>/`.

Serving `text/html` for humans and `application/json` for programs — that
tiny `/api` route is the seed of every device dashboard and REST-controlled
gadget you'll build.

## Cheat sheet

| Concept | Detail |
|---------|--------|
| `WiFi.mode(WIFI_STA)` + `WiFi.begin(ssid, pass)` | Join a network as a client |
| `WiFi.status() == WL_CONNECTED` | Poll until (and monitor after) connection |
| `WiFi.localIP()` / `WiFi.RSSI()` | Board's IP / signal strength (dBm) |
| Wokwi network | SSID `Wokwi-GUEST`, empty password, has internet |
| 2.4 GHz only | ESP32 cannot see 5 GHz networks |
| `HTTPClient` GET | `begin(url)` → `GET()` → check code → `getString()` → `end()` |
| Status codes | 200 OK; negative = connection failed |
| `WebServer server(80)` | HTTP server on the board |
| `server.on(path, handler)` + `server.begin()` | Routing setup |
| `server.handleClient()` in `loop()` | Actually serves requests — keep loop non-blocking |

## Exercise

Combine this module with module 4: an ESP32 with a potentiometer ("sensor")
and an LED. Serve a page at `/` showing the current reading (auto-refresh
every 2 s), a JSON endpoint at `/data` returning `{"raw":...,"percent":...}`,
and two control routes `/led/on` and `/led/off` that switch the LED and
redirect back to `/` (`server.sendHeader("Location", "/");
server.send(303);`). Everything must stay responsive: the page and the LED
routes should react instantly even while you twist the knob. Verify all
three routes from the browser, and confirm in the serial monitor that free
heap stays stable across many page loads.
