# OTA Firmware Updates

Everything so far ended with `idf.py flash` and a USB cable. That stops
being possible the moment a device is glued inside a wall, buried in a
machine, or sold to someone in another country. **OTA** (over-the-air)
updates let firmware replace itself over WiFi — and the interesting part
isn't the download, it's making sure a bad build doesn't brick the fleet.
This module covers the partition layout OTA requires, the two APIs ESP-IDF
offers, and the rollback machinery that turns "we shipped a broken update"
from a truck roll into a reboot.

## Why it needs two app slots

A program cannot overwrite the flash it is currently executing from. So OTA
works by keeping **two app partitions** and alternating between them:

1. Firmware runs from `ota_0`.
2. It downloads the new image into `ota_1` — the slot it isn't using.
3. It verifies the image, then writes the *otadata* partition to say "boot
   `ota_1` next".
4. Reboot. The bootloader reads otadata and starts `ota_1`.
5. The next update goes back into `ota_0`.

`otadata` is a tiny (0x2000) partition holding two mirrored copies of the
boot selection plus each slot's state — it is the single source of truth the
second-stage bootloader consults on every boot.

The default single-app partition table has no room for this, so OTA always
means a custom `partitions.csv`. On a 4 MB module:

```csv
# Name,     Type, SubType,  Offset,   Size,      Flags
nvs,        data, nvs,      0x9000,   0x4000,
otadata,    data, ota,      0xd000,   0x2000,
phy_init,   data, phy,      0xf000,   0x1000,
ota_0,      app,  ota_0,    0x10000,  0x1A0000,
ota_1,      app,  ota_1,    ,         0x1A0000,
storage,    data, littlefs, ,         0x80000,
```

Then in `menuconfig` (module 2-02): **Partition Table → Custom partition
table CSV**, filename `partitions.csv`. Note what this costs — your app now
gets ~1.6 MB instead of ~3.4 MB, because you're paying for a spare copy of
itself. Budget for that *before* you write the firmware, not after.

!!! warning "App partitions must be 64 KB-aligned"
    `app` partitions must start on a 0x10000 boundary. Leave the offset
    column blank and the generator packs and aligns them for you — safer
    than hand-computing offsets and getting a cryptic build failure. Verify
    the result with `idf.py partition-table`.

## The easy path: `esp_https_ota()`

If the new image lives behind an HTTPS URL, one call does the whole job —
download, write, verify, set boot partition:

```c
#include "esp_https_ota.h"
#include "esp_http_client.h"
#include "esp_log.h"
#include "esp_system.h"

static const char *TAG = "ota";

/* embedded with EMBED_TXTFILES in CMakeLists.txt */
extern const uint8_t server_cert_pem_start[] asm("_binary_ca_cert_pem_start");

esp_err_t do_ota_update(void)
{
    esp_http_client_config_t http_cfg = {
        .url               = "https://firmware.example.com/app-1.4.2.bin",
        .cert_pem          = (const char *)server_cert_pem_start,
        .timeout_ms        = 10000,
        .keep_alive_enable = true,
    };
    esp_https_ota_config_t ota_cfg = {
        .http_config = &http_cfg,
    };

    esp_err_t err = esp_https_ota(&ota_cfg);
    if (err == ESP_OK) {
        ESP_LOGI(TAG, "update written — rebooting into the new slot");
        esp_restart();
    }
    ESP_LOGE(TAG, "ota failed: %s", esp_err_to_name(err));
    return err;
}
```

`esp_https_ota()` blocks for the whole transfer, so call it from a dedicated
task with a generous stack (8 KB is a reasonable start) — not from a
callback and not from `app_main()` if anything else needs to keep running.

## The manual path: `esp_ota_*`

When the image arrives some other way — MQTT chunks, a local HTTP server, an
SD card, a BLE transfer — you drive the same machinery yourself:

```c
#include "esp_ota_ops.h"

const esp_partition_t *update = esp_ota_get_next_update_partition(NULL);
esp_ota_handle_t handle = 0;

/* OTA_WITH_SEQUENTIAL_WRITES avoids erasing the whole partition up front */
ESP_ERROR_CHECK(esp_ota_begin(update, OTA_WITH_SEQUENTIAL_WRITES, &handle));

while ((n = receive_next_chunk(buf, sizeof(buf))) > 0) {
    ESP_ERROR_CHECK(esp_ota_write(handle, buf, n));   /* append, in order */
}

esp_err_t err = esp_ota_end(handle);                  /* validates the image */
if (err != ESP_OK) {
    ESP_LOGE(TAG, "image rejected: %s", esp_err_to_name(err));
    return err;                                       /* boot partition untouched */
}
ESP_ERROR_CHECK(esp_ota_set_boot_partition(update));
esp_restart();
```

`esp_ota_get_next_update_partition(NULL)` picks the slot you are *not*
running from, so the alternation is handled for you. `esp_ota_end()` is the
gate: it checks the image header and, if secure boot is on, the signature.
Only call `esp_ota_set_boot_partition()` after it returns `ESP_OK` — that
call is the point of no return.

## Rollback: surviving a bad build

Writing a valid image is not the same as writing a *working* one. Firmware
that flashes perfectly and then crashes before joining WiFi is unreachable —
you cannot push a fix to a device that never comes online. ESP-IDF's answer
is a first-boot probation period, enabled in `menuconfig` under
**Bootloader config → Enable app rollback support**
(`CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE`).

With it on, a freshly flashed slot boots in state `ESP_OTA_IMG_PENDING_VERIFY`.
The new firmware must actively declare itself healthy:

```c
#include "esp_ota_ops.h"

void app_main(void)
{
    const esp_partition_t *running = esp_ota_get_running_partition();
    esp_ota_img_states_t state;

    if (esp_ota_get_state_partition(running, &state) == ESP_OK &&
        state == ESP_OTA_IMG_PENDING_VERIFY) {

        ESP_LOGW(TAG, "first boot of a new image — on probation");

        if (self_test_ok()) {                 /* WiFi joined? sensors respond? */
            esp_ota_mark_app_valid_cancel_rollback();
            ESP_LOGI(TAG, "image confirmed good");
        } else {
            ESP_LOGE(TAG, "self-test failed — rolling back now");
            esp_ota_mark_app_invalid_rollback_and_reboot();   /* does not return */
        }
    }
    /* ... normal startup ... */
}
```

If the device resets before either call — panic, watchdog (module 2-08),
brownout — the bootloader marks the slot `ESP_OTA_IMG_ABORTED` and boots the
previous, known-good slot instead. That is the safety net: **a crash during
probation is automatically undone.**

Make `self_test_ok()` mean something. "The code reached `app_main()`" is
almost worthless as a health check; "we obtained an IP and the broker
accepted our credentials within 60 seconds" is a real one, because it proves
the device can still be reached to receive the *next* update.

!!! warning "Marking the app valid too early defeats the whole mechanism"
    Calling `esp_ota_mark_app_valid_cancel_rollback()` as the first line of
    `app_main()` — a very common copy-paste — turns rollback into a no-op.
    You are telling the bootloader "this build is fine" before anything has
    been tested. Delay the call until connectivity actually works, and let a
    watchdog reset handle the case where it never does.

## Versioning

Every ESP-IDF image carries an `esp_app_desc_t` in its header. Set the
version from a `version.txt` file or the `PROJECT_VER` CMake variable, then
read it at runtime:

```c
#include "esp_app_desc.h"

const esp_app_desc_t *app = esp_app_get_description();
ESP_LOGI(TAG, "running %s v%s, built %s %s (idf %s)",
         app->project_name, app->version, app->date, app->time, app->idf_ver);
```

Publish that string over MQTT at startup and your fleet dashboard tells you
who is on what — which is how you find out an update stalled. For rejecting
*downgrades* to known-vulnerable builds, ESP-IDF has a separate hardware
mechanism, anti-rollback (`CONFIG_BOOTLOADER_APP_ANTI_ROLLBACK`), which
burns a monotonic security version into eFuses; note it is permanent and
irreversible, so treat it as a production decision rather than a
development-time toggle.

## Cheat sheet

| Concept | Detail |
|---------|--------|
| Why two slots | Can't overwrite the flash you're executing from — alternate `ota_0`/`ota_1` |
| `otadata` | 0x2000 data partition holding the boot selection and per-slot state |
| Partition table | Custom `partitions.csv` + menuconfig; `app` partitions align to 64 KB |
| Cost | Roughly half your flash budget goes to the spare slot |
| `esp_https_ota(&cfg)` | One blocking call: download → write → verify. Needs its own task |
| `esp_ota_get_next_update_partition(NULL)` | The slot you're *not* running from |
| `esp_ota_begin/write/end` | Manual path; `end()` validates before you commit |
| `esp_ota_set_boot_partition()` | Point of no return — only after `esp_ota_end() == ESP_OK` |
| `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE` | New image boots as `ESP_OTA_IMG_PENDING_VERIFY` |
| `esp_ota_mark_app_valid_cancel_rollback()` | "Confirmed good" — call it *late*, after a real self-test |
| `esp_ota_mark_app_invalid_rollback_and_reboot()` | Give up now and return to the old slot |
| Crash during probation | Bootloader falls back to the previous slot automatically |
| `esp_app_get_description()` | Version, build date, IDF version from the image header |

## Exercise

Convert your module 2-02 project to a two-OTA-slot layout: write
`partitions.csv`, select it in `menuconfig`, and confirm the result with
`idf.py partition-table`. Add an OTA task that fetches an image over HTTP
from a local `python3 -m http.server` on your laptop, plus a startup block
that logs which partition it is running from (`esp_ota_get_running_partition()->label`)
and the version string from `esp_app_get_description()`. Update twice, and
verify the label alternates `ota_0` → `ota_1` → `ota_0`.

Then test the safety net deliberately. Enable rollback support, add a
self-test that requires an IP address within 30 s before calling
`esp_ota_mark_app_valid_cancel_rollback()`, and build a *deliberately
broken* image with the wrong WiFi password. Flash it over OTA and confirm
the device reboots back into the previous, working firmware on its own —
that recovery, not the download, is the thing you actually needed to prove.
