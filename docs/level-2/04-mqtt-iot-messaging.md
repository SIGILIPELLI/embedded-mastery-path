# MQTT & IoT Messaging

Module 1-09 got an ESP32 onto WiFi and had it serve a web page — a device
that answers when asked. That model breaks down the moment you have fifty
devices behind a home router, on flaky links, that need to *push* readings
the instant something changes. **MQTT** is what the IoT world settled on for
that shape of problem: a tiny publish/subscribe protocol over a single
long-lived TCP connection, originally designed for satellite links with
terrible bandwidth. This module covers the model, then builds a real
ESP-IDF client with the `esp-mqtt` component — the same one module 2-10's
project leans on.

## Publish/subscribe, and why it fits devices

HTTP is request/response: the client must know the server's address, must
initiate, and gets one answer per question. MQTT inverts it. Every device
opens **one outbound TCP connection** to a **broker** (Mosquitto, EMQX, AWS
IoT Core, HiveMQ) and keeps it open. After that:

- A device **publishes** a message to a **topic** — a slash-separated string.
- Any client **subscribed** to a matching topic receives it.
- Publishers and subscribers never learn about each other. The broker is the
  only address anyone needs.

That decoupling is the whole point. A sensor doesn't care whether one
dashboard, three dashboards, or nothing at all is listening. And because the
connection is outbound and persistent, a device behind a router with no port
forwarding is fully reachable — the broker pushes down the socket the device
already opened.

## Topics and wildcards

Topics are hierarchical, case-sensitive, and created implicitly — there is
no "declare topic" step:

```
home/livingroom/temperature
home/livingroom/humidity
factory/line3/press02/vibration
```

Subscribers may use two wildcards:

| Wildcard | Matches | Example | Result |
|----------|---------|---------|--------|
| `+` | exactly one level | `home/+/temperature` | matches `home/kitchen/temperature`, **not** `home/a/b/temperature` |
| `#` | all remaining levels (must be last) | `home/#` | everything under `home/` |

Design topics so the specific parts go deep and wildcards stay useful. A
workable fleet convention:

```
<site>/<device-id>/<measurement>          # telemetry, device → broker
<site>/<device-id>/cmd/<command>          # commands, broker → device
<site>/<device-id>/status                 # online/offline, retained
```

A device subscribes to `<site>/<its-own-id>/cmd/#` and nothing else; a
dashboard subscribes to `<site>/+/temperature`. Avoid a leading `/` (it
creates a nameless first level) and never put anything that varies at
runtime — a timestamp, a sequence number — into the topic, or the broker
accumulates millions of one-shot topics.

## QoS, retain, and the last will

Three flags carry most of MQTT's reliability semantics.

**QoS (Quality of Service)** is per message, negotiated down to the weaker
of publisher and subscriber:

| QoS | Guarantee | Cost | Use for |
|-----|-----------|------|---------|
| 0 | at most once — fire and forget | 1 packet | high-rate telemetry where the next sample is seconds away |
| 1 | at least once — retried until ACKed, **duplicates possible** | 2 packets | commands, alarms |
| 2 | exactly once — four-way handshake | 4 packets | billing-grade events; rarely worth it |

QoS 1 is the honest default for anything that matters. Duplicates are
*expected* there — so make command handlers idempotent ("set relay to on")
rather than incremental ("toggle relay").

**Retain** tells the broker to store the last retained message per topic and
hand it to every new subscriber immediately. Perfect for state (`status`,
config, last known reading), wrong for events. A dashboard that connects at
3 a.m. sees the last temperature instantly instead of waiting for the next
sample.

**LWT (Last Will and Testament)** is a message the *device* registers at
connect time, which the *broker* publishes if the connection drops without a
clean disconnect. Combine it with retain and you get reliable presence:
publish `"online"` retained on connect, register an LWT of `"offline"`
retained on the same topic, and the topic reflects truth even when the
device loses power mid-sentence.

## An ESP-IDF MQTT client

`esp-mqtt` is event-driven: you register one handler and the client's own
task calls you back. Nothing here blocks `app_main()`.

```c
#include "esp_log.h"
#include "esp_event.h"
#include "mqtt_client.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

static const char *TAG = "mqtt";
static esp_mqtt_client_handle_t s_client;

#define DEVICE_ID  "esp32-01"
#define TOPIC_TEMP "home/" DEVICE_ID "/temperature"
#define TOPIC_STAT "home/" DEVICE_ID "/status"
#define TOPIC_CMD  "home/" DEVICE_ID "/cmd/#"

static void mqtt_event_handler(void *handler_args, esp_event_base_t base,
                               int32_t event_id, void *event_data)
{
    esp_mqtt_event_handle_t event = event_data;

    switch ((esp_mqtt_event_id_t)event_id) {
    case MQTT_EVENT_CONNECTED:
        ESP_LOGI(TAG, "connected to broker");
        /* retained "online" — pairs with the LWT registered below */
        esp_mqtt_client_publish(event->client, TOPIC_STAT, "online", 0, 1, 1);
        esp_mqtt_client_subscribe(event->client, TOPIC_CMD, 1);
        break;

    case MQTT_EVENT_DISCONNECTED:
        ESP_LOGW(TAG, "disconnected — the client reconnects on its own");
        break;

    case MQTT_EVENT_DATA:
        /* topic and data are NOT null-terminated — always use the lengths */
        ESP_LOGI(TAG, "rx %.*s = %.*s",
                 event->topic_len, event->topic,
                 event->data_len,  event->data);
        break;

    case MQTT_EVENT_ERROR:
        ESP_LOGE(TAG, "mqtt error");
        break;

    default:
        break;
    }
}

void mqtt_start(void)
{
    esp_mqtt_client_config_t cfg = {
        .broker.address.uri                  = "mqtt://192.168.1.50:1883",
        .credentials.username                = "esp32",
        .credentials.authentication.password = "secret",
        .session.keepalive                   = 30,
        .session.last_will.topic             = TOPIC_STAT,
        .session.last_will.msg               = "offline",
        .session.last_will.qos               = 1,
        .session.last_will.retain            = 1,
    };

    s_client = esp_mqtt_client_init(&cfg);
    ESP_ERROR_CHECK(esp_mqtt_client_register_event(
        s_client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL));
    ESP_ERROR_CHECK(esp_mqtt_client_start(s_client));   /* returns immediately */
}

void publish_task(void *pv)
{
    char payload[32];
    for (;;) {
        float c = 21.5f;                       /* real sensor read goes here */
        int n = snprintf(payload, sizeof(payload), "%.2f", c);
        /* qos 1, retain 1 — the last reading is state, not an event */
        esp_mqtt_client_publish(s_client, TOPIC_TEMP, payload, n, 1, 1);
        vTaskDelay(pdMS_TO_TICKS(10000));
    }
}
```

Declare the dependency in `main/CMakeLists.txt` with
`REQUIRES mqtt esp_event`, bring WiFi up first — `esp_mqtt_client_start()`
makes no progress until the stack has an IP — and watch the result with
`mosquitto_sub -h 192.168.1.50 -t 'home/#' -v`.

!!! warning "`event->topic` is not a C string"
    `esp_mqtt_event_t` gives you `topic`/`topic_len` and `data`/`data_len`
    pointing into the client's receive buffer. They are **not**
    null-terminated, and the buffer is reused after your handler returns.
    `strcmp(event->topic, "...")` reads past the end; stashing the pointer
    for later gives you garbage. Use `%.*s`, `strncmp()` bounded by
    `topic_len`, or `memcpy` the bytes out while you still own them.

## Traps worth knowing

- **The event handler runs on the MQTT client's own task.** Blocking in it —
  a long I2C read, a `vTaskDelay(5000)`, an SD write — stalls the whole
  client, keepalive pings included, and the broker eventually drops you.
  Push work onto a queue (module 2-01) and return.
- **Large payloads arrive split across events.** If a message exceeds the
  client's RX buffer you get several `MQTT_EVENT_DATA` events with
  `current_data_offset` advancing and `total_data_len` giving the whole
  size. Code that assumes one event per message silently truncates JSON.
- **`esp_mqtt_client_publish()` blocks; `esp_mqtt_client_enqueue()` doesn't.**
  Calling `publish()` at QoS 1 from inside the event handler can deadlock
  the client task against itself. From a handler, use `enqueue()`.
- **Retain plus high frequency is a footgun.** Every retained publish is a
  durable write on many brokers. Retain state topics; don't retain a 10 Hz
  stream.
- **`mqtt://` is plaintext**, credentials included. Use `mqtts://` with a CA
  certificate in `.broker.verification.certificate` for anything that leaves
  your LAN.

## Cheat sheet

| Concept | Detail |
|---------|--------|
| Model | Publish/subscribe via a broker; one persistent outbound TCP connection |
| Topic | `site/device/measurement` — hierarchical, implicit, no leading `/` |
| `+` / `#` | One level / all remaining levels (must be last) — subscribe side only |
| QoS 0 / 1 / 2 | At most once / at least once (**dupes**) / exactly once |
| Retain | Broker keeps the last message per topic and replays it — for *state* |
| LWT | Broker publishes it on an unclean drop — with retain, gives presence |
| `.broker.address.uri` | `mqtt://host:1883` or `mqtts://host:8883` |
| `esp_mqtt_client_init()` / `_start()` | Non-blocking; a background task owns the connection |
| `esp_mqtt_client_register_event(c, ESP_EVENT_ANY_ID, h, arg)` | One handler, switch on `event_id` |
| `event->topic_len` / `data_len` | Payload is **not** null-terminated — always use lengths |
| `esp_mqtt_client_enqueue()` | Non-blocking publish — the safe call from inside a handler |
| Reconnection | Automatic — don't write your own retry loop |

## Exercise

Build a two-topic device on top of the code above. Publish a simulated
temperature every 10 s to `home/<id>/temperature` at QoS 1 retained, and
subscribe to `home/<id>/cmd/interval` so that publishing `2` with
`mosquitto_pub` changes the sampling period to 2 s at runtime — parse the
payload using `data_len` rather than `atoi(event->data)`, and validate the
range (1–3600) before applying it. Move the actual period change out of the
event handler and into the publishing task via a queue, so nothing blocks
the client task. Then verify presence really works: subscribe to
`home/<id>/status`, pull the ESP32's power, and confirm the broker publishes
`offline` within roughly 1.5× your keepalive — proof the LWT is registered,
not merely configured.
