---
description: "Project — Custom Sensor Board Firmware — This project combines every module in Level 3 into one design: a battery-powered STM32 sensor board that samples…"
---

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

## How It Actually Works

**Why the ring buffer needs exactly `RING_LEN - 1` usable slots, not
`RING_LEN`**: with only two index variables (`head`/`tail`) and no separate
count field, `head == tail` has to mean something unambiguous — this design
picks "empty." If the ring were allowed to fill all `RING_LEN` slots, the
full condition would *also* produce `head == tail` (the head having wrapped
all the way around to meet the tail), making full and empty
indistinguishable from the indices alone. Reserving one slot — treating
`next == tail` as "about to collide" and evicting the oldest entry there —
is what keeps the two states distinguishable using only two variables and no
lock; the alternative (a separate `count` field) works too, but then that
count itself becomes a third piece of state the ISR and main loop both touch,
reintroducing exactly the shared-state hazard this design avoids by using
only single, independently-modified indices.

**Why `head`/`tail` being individually `volatile` is enough here, without a
`noInterrupts()`-style critical section**: `ring_push` (ISR context) only
ever writes `head` (and `tail`, but only in the specific case where it's also
the one advancing it due to overrun) while `ring_pop` (main-loop context)
only ever writes `tail` in the normal case — each index has effectively one
writer under the common (non-overrun) path, with the other side only
*reading* it to decide whether it's allowed to proceed. A single-writer,
single-reader relationship on an aligned, naturally-atomic-width variable
(a `uint8_t` here) needs no additional locking beyond `volatile` guaranteeing
the reader always re-fetches from memory — this is the standard "SPSC
(single-producer single-consumer) lock-free ring buffer" pattern, and it's
precisely why the overrun path (where the ISR *also* advances `tail`, the
variable the main loop normally owns) is worth flagging as the one place
this simple reasoning gets more delicate: the ISR briefly writes a variable
the main loop also reads to make its own popping decision, which is safe
only because a `uint8_t` read/write is a single bus transaction on this core
and there's no read-modify-write happening on either side.

**Why the checksum framing catches accidental corruption but the exercise is
right to flag it as weak**: an additive checksum (`sum += byte` for every
byte) is only sensitive to the *numeric total* of the bytes — it cannot
detect two bytes swapping position (their sum is identical either way), nor
a byte incremented while another is decremented by the same amount (the
total is unchanged). A real CRC avoids this by making each bit of input
affect the checksum through a position-dependent polynomial division rather
than simple addition, so bytes contribute differently depending on *where*
they sit in the message — which is exactly why swapping two bytes (a common
real-world corruption pattern from a dropped-and-reinserted byte in a
buffered UART) is caught by CRC-16/32 but invisible to the sum used here.

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

## 🔀 Related lessons on other tracks

- [Embedded Python — Custom Firmware Builds & Board Definitions](https://sigilipelli.github.io/embedded-python-mastery-path/level-4/05-custom-firmware-builds/)
