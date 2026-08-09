# Watchdogs & Robust Firmware

Firmware on your desk gets rebooted whenever it misbehaves. Firmware in the
field does not — nobody is going to drive out and power-cycle a sensor on a
rooftop. The difference between a hobby build and a product is almost
entirely in what happens *after* something goes wrong: a task that stops
responding, a bus that latches up, a supply that sags when the WiFi radio
transmits. This module covers the hardware that catches those cases, the
reset-reason plumbing that tells you which one happened, and the defensive
patterns that keep a device alive long enough to receive the fix from
module 2-05.

## The three watchdogs

A watchdog is a hardware timer that resets the chip unless software keeps
telling it things are fine. The ESP32 has three, at different layers:

| Watchdog | Watches | Typical trigger |
|----------|---------|-----------------|
| **Task WDT (TWDT)** | Registered tasks reporting in | A task stuck in a loop, blocked forever on a mutex, or spinning without yielding |
| **Interrupt WDT (IWDT)** | The FreeRTOS tick interrupt still running | An ISR that never returns, or a critical section held far too long |
| **RTC WDT** | The boot process itself | A hang in the bootloader or very early startup |

The IWDT is enabled by default (`CONFIG_ESP_INT_WDT`) with a timeout in the
hundreds of milliseconds, and you should leave it alone — if it fires, the
bug is in your ISR discipline (module 1-07), not in the watchdog. The one
you configure deliberately is the Task WDT.

## Using the Task Watchdog

Subscribe a task, then have it check in inside its main loop. If it fails to
check in within the timeout, the TWDT panics with a backtrace naming exactly
which task went silent:

```c
#include "esp_task_wdt.h"

void sensor_task(void *pv)
{
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));   /* NULL = the calling task */

    for (;;) {
        if (read_sensor() != ESP_OK) {
            ESP_LOGW(TAG, "sensor read failed");
        }
        esp_task_wdt_reset();                  /* "still alive" */
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void app_main(void)
{
    esp_task_wdt_config_t twdt = {
        .timeout_ms    = 10000,                /* generous vs. the 1 s loop */
        .idle_core_mask = (1 << 0) | (1 << 1), /* also watch both idle tasks */
        .trigger_panic = true,                 /* reset, don't just warn */
    };
    /* ESP-IDF may already have initialised the TWDT at startup */
    esp_err_t err = esp_task_wdt_init(&twdt);
    if (err == ESP_ERR_INVALID_STATE) {
        ESP_ERROR_CHECK(esp_task_wdt_reconfigure(&twdt));
    }

    xTaskCreate(sensor_task, "sensor", 4096, NULL, 5, NULL);
}
```

Watching the **idle tasks** via `idle_core_mask` is the cheap catch-all: the
idle task only runs when nothing else is ready, so if it stops being
scheduled, some task is hogging a core without yielding. That one setting
catches a whole class of accidental `while(1)` bugs in code you didn't
instrument.

Set the timeout to several times the loop period, not just above it. A
10 s timeout on a 1 s loop tolerates one slow network call; a 1.2 s timeout
makes the watchdog itself the most likely cause of your resets.

!!! warning "Don't reset the watchdog from a timer callback"
    The tempting "fix" for a nagging TWDT warning is to call
    `esp_task_wdt_reset()` from somewhere that always runs — a timer
    callback, a separate keepalive task, the top of an event handler. That
    converts the watchdog into decoration: it now proves the timer works,
    not that your task does. Only the task being monitored should reset its
    own subscription. If a task legitimately blocks for longer than the
    timeout, either raise the timeout or unsubscribe with
    `esp_task_wdt_delete(NULL)` around the blocking call — deliberately, and
    with a comment saying why.

## Knowing why you rebooted

A device that silently reboots twice a day is a mystery; one that logs *why*
is a bug report. Read the reason at startup and count the bad ones in NVS
(module 2-07):

```c
#include "esp_system.h"

static const char *reset_reason_str(esp_reset_reason_t r)
{
    switch (r) {
    case ESP_RST_POWERON:   return "power-on";
    case ESP_RST_EXT:       return "external pin";
    case ESP_RST_SW:        return "esp_restart()";
    case ESP_RST_PANIC:     return "panic / exception";
    case ESP_RST_INT_WDT:   return "interrupt watchdog";
    case ESP_RST_TASK_WDT:  return "task watchdog";
    case ESP_RST_WDT:       return "other watchdog";
    case ESP_RST_DEEPSLEEP: return "deep sleep wake";
    case ESP_RST_BROWNOUT:  return "brownout";
    default:                return "unknown";
    }
}

void log_boot_reason(void)
{
    esp_reset_reason_t r = esp_reset_reason();
    ESP_LOGI(TAG, "boot reason: %s", reset_reason_str(r));

    if (r == ESP_RST_PANIC || r == ESP_RST_TASK_WDT || r == ESP_RST_INT_WDT) {
        uint32_t n = nvs_read_u32_or("crashes", 0) + 1;
        nvs_write_u32("crashes", n);
        ESP_LOGW(TAG, "unclean reboot #%lu", n);
        /* publish this over MQTT (module 2-04) — it is your field telemetry */
    }
}
```

That counter is worth more than it looks. A fleet where 2% of devices
reboot on brownout every night tells you the power supply is marginal;
`ESP_RST_TASK_WDT` clustered on one firmware version tells you which release
to roll back. It also pairs directly with OTA rollback (module 2-05): a
crash loop during probation is exactly what the bootloader's fallback is
there for.

## Brownout: when the supply sags

The ESP32's radio pulls current in short, sharp bursts — a WiFi transmit
can spike to several hundred milliamps. On a thin USB cable, a weak
regulator, or a battery near its cutoff, the rail dips and the chip's
behaviour becomes undefined: corrupted flash writes, garbage in RAM, silent
misbehaviour.

The brownout detector (`CONFIG_ESP_BROWNOUT_DET`, on by default) resets the
chip when the supply crosses a configurable threshold, turning that undefined
behaviour into a clean, detectable reboot reported as `ESP_RST_BROWNOUT`.
Leave it enabled. If you see it in the field, the fix is hardware — a fatter
supply, a shorter cable, and a bulk capacitor (a 100 µF electrolytic plus a
0.1 µF ceramic across the module's 3.3 V and GND) to ride out the bursts —
not a lower threshold.

## Defensive patterns

The watchdogs are a last resort. These patterns stop you needing them:

**Bounded retries with backoff.** An infinite `while (!connected) retry();`
turns a transient outage into a hang. Cap the attempts, back off
exponentially, and have a defined give-up branch:

```c
esp_err_t connect_with_backoff(void)
{
    uint32_t delay_ms = 500;
    for (int attempt = 0; attempt < 6; attempt++) {
        if (try_connect() == ESP_OK) return ESP_OK;
        ESP_LOGW(TAG, "attempt %d failed, retrying in %lu ms", attempt, delay_ms);
        vTaskDelay(pdMS_TO_TICKS(delay_ms));
        esp_task_wdt_reset();
        delay_ms = (delay_ms < 30000) ? delay_ms * 2 : 30000;   /* capped */
    }
    return ESP_ERR_TIMEOUT;      /* the caller decides: degrade, or restart */
}
```

**Never ignore an `esp_err_t`.** Every ESP-IDF call returns one. Use
`ESP_ERROR_CHECK()` for genuinely unrecoverable setup (it aborts loudly with
a backtrace — better than limping on), and explicit handling for anything
that can fail in normal operation. A dropped return code is a bug that
reappears as an unexplained reboot six weeks later.

**Timeouts on everything that blocks.** `portMAX_DELAY` on a queue the
producer will always feed is fine; `portMAX_DELAY` on a mutex a buggy task
might never release is a deadlock waiting to happen. Prefer a finite timeout
plus a "what do I do if this times out" branch.

**Watch the heap.** A slow leak looks like weeks of uptime followed by
allocation failures. Log `esp_get_free_heap_size()` and
`esp_get_minimum_free_heap_size()` periodically; if the minimum keeps
falling over days, you have a leak, not noise.

**Deliberate restart beats limping.** If the device reaches a state it can't
recover from, `esp_restart()` is a legitimate, well-tested recovery path —
just log the reason first, and rate-limit it so you don't build a reboot
loop that outruns your OTA window.

## Cheat sheet

| Concept | Detail |
|---------|--------|
| Task WDT | Resets the chip if a subscribed task stops checking in |
| `esp_task_wdt_add(NULL)` / `esp_task_wdt_reset()` | Subscribe this task / report in from it |
| `esp_task_wdt_config_t` | `.timeout_ms`, `.idle_core_mask`, `.trigger_panic` |
| `idle_core_mask` | Watch the idle tasks — catches any task hogging a core |
| Timeout sizing | Several × the loop period; too tight and the WDT *is* the bug |
| Anti-pattern | Resetting the WDT from a timer/keepalive — it then proves nothing |
| Interrupt WDT | On by default; firing means an ISR or critical section ran too long |
| `esp_task_wdt_delete(NULL)` | Unsubscribe around a known-long blocking call, deliberately |
| `esp_reset_reason()` | `ESP_RST_POWERON` / `PANIC` / `TASK_WDT` / `INT_WDT` / `BROWNOUT` / ... |
| Crash counter in NVS | Cheapest field telemetry there is — publish it |
| Brownout detector | On by default; `ESP_RST_BROWNOUT` means fix the *hardware* |
| Retry policy | Bounded attempts, exponential backoff, capped, with a give-up branch |
| Heap trend | `esp_get_minimum_free_heap_size()` falling over days = leak |
| `esp_restart()` | A valid recovery path — log first, rate-limit it |

## Exercise

Instrument a two-task project (module 2-01) for field survival. Subscribe
both tasks to the Task WDT with a 10 s timeout and `idle_core_mask` covering
both cores. Add a serial command `hang` that makes one task enter
`while (1) {}` — confirm the device resets, and that `idf.py monitor` names
the offending task in the backtrace. Add a second command `block` that takes
a mutex and never gives it, and confirm the *other* task is the one the
watchdog reports; explain in a comment why the reported task isn't the
guilty one.

Then add boot forensics: at startup log `esp_reset_reason()` in human
readable form, keep separate NVS counters for panic, task-WDT and brownout
resets, and print all three every boot. Trigger each of the three
deliberately — `hang` for the watchdog, a null-pointer dereference for the
panic, and a brownout by browning out the supply (or by lowering
`CONFIG_ESP_BROWNOUT_DET_LVL_SEL` until it trips) — and confirm each
increments only its own counter across a power cycle.
