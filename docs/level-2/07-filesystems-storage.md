# Filesystems & Storage (LittleFS, SD)

Module 1-08 introduced NVS for keeping a setting across a reboot. That
covers configuration, but not "log a reading every ten seconds for six
months". This module covers the rest of the storage picture: **LittleFS**
for files in the ESP32's own flash, **FAT on an SD card** for volume, and
the physics of NAND flash that decides which one you should reach for. Get
this wrong and the failure mode is nasty — not a crash, but a device that
works fine for eight months and then quietly stops writing.

## Choosing the right one

| Need | Use | Why |
|------|-----|-----|
| A handful of settings, key → value | **NVS** | Transactional, wear-levelled, tiny, survives power loss |
| Files in internal flash — logs, certificates, a web UI | **LittleFS** | Power-fail safe, wear-levelled, POSIX file API |
| Gigabytes, or data a human will read on a laptop | **FAT on SD** | Removable, huge, universally readable |
| Read-only assets baked in at build time | **SPIFFS** or an embedded blob | Simple; SPIFFS is legacy and not power-fail safe |

The dividing line between NVS and a filesystem is *shape*, not size. If your
data is naturally `key = value` and you always read the whole value, NVS is
both simpler and safer. If it's append-only, or streamed, or something you'd
want to `fread()` a chunk of, you want a filesystem.

## NVS in ESP-IDF

The C API is more explicit than the Arduino `Preferences` wrapper, and the
error handling matters:

```c
#include "nvs_flash.h"
#include "nvs.h"

void storage_init(void)
{
    esp_err_t err = nvs_flash_init();
    if (err == ESP_ERR_NVS_NO_FREE_PAGES ||
        err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        /* partition is full or was written by a newer NVS — wipe and retry */
        ESP_ERROR_CHECK(nvs_flash_erase());
        err = nvs_flash_init();
    }
    ESP_ERROR_CHECK(err);
}

void save_interval(int32_t seconds)
{
    nvs_handle_t h;
    ESP_ERROR_CHECK(nvs_open("config", NVS_READWRITE, &h));
    ESP_ERROR_CHECK(nvs_set_i32(h, "interval", seconds));
    ESP_ERROR_CHECK(nvs_commit(h));      /* nothing is durable until commit */
    nvs_close(h);
}

int32_t load_interval(int32_t fallback)
{
    nvs_handle_t h;
    if (nvs_open("config", NVS_READONLY, &h) != ESP_OK) return fallback;

    int32_t v = fallback;
    esp_err_t err = nvs_get_i32(h, "interval", &v);
    nvs_close(h);
    /* ESP_ERR_NVS_NOT_FOUND on first boot is normal, not an error */
    return (err == ESP_OK) ? v : fallback;
}
```

Three rules that bite people: **`nvs_commit()` is mandatory** (a set without
a commit can be lost on reset); **keys and namespace names max out at 15
characters** and are silently rejected beyond that; and **first boot always
returns `ESP_ERR_NVS_NOT_FOUND`**, so every read needs a default. Also note
that `nvs_flash_init()` must run before WiFi — the WiFi stack stores
calibration data in NVS itself.

## LittleFS: files in internal flash

LittleFS was designed for microcontrollers with a hard requirement: survive
power loss at *any* instruction without corrupting the filesystem. It does
copy-on-write with metadata pairs, so a half-finished write leaves the
previous version intact. It also wear-levels internally, spreading erases
across the partition.

Add it from the component registry and declare a partition:

```yaml
# main/idf_component.yml
dependencies:
  joltwallet/littlefs: "^1.14.0"
```

```csv
# partitions.csv
# Name,     Type, SubType,  Offset,   Size
nvs,        data, nvs,      0x9000,   0x4000,
phy_init,   data, phy,      0xf000,   0x1000,
factory,    app,  factory,  0x10000,  0x180000,
storage,    data, littlefs, ,         0x100000,
```

Mounting registers it with the VFS layer, after which it is plain C stdio:

```c
#include "esp_littlefs.h"

void fs_mount(void)
{
    esp_vfs_littlefs_conf_t conf = {
        .base_path              = "/data",
        .partition_label        = "storage",
        .format_if_mount_failed = true,
        .dont_mount             = false,
    };
    ESP_ERROR_CHECK(esp_vfs_littlefs_register(&conf));

    size_t total = 0, used = 0;
    if (esp_littlefs_info(conf.partition_label, &total, &used) == ESP_OK) {
        ESP_LOGI(TAG, "littlefs: %u/%u bytes used", used, total);
    }
}

void log_reading(float celsius)
{
    FILE *f = fopen("/data/log.csv", "a");     /* append */
    if (!f) { ESP_LOGE(TAG, "open failed: %s", strerror(errno)); return; }

    fprintf(f, "%lld,%.2f\n", esp_timer_get_time() / 1000, celsius);
    fclose(f);          /* closing flushes — this is where data becomes durable */
}
```

`fopen`, `fprintf`, `fread`, `rename`, `unlink`, `opendir` — all of it works
under `/data` because the VFS maps the path prefix onto the filesystem
driver. The same code compiles on a laptop, which makes the parsing logic
testable off-device (module 2-09).

!!! warning "Never leave a log file open across power cycles"
    Buffered `fprintf()` output lives in RAM until a flush. If the device
    resets with the file open, everything since the last flush is gone —
    and on a filesystem without LittleFS's guarantees, the file itself can
    be corrupted. Open, write, `fclose()`, done. If you must hold a file
    open for throughput, call `fflush(f)` then `fsync(fileno(f))` at a
    defined checkpoint, and treat everything after it as expendable.

## Flash wear, honestly

The ESP32's SPI flash erases in **4 KB sectors** and each sector tolerates
roughly 100,000 erase cycles. You cannot rewrite one byte — the layer
underneath reads the sector, erases it, and writes it back. So the real cost
of a write is the *sector erase* it triggers.

Do the arithmetic before you design the schema. A 1 MB LittleFS partition is
256 sectors; appending 32 bytes every 10 seconds is ~276 KB/day, so with
wear levelling spreading erases evenly the partition takes many years to
exhaust. But rewriting a *single fixed-size record in place* every 10
seconds hammers the same handful of sectors, and 100,000 cycles at 6 per
minute is under two weeks. Same data rate, wildly different lifetime.

The practical rules:

- **Append; don't rewrite.** Appending advances into fresh space. Updating
  in place erases the same sector forever.
- **Batch small writes.** Buffer readings in RAM and flush once a minute
  instead of six times.
- **Don't put a fast-changing counter in NVS.** NVS wear-levels, but it is
  designed for settings, not telemetry.
- **Reserve headroom.** A filesystem near 100% full does far more work per
  write; keep 10–20% free.

## FAT on an SD card

When you need gigabytes, or you want a human to pull the card and open the
CSV in a spreadsheet, use an SD card over SPI:

```c
#include "esp_vfs_fat.h"
#include "sdmmc_cmd.h"
#include "driver/sdspi_host.h"

static sdmmc_card_t *card;

esp_err_t sd_mount(void)
{
    sdmmc_host_t host = SDSPI_HOST_DEFAULT();

    spi_bus_config_t bus = {
        .mosi_io_num = GPIO_NUM_23, .miso_io_num = GPIO_NUM_19,
        .sclk_io_num = GPIO_NUM_18, .quadwp_io_num = -1, .quadhd_io_num = -1,
        .max_transfer_sz = 4000,
    };
    ESP_ERROR_CHECK(spi_bus_initialize(host.slot, &bus, SDSPI_DEFAULT_DMA));

    sdspi_device_config_t slot = SDSPI_DEVICE_CONFIG_DEFAULT();
    slot.gpio_cs = GPIO_NUM_5;
    slot.host_id = host.slot;

    esp_vfs_fat_mount_config_t mount = {
        .format_if_mount_failed = false,   /* true would erase a user's card! */
        .max_files              = 5,
        .allocation_unit_size   = 16 * 1024,
    };

    esp_err_t err = esp_vfs_fat_sdspi_mount("/sdcard", &host, &slot, &mount, &card);
    if (err == ESP_OK) sdmmc_card_print_info(stdout, card);
    return err;
}
```

After that, `/sdcard/log.csv` behaves like any other path. Unmount with
`esp_vfs_fat_sdcard_unmount("/sdcard", card)` before the card is pulled.

SD cards are not a free upgrade: they draw 100 mA-plus in bursts (a weak
3.3 V regulator causes mysterious mount failures), the cheap ones lie about
completing writes, and FAT gives you none of LittleFS's power-fail
guarantees — yanking power mid-write can cost you the whole directory
entry. For a logger that must not lose data, write to LittleFS and *copy* to
SD; for bulk capture where a lost chunk is acceptable, write to SD directly.

## Cheat sheet

| Concept | Detail |
|---------|--------|
| NVS | Key/value settings; `nvs_open` → `set` → **`nvs_commit`** → `nvs_close` |
| NVS limits | Key and namespace ≤ 15 chars; `ESP_ERR_NVS_NOT_FOUND` on first boot is normal |
| `nvs_flash_init()` | Handle `NO_FREE_PAGES`/`NEW_VERSION_FOUND` by erasing; must run before WiFi |
| LittleFS | Power-fail safe, wear-levelled files in internal flash |
| `esp_vfs_littlefs_register(&conf)` | Mounts at `base_path`; then use plain `fopen`/`fprintf` |
| `esp_littlefs_info(label, &total, &used)` | Space check — act before the partition fills |
| Partition | Custom `partitions.csv` entry, subtype `littlefs` (or `spiffs`) |
| SPIFFS | Legacy, no power-fail guarantee, slows badly when full — prefer LittleFS |
| Flash erase unit | 4 KB sector, ~100k cycles — the write cost is the *erase* |
| Wear rule | **Append, don't rewrite in place**; batch small writes; keep 10–20% free |
| FAT/SD | `esp_vfs_fat_sdspi_mount()` → `/sdcard/...`; unmount before removal |
| `format_if_mount_failed` on SD | Leave `false` — `true` silently wipes a user's card |
| Durability | `fclose()` (or `fflush` + `fsync`) is where the data actually lands |

## Exercise

Build a logger with a two-tier storage design. Mount a 1 MB LittleFS
partition at `/data`, buffer readings in a RAM array of 30 samples, and
flush the batch to `/data/log.csv` with one `fopen`/`fprintf`/`fclose` cycle
— then verify with `esp_littlefs_info()` that used bytes grow by roughly the
size you wrote, not by a whole sector per sample.

Add rotation: when `log.csv` exceeds 64 KB, `rename()` it to `log.1.csv`
(deleting any previous `log.1.csv` first) and start fresh, so storage is
bounded regardless of uptime. Then prove the power-fail claim — start a
write loop and yank USB power mid-run, at least five times. On each reboot
confirm the filesystem mounts without reformatting and the surviving CSV
parses cleanly to the last complete line. Finally, note in a comment how
many days of logging your partition supports at your chosen rate, using the
4 KB / 100k-cycle arithmetic above.
