# Project — WiFi Data Logger

Nine modules of Level 2 have each solved one problem in isolation. This one
puts them together into a single device that could plausibly be left
somewhere for a year: it samples a sensor on a schedule, writes every
reading to local flash so nothing is lost when the network isn't there,
streams what it can to an MQTT broker, survives its own bugs, and can
replace its own firmware over the air with an automatic rollback if the new
build turns out to be broken.

The interesting engineering here isn't any single API — you've met them all.
It's the *seams*: what happens when the broker is unreachable, which task is
allowed to block, what gets dropped when something has to be dropped, and
how a bad update undoes itself.

## What you're building

Three tasks, two queues, one rule: **the network is never allowed to stall
the sensor or the disk.**

```
 sampler_task (prio 6)     logger_task (prio 5)        uplink_task (prio 4)
 ┌───────────────┐         ┌──────────────────┐        ┌─────────────────┐
 │ read sensor   │ sample_q│ batch 30 samples │uplink_q│ publish MQTT    │
 │ stamp with    ├────────►│ append LittleFS  ├───────►│ QoS 1, retained │
 │ wall-clock    │  (60)   │ forward, no wait │  (30)  │ drop on backlog │
 └───────────────┘         └──────────────────┘        └─────────────────┘
                                                    ota_task (prio 3, on demand)
```

`sample_q` is deep enough (60 slots) that a slow flash write never costs a
reading. `uplink_q` is deliberately *shallow* and non-blocking on send: if
WiFi is down for an hour, the logger keeps writing to flash at full speed
and the uplink queue simply overflows. Data on flash is the source of truth;
the MQTT stream is a convenience.

## Partitions and configuration

Two app slots for OTA (module 2-05), a filesystem for the log (module 2-07),
and a core dump partition (module 2-09). On a 4 MB module:

```csv
# Name,     Type, SubType,  Offset,   Size
nvs,        data, nvs,      0x9000,   0x4000,
otadata,    data, ota,      0xd000,   0x2000,
phy_init,   data, phy,      0xf000,   0x1000,
coredump,   data, coredump, ,         0x10000,
ota_0,      app,  ota_0,    0x20000,  0x180000,
ota_1,      app,  ota_1,    ,         0x180000,
storage,    data, littlefs, ,         0xC0000,
```

In `menuconfig`: custom partition table CSV, **Bootloader config → Enable
app rollback support**, **Core dump → Data destination → Flash**, and a Task
WDT timeout of 10 s. Set `PROJECT_VER` in your top-level `CMakeLists.txt`
(or a `version.txt`) so `esp_app_get_description()->version` means something.

## Shared types

One header every task agrees on. Note that a sample is a plain value type —
it gets *copied* through the queues, so no task ever holds a pointer into
another task's memory:

```c
/* main/logger.h */
#pragma once
#include <time.h>
#include "freertos/FreeRTOS.h"
#include "freertos/queue.h"

typedef struct {
    time_t  ts;          /* wall clock, seconds since epoch */
    float   celsius;
    float   humidity;
    uint32_t seq;        /* monotonic; makes gaps visible downstream */
} sample_t;

extern QueueHandle_t g_sample_q;   /* sampler -> logger */
extern QueueHandle_t g_uplink_q;   /* logger  -> uplink  */
```

## Task 1 — the sampler

The only task with a hard timing requirement, so it gets the highest
priority and does the least work. `vTaskDelayUntil()` rather than
`vTaskDelay()` keeps the period fixed regardless of how long the read took —
otherwise your "10 second" interval slowly drifts:

```c
static const char *TAG = "sampler";
static uint32_t s_seq;

void sampler_task(void *pv)
{
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));
    TickType_t last = xTaskGetTickCount();

    for (;;) {
        sample_t s = { .ts = time(NULL), .seq = ++s_seq };

        if (sensor_read(&s.celsius, &s.humidity) == ESP_OK) {
            /* short timeout, not portMAX_DELAY: a jammed logger must not
               silently stop the clock on sampling */
            if (xQueueSend(g_sample_q, &s, pdMS_TO_TICKS(50)) != pdTRUE) {
                ESP_LOGW(TAG, "sample_q full, dropped seq=%lu", s.seq);
            }
        } else {
            ESP_LOGW(TAG, "sensor read failed at seq=%lu", s.seq);
        }

        esp_task_wdt_reset();
        vTaskDelayUntil(&last, pdMS_TO_TICKS(g_interval_ms));
    }
}
```

Keeping `seq` monotonic across drops is what makes the drop *visible*: a gap
in sequence numbers on the dashboard is a fact, whereas a missing row is
indistinguishable from a device that was switched off.

## Task 2 — the logger

Batches thirty samples in RAM, then writes them in one `fopen`/`fclose`
cycle (module 2-07's wear arithmetic — thirty separate appends would erase
the same sector thirty times). It also forwards each sample to the uplink,
but never waits to do so:

```c
#define BATCH 30
static const char *TAG = "logger";

void logger_task(void *pv)
{
    sample_t batch[BATCH];
    int n = 0;

    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));

    for (;;) {
        sample_t s;
        if (xQueueReceive(g_sample_q, &s, pdMS_TO_TICKS(5000)) == pdTRUE) {
            batch[n++] = s;
            /* timeout 0 — if the uplink is backed up, drop and move on */
            xQueueSend(g_uplink_q, &s, 0);
        }

        if (n == BATCH) {
            FILE *f = fopen("/data/log.csv", "a");
            if (f) {
                for (int i = 0; i < n; i++) {
                    fprintf(f, "%lld,%lu,%.2f,%.1f\n",
                            (long long)batch[i].ts, batch[i].seq,
                            batch[i].celsius, batch[i].humidity);
                }
                fclose(f);              /* durability happens here */
                ESP_LOGI(TAG, "flushed %d samples", n);
            } else {
                ESP_LOGE(TAG, "open failed: %s", strerror(errno));
            }
            n = 0;
            rotate_if_large("/data/log.csv", 64 * 1024);
        }
        esp_task_wdt_reset();
    }
}
```

The `xQueueReceive` timeout of 5 s is not decoration — it guarantees the
loop reaches `esp_task_wdt_reset()` even when no samples arrive, so a
stalled sensor doesn't look like a hung logger to the watchdog.

!!! warning "Priority inversion is one shared mutex away"
    The moment two tasks touch the filesystem — the logger appending, and a
    future HTTP handler reading the file back — they need a mutex, and that
    mutex must be a real one. `xSemaphoreCreateMutex()` implements priority
    inheritance: while the low-priority reader holds the lock, it is
    temporarily boosted so a mid-priority task (the uplink, say) can't
    preempt it and leave the high-priority logger waiting indefinitely.
    `xSemaphoreCreateBinary()` looks identical at the call site and provides
    none of that. In a design like this one, the symptom of getting it wrong
    is a logger that misses its flush deadline only when WiFi is busy —
    intermittent, load-dependent, and miserable to reproduce.

## Task 3 — the uplink

Publishes each sample as JSON. `esp_mqtt_client_enqueue()` rather than
`esp_mqtt_client_publish()` so this task never blocks on the client's
internal state, and QoS 1 with retain so a dashboard connecting later sees
the latest reading immediately (module 2-04):

```c
#define TOPIC_DATA "site1/" DEVICE_ID "/reading"

void uplink_task(void *pv)
{
    sample_t s;
    char json[128];

    for (;;) {
        if (xQueueReceive(g_uplink_q, &s, portMAX_DELAY) != pdTRUE) continue;

        int len = snprintf(json, sizeof(json),
                           "{\"ts\":%lld,\"seq\":%lu,\"c\":%.2f,\"rh\":%.1f}",
                           (long long)s.ts, s.seq, s.celsius, s.humidity);

        /* enqueue: non-blocking, safe to call from anywhere. Returns the
           message id, or -1 if the outbox is full — which is fine here. */
        if (esp_mqtt_client_enqueue(g_mqtt, TOPIC_DATA, json, len, 1, 1, true) < 0) {
            ESP_LOGW(TAG, "outbox full, seq=%lu not queued", s.seq);
        }
    }
}
```

This task blocks on `portMAX_DELAY` and deliberately has **no** watchdog
subscription: it is allowed to sit idle for hours when nothing is being
sampled, and subscribing it would make a quiet night look like a hang.

## Time, before anything else

A log full of `esp_timer_get_time()` microseconds is nearly useless once the
device has rebooted twice. Get real time from SNTP after WiFi is up, before
the sampler starts:

```c
#include "esp_netif_sntp.h"

esp_err_t time_sync(void)
{
    esp_sntp_config_t cfg = ESP_NETIF_SNTP_DEFAULT_CONFIG("pool.ntp.org");
    ESP_ERROR_CHECK(esp_netif_sntp_init(&cfg));

    esp_err_t err = esp_netif_sntp_sync_wait(pdMS_TO_TICKS(15000));
    if (err != ESP_OK) {
        ESP_LOGW(TAG, "no NTP — timestamps will be relative to boot");
    }
    setenv("TZ", "UTC0", 1);   /* log in UTC; convert for humans downstream */
    tzset();
    return err;
}
```

Log in UTC. Local time in a stored log is a bug waiting for a daylight
saving transition to duplicate an hour of readings.

## Self-updating, safely

OTA is triggered by a command on `site1/<id>/cmd/ota`, handled the way
module 2-04 insists: the MQTT event handler does not do the work, it just
signals a dedicated task with a big stack. And the *new* firmware must earn
its keep before the bootloader accepts it:

```c
static bool self_test_ok(void)
{
    /* meaningful health: can we still be reached to receive the NEXT fix? */
    return wifi_has_ip() && mqtt_is_connected() && littlefs_mounted();
}

void app_main(void)
{
    log_boot_reason();            /* module 2-08 */
    report_last_crash();          /* module 2-09 */
    storage_init();
    fs_mount();

    wifi_start();
    time_sync();
    mqtt_start();

    g_sample_q = xQueueCreate(60, sizeof(sample_t));
    g_uplink_q = xQueueCreate(30, sizeof(sample_t));
    configASSERT(g_sample_q && g_uplink_q);

    xTaskCreatePinnedToCore(sampler_task, "sampler", 3072, NULL, 6, NULL, 1);
    xTaskCreatePinnedToCore(logger_task,  "logger",  4096, NULL, 5, NULL, 1);
    xTaskCreatePinnedToCore(uplink_task,  "uplink",  4096, NULL, 4, NULL, 0);

    /* rollback probation — only now, once everything really works */
    const esp_partition_t *running = esp_ota_get_running_partition();
    esp_ota_img_states_t state;
    if (esp_ota_get_state_partition(running, &state) == ESP_OK &&
        state == ESP_OTA_IMG_PENDING_VERIFY) {

        for (int i = 0; i < 60 && !self_test_ok(); i++) {
            vTaskDelay(pdMS_TO_TICKS(1000));
        }
        if (self_test_ok()) {
            esp_ota_mark_app_valid_cancel_rollback();
            ESP_LOGI(TAG, "new image confirmed good");
        } else {
            ESP_LOGE(TAG, "self-test failed — rolling back");
            esp_ota_mark_app_invalid_rollback_and_reboot();
        }
    }
}
```

!!! warning "Stack sizes here are starting points, not answers"
    `3072` and `4096` bytes (ESP-IDF counts stack in **bytes**, not words —
    module 2-01) are plausible for these tasks, but `logger_task` alone puts
    a 30-element `sample_t` array on its stack, and `snprintf` with floats
    pulls in a formatter that is not cheap either. Do not guess and hope:
    call `uxTaskGetStackHighWaterMark(NULL)` in each task after it has run a
    full cycle including its worst-case path (a flush *and* a rotation),
    then size each stack to the observed worst case plus roughly 50%. A
    stack overflow here won't crash in the task that overflowed — it will
    corrupt whatever sits next to it, and you will spend a day debugging the
    innocent neighbour.

## Cheat sheet

| Piece | Choice made here | Why |
|-------|------------------|-----|
| Topology | 3 tasks, 2 queues, samples copied by value | No shared mutable state between tasks |
| Priorities | sampler 6 > logger 5 > uplink 4 > ota 3 | Timing-critical first; network last |
| `sample_q` depth 60 | Deep, 50 ms send timeout | A slow flash write must not cost a reading |
| `uplink_q` depth 30 | Shallow, **0 ms** send timeout | A dead network must not stall the logger |
| `vTaskDelayUntil()` | Fixed period, drift-free | `vTaskDelay()` adds the work time to every cycle |
| Batching 30 samples | One `fopen`/`fclose` per batch | Erase cost is per 4 KB sector, not per byte |
| Rotation at 64 KB | `rename()` to `log.1.csv` | Bounded storage regardless of uptime |
| MQTT publish | `esp_mqtt_client_enqueue()`, QoS 1, retain | Non-blocking; dashboards get state on connect |
| Timestamps | SNTP, stored as UTC epoch seconds | Local time in a log breaks twice a year |
| Task WDT | sampler + logger subscribed; uplink not | Only tasks with a duty cycle should be watched |
| OTA trigger | MQTT command → dedicated 8 KB task | Never do the work in the event handler |
| Rollback gate | `self_test_ok()` = IP + broker + filesystem | Proves the device can receive the *next* fix |
| Mutex type | `xSemaphoreCreateMutex()` for the filesystem | Priority inheritance; binary semaphores have none |
| Crash reporting | Reset reason + core dump published at boot | Field telemetry you can't get any other way |

## Running it

You need an ESP32, a sensor (a DHT22, a BME280, or the simulated readings
from module 2-04 if you have neither), and a broker — `mosquitto` on your
laptop is fine.

1. `idf.py set-target esp32`, then `idf.py menuconfig` for the partition
   table, rollback support, core dump destination, and your WiFi/broker
   credentials.
2. `idf.py build flash monitor`. Confirm the boot log shows the reset
   reason, the running partition label, the version string, an IP address,
   an NTP sync, and `connected to broker` — in that order.
3. Watch the data with `mosquitto_sub -h <broker> -t 'site1/#' -v`. Readings
   should appear at your interval with sequence numbers that never skip.
4. Pull the network cable on your router (or stop `mosquitto`) for ten
   minutes. The MQTT stream stops; the CSV must not. Reconnect and confirm
   the device resumes on its own with no code of yours involved.
5. Power-cycle the board mid-write, several times. `/data/log.csv` should
   still mount and parse to the last complete line — that is the LittleFS
   guarantee from module 2-07, and it is worth proving rather than assuming.
6. Serve a new build from `python3 -m http.server`, publish the OTA command,
   and confirm the running partition alternates `ota_0` → `ota_1`.
7. Then build a deliberately broken image — wrong broker password — and push
   it. The device must fail its self-test and come back on the previous
   firmware by itself. If it doesn't, your rollback isn't real.

Steps 4 through 7 are the project. Step 3 is just a demo.

## Stretch goals

- **Backfill after an outage.** Track the sequence number of the last
  successfully published sample in NVS, and on reconnect replay the missed
  rows from the CSV before resuming live data. Rate-limit the replay so it
  can't starve the sampler.
- **Deep sleep between samples.** For a battery build, sample once a minute
  and `esp_deep_sleep()` in between. Everything changes: the queues die with
  RAM, so state must move to RTC memory or NVS, and reconnecting WiFi
  becomes the dominant energy cost — batch ten readings per wake instead.
- **Remote log retrieval.** Serve `/data/log.csv` over HTTP so it can be
  pulled without a cable, streaming it in chunks rather than reading the
  whole file into RAM.
- **A real health topic.** Publish free heap, minimum-ever free heap, each
  task's stack high-water mark, uptime, reset reason and firmware version to
  `site1/<id>/health` once a minute. Leave it running for a week and look at
  whether minimum free heap trends downward — that graph, not any single
  test, is what tells you the firmware is actually stable.
- **Signed updates.** Turn on secure boot and flash encryption before you
  put this anywhere real. An OTA endpoint without signature verification is
  a remote code execution feature you shipped on purpose.
