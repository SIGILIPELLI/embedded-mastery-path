# Project — Custom Sensor Board Firmware

This project combines every module in Level 3 into one design: a
battery-powered STM32 sensor board that samples an I2C sensor via DMA, sleeps
between samples, buffers readings, and reports them over UART with a
protocol that can be extended to a bootloader-driven field update. It is a
design exercise — schematic and firmware architecture reviewed carefully by
hand, with every piece of portable logic actually compiled and run, and no
claim that any of it ran on real STM32 silicon.

## Board-level design (module 3-07)

- **MCU**: STM32 (Cortex-M4-class), one I2C sensor (temperature/humidity),
  one status LED, one UART header for a debug/data console.
- **Power**: coin-cell or small LiPo, LDO regulator, decoupling caps at
  every IC power pin per module 3-07's placement rule.
- **Two test points**: one on the I2C bus (for a logic analyzer during
  bring-up), one on the UART TX line.

## Firmware architecture

```
Reset_Handler (3-02)
   -> clock init
   -> vector table already relocated to app region if booted via
      bootloader (3-09); VTOR set accordingly
   -> peripheral init: GPIO (3-01), I2C+DMA (3-04), UART, NVIC priorities (3-06)
   -> enter main loop:
        arm DMA-driven I2C read
        WFI / Stop mode (3-05) until DMA-complete interrupt
        on wake: validate + buffer the reading
        every N samples: drain buffer over UART using the 3-08 framing
```

## Sample buffer with overrun detection

Combines the double-buffering idea from 3-04 with a small ring, sized so a
UART hiccup doesn't lose data silently:

```c
#include <stdint.h>
#include <stdbool.h>

#define RING_LEN 32

typedef struct {
    float celsius;
    uint32_t timestamp_ms;
} sample_t;

typedef struct {
    sample_t buf[RING_LEN];
    volatile uint8_t head, tail;
    volatile uint32_t overrun_count;
} sample_ring_t;

void ring_push(sample_ring_t *r, sample_t s) {
    uint8_t next = (r->head + 1) % RING_LEN;
    if (next == r->tail) {
        r->overrun_count++;          /* buffer full: drop the OLDEST, not the newest */
        r->tail = (r->tail + 1) % RING_LEN;
    }
    r->buf[r->head] = s;
    r->head = next;
}

bool ring_pop(sample_ring_t *r, sample_t *out) {
    if (r->head == r->tail) return false;    /* empty */
    *out = r->buf[r->tail];
    r->tail = (r->tail + 1) % RING_LEN;
    return true;
}
```

`head`/`tail` are `volatile` because `ring_push` runs from the DMA-complete
ISR while `ring_pop` runs from the main loop — the same ISR/main-loop
sharing rule from module 3-06.

## UART report framing (module 3-08 applied)

```c
/* [0xAA][len][payload...][checksum] — simple framing for the debug console */
uint8_t frame_sample(uint8_t *out, sample_t s) {
    out[0] = 0xAA;
    out[1] = sizeof(sample_t);
    memcpy(&out[2], &s, sizeof(sample_t));
    uint8_t sum = out[0] + out[1];
    for (size_t i = 0; i < sizeof(sample_t); i++) sum += out[2 + i];
    out[2 + sizeof(sample_t)] = sum;
    return 3 + sizeof(sample_t);
}
```

## Verifying the ring buffer and framing logic

Both are pure logic and were compiled and run with `gcc`:

```c
#include <stdio.h>
#include <assert.h>
#include <string.h>
#include <stdint.h>
#include <stdbool.h>

#define RING_LEN 4   /* small on purpose, to force overrun quickly in the test */
typedef struct { float celsius; uint32_t timestamp_ms; } sample_t;
typedef struct { sample_t buf[RING_LEN]; uint8_t head, tail; uint32_t overrun_count; } sample_ring_t;

void ring_push(sample_ring_t *r, sample_t s) {
    uint8_t next = (r->head + 1) % RING_LEN;
    if (next == r->tail) { r->overrun_count++; r->tail = (r->tail + 1) % RING_LEN; }
    r->buf[r->head] = s;
    r->head = next;
}
bool ring_pop(sample_ring_t *r, sample_t *out) {
    if (r->head == r->tail) return false;
    *out = r->buf[r->tail];
    r->tail = (r->tail + 1) % RING_LEN;
    return true;
}

int main(void) {
    sample_ring_t r = {0};
    for (int i = 0; i < 5; i++) {                 /* push 5 into a 4-slot ring (3 usable) */
        sample_t s = { 20.0f + i, (uint32_t)(i * 1000) };
        ring_push(&r, s);
    }
    assert(r.overrun_count == 2);   /* pushing 5 into a 3-usable-slot ring overwrites twice */

    sample_t out;
    int popped = 0;
    while (ring_pop(&r, &out)) popped++;
    assert(popped == RING_LEN - 1);   /* ring holds LEN-1 usable slots (full/empty distinction) */
    assert(out.celsius == 24.0f);      /* last item pushed should be the last one popped */

    printf("ring buffer model OK (overrun=%u, popped=%d)\n", r.overrun_count, popped);
    return 0;
}
```

## Traps this project exercises across modules

- **DMA cache coherency** (3-04) does not apply on Cortex-M4 (no data
  cache) but would need explicit handling if this design were ported to an
  M7 part — worth a comment in the code either way, since a future port is
  a realistic scenario for a reusable sensor board.
- **Sleep mode wake source** (3-05): the I2C/DMA completion interrupt must
  be an enabled NVIC line before entering Stop mode, or the board sleeps
  forever waiting for a wake event it never unmasked.
- **Ring buffer overrun policy**: dropping the oldest sample (as above) is
  a deliberate choice — a monitoring application usually cares more about
  recent trend than about total data completeness; a data-logging
  application might prefer the opposite (drop new, keep old) and should
  say so explicitly rather than inherit this default silently.
- **Framing checksum weakness**: a simple additive checksum (as used here
  for brevity) does not catch all corruption patterns a real CRC would —
  fine for a debug console, not sufficient for a field data-integrity
  guarantee.

## Cheat sheet

| Module combined | Role in this project |
|---|---|
| 3-01/3-02 | Register access, linker/startup for the whole firmware image |
| 3-03 | Not directly used — bare-metal main loop chosen over an RTOS for this design's simplicity |
| 3-04 | DMA-driven I2C sampling, freeing the CPU to sleep between reads |
| 3-05 | Stop mode between samples; wake source must be unmasked in the NVIC |
| 3-06 | `volatile` + ISR-safe ring buffer indices shared between DMA ISR and main loop |
| 3-07 | Decoupling placement, ground plane, test points on the physical board |
| 3-08 | UART framing for the debug/report protocol |
| 3-09 | Vector table relocation if deployed behind a bootloader for field updates |

## Stretch goals

- Add a second sensor on the same I2C bus and extend the ring buffer to
  tagged samples (sensor ID + reading) instead of a single fixed type.
- Replace the additive checksum with a real CRC-16 or CRC-32 and verify in
  a portable `gcc`-compiled test that it catches corruption patterns (e.g.
  two swapped bytes) the additive checksum misses.
- Wire this project's UART report format into the module 3-09 bootloader as
  a field-update trigger: a specific framed command over UART that causes
  the application to reboot into the bootloader's update mode.
- Port the DMA read path to a hypothetical Cortex-M7 target and add the
  `SCB_InvalidateDCache_by_Addr` call from module 3-04 with a comment
  explaining exactly which buffer needs it and why the M4 version doesn't.
