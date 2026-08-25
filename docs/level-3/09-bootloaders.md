# Bootloaders & Firmware Updates

Module 2-05 covered ESP32 OTA at the API level — `esp_ota_begin/write/end`,
handled almost entirely by ESP-IDF. This module builds the mechanism a
bootloader like that relies on from the ground up: how a chip decides which
firmware image to run, how a bootloader safely writes a *new* image without
bricking the device if power is lost mid-write, and the vector-table
relocation trick that lets application code run somewhere other than
address `0`.

## The two-stage boot model

```
[reset vector @ 0x08000000] -> Bootloader (small, rarely updated)
                                   |
                                   +-- validates application image
                                   +-- jumps to Application @ 0x08008000
```

The bootloader occupies the low, fixed flash address the CPU always starts
executing from (module 3-01's reset mechanism). It never changes in normal
operation — only the *application* region gets overwritten during an
update, which means even a firmware update gone wrong can't corrupt the code
responsible for recovering from it, as long as the bootloader itself is
correct and small enough to be worth the extra scrutiny of rarely touching
it.

## Jumping from bootloader to application

```c
typedef void (*app_entry_t)(void);

#define APP_START_ADDR   0x08008000UL

void jump_to_application(void) {
    uint32_t app_stack = *(volatile uint32_t *)(APP_START_ADDR + 0);
    uint32_t app_reset  = *(volatile uint32_t *)(APP_START_ADDR + 4);

    /* relocate the vector table: from this point on, interrupts/exceptions
       must use the APPLICATION's vector table, not the bootloader's */
    #define SCB_VTOR (*(volatile uint32_t *)0xE000ED08)
    SCB_VTOR = APP_START_ADDR;

    __asm volatile ("msr msp, %0" :: "r" (app_stack));  /* set the app's stack pointer */

    app_entry_t app_entry = (app_entry_t)app_reset;
    app_entry();                                         /* never returns */
}
```

This mirrors module 3-01's reset mechanism exactly — the CPU normally reads
SP and PC from address `0x0`; a bootloader does the same read manually from
wherever it decided the application lives, then sets `VTOR` so any
subsequent interrupt looks up its handler in the *application's* table
instead of the bootloader's leftover one. **Forgetting to set `VTOR`** is
one of the most common bootloader bugs: the application appears to boot
(main() runs), then the first interrupt it takes jumps into the
bootloader's vector table instead of its own, executing whatever handler
happened to be at that offset — a crash that looks unrelated to the actual
cause.

## Safe firmware update: never erase what you can't yet replace

The single rule that keeps a failed update from bricking a device: **never
erase the region an old, known-good image occupies until the new image is
fully written and verified.** A naive updater that erases the application
region, then starts writing the new image, guarantees a bricked device on
any power loss during that write.

```c
typedef struct {
    uint32_t magic;          /* sentinel: is this slot valid? */
    uint32_t size;
    uint32_t crc32;
    uint32_t version;
} image_header_t;

#define IMAGE_MAGIC 0x46495254u   /* "FIRT" */

/* dual-bank scheme: write the new image to the INACTIVE bank while the
   active bank keeps running; only flip which bank boots after the new
   image is verified in place. */
int apply_update(const uint8_t *new_image, uint32_t size, uint32_t inactive_bank_addr) {
    image_header_t hdr;
    memcpy(&hdr, new_image, sizeof(hdr));
    if (hdr.magic != IMAGE_MAGIC || hdr.size != size) return -1;

    flash_erase(inactive_bank_addr, size);            /* erasing the INACTIVE bank is safe */
    flash_write(inactive_bank_addr, new_image, size);

    uint32_t computed_crc = crc32(new_image + sizeof(hdr), size - sizeof(hdr));
    if (computed_crc != hdr.crc32) return -2;          /* corrupted write — old bank still bootable */

    set_active_bank(inactive_bank_addr);               /* only now: flip which bank boots */
    return 0;
}
```

A power loss at any point before `set_active_bank()` leaves the currently
running (old) image completely untouched — the device just reboots into the
firmware it already had. This dual-bank approach costs twice the flash of a
single-image scheme, which is exactly the tradeoff it's making: flash space
for update safety.

## Verifying the CRC/header validation logic

The validation logic — not real flash erase/write, which needs actual
hardware — is pure computation and was compiled and run with `gcc`:

```c
#include <stdio.h>
#include <assert.h>
#include <string.h>
#include <stdint.h>

typedef struct { uint32_t magic, size, crc32, version; } image_header_t;
#define IMAGE_MAGIC 0x46495254u

/* simple additive checksum standing in for a real CRC32 for this test */
static uint32_t simple_checksum(const uint8_t *data, uint32_t len) {
    uint32_t sum = 0;
    for (uint32_t i = 0; i < len; i++) sum = (sum * 31) + data[i];
    return sum;
}

int validate_image(const uint8_t *image, uint32_t total_size) {
    image_header_t hdr;
    memcpy(&hdr, image, sizeof(hdr));
    if (hdr.magic != IMAGE_MAGIC) return -1;
    if (hdr.size != total_size) return -2;
    uint32_t actual = simple_checksum(image + sizeof(hdr), total_size - sizeof(hdr));
    if (actual != hdr.crc32) return -3;
    return 0;
}

int main(void) {
    uint8_t payload[16] = "hello firmware!";
    uint8_t image[sizeof(image_header_t) + sizeof(payload)];
    image_header_t hdr = { IMAGE_MAGIC, sizeof(image), 0, 1 };
    hdr.crc32 = simple_checksum(payload, sizeof(payload));
    memcpy(image, &hdr, sizeof(hdr));
    memcpy(image + sizeof(hdr), payload, sizeof(payload));

    assert(validate_image(image, sizeof(image)) == 0);

    image[sizeof(hdr) + 2] ^= 0xFF;   /* corrupt one payload byte */
    assert(validate_image(image, sizeof(image)) == -3);   /* CRC catches it */

    printf("image validation model OK\n");
    return 0;
}
```

## Traps in bootloader design

- **Forgetting `VTOR` relocation**, as above — application boots, then dies
  on the first interrupt.
- **Erasing before verifying**, as above — the single most bricking-prone
  mistake in update design.
- **No rollback on repeated boot failure**: a device that boots the new
  image, which then crashes before marking itself "known good," and reboots
  into the *same* broken image forever needs a boot counter and an automatic
  fallback to the previous bank after N failed boots — without it, one bad
  update permanently bricks every device that received it.
- **Weak or missing signature verification**: CRC catches corruption, not
  malicious images — module 4-02 covers cryptographic signing for update
  authenticity, a materially different and stronger guarantee.

## Cheat sheet

| Concept | Detail |
|---|---|
| Two-stage boot | Fixed bootloader at reset address, jumps to application elsewhere in flash |
| `SCB_VTOR` | Must be set to the application's vector table address after the jump |
| Dual-bank update | Write+verify the new image in the inactive bank; flip active bank only after |
| Never erase-before-verify | The rule that prevents power-loss bricking |
| Boot counter / rollback | Falls back to the previous bank after N failed boots of a new image |
| CRC vs signature | CRC catches corruption; only cryptographic signing catches tampering (module 4-02) |
| Verification here | Header/CRC validation logic compiled/run with `gcc`; real flash erase/write needs actual hardware |

## Exercise

Extend `validate_image` into a small state machine usable by a real
bootloader: `BOOT_TRY_NEW`, `BOOT_CONFIRMED`, `BOOT_ROLLED_BACK`, tracked via
a boot-attempt counter persisted in a header field. Write a portable C
simulation where a "new" image fails to call a `mark_boot_successful()`
function for 3 simulated boots in a row, and assert that your state machine
then reports `BOOT_ROLLED_BACK` and would select the previous bank on the
next real boot. Compile and run it with `gcc`, and in a comment explain
where the boot counter itself must be stored so a hard reset doesn't reset
it back to zero along with everything else.
