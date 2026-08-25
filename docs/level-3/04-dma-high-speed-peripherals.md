# DMA & High-Speed Peripherals

Every peripheral read/write so far has gone through the CPU: read a
register, store it in a variable, move on. That works for a button or an
occasional I2C transaction, but it falls apart for a 1 Msps ADC stream or a
SPI display refresh — the CPU would spend nearly all its time shuffling
bytes it never actually needs to look at. **DMA** (Direct Memory Access) is
a separate piece of hardware that moves data between a peripheral and memory
without the CPU executing a single load/store instruction per byte.

## What DMA actually is

A DMA controller is a small state machine with its own registers: a source
address, a destination address, a transfer count, and a trigger. Once
configured and armed, it runs independently — the CPU is free to do other
work (or sleep) until the DMA controller raises an interrupt saying "done."

```c
#define DMA1_S0CR    (*(volatile uint32_t *)0x40026010)  /* stream 0 config */
#define DMA1_S0NDTR  (*(volatile uint32_t *)0x40026014)  /* number of items */
#define DMA1_S0PAR   (*(volatile uint32_t *)0x40026018)  /* peripheral address */
#define DMA1_S0M0AR  (*(volatile uint32_t *)0x4002601C)  /* memory address */

void dma_start_adc_capture(uint32_t *dest, uint32_t adc_dr_addr, uint16_t count) {
    DMA1_S0PAR  = adc_dr_addr;             /* source: ADC data register */
    DMA1_S0M0AR = (uint32_t)dest;          /* destination: our RAM buffer */
    DMA1_S0NDTR = count;                   /* how many transfers */
    DMA1_S0CR  |= (1u << 0);               /* EN: arm the stream */
}
```

Once armed, every ADC conversion automatically lands in `dest[]` — no
`ADC_DR` read anywhere in application code. The CPU only re-enters the
picture when the transfer completes (via an interrupt) or, for continuous
capture, never at all if the stream is configured in circular mode.

## Cache coherency: the trap that doesn't exist on Cortex-M0/M3, but does on M7

On chips without a data cache (most Cortex-M0/M3/M4 parts), what DMA writes
to RAM is immediately what the CPU sees on its next read — memory is memory.
On a **Cortex-M7** with a data cache, that stops being true: the CPU may
have a stale cached copy of `dest[]` from *before* the DMA write, and reads
old data even though the DMA transfer genuinely completed.

```c
void adc_capture_done_handler(uint32_t *dest, uint16_t count) {
    /* M7 with D-Cache enabled: the CPU's cached copy of dest[] may predate
       the DMA write. Must invalidate before reading. */
    SCB_InvalidateDCache_by_Addr((uint32_t *)dest, count * sizeof(uint32_t));

    for (uint16_t i = 0; i < count; i++) {
        process_sample(dest[i]);   /* now guaranteed to see DMA's data */
    }
}
```

The same problem runs the other direction: if the CPU writes a buffer that
DMA will then *read* (e.g. a UART TX buffer), the cache may hold those
writes and not have flushed them to RAM yet when DMA starts reading — the
buffer must be explicitly **cleaned** (written back) before arming the DMA
transfer. Missing either direction produces a bug that is maddening to
chase: the data is *usually* right (cache often does flush/evict in time
under light load) and wrong only under specific timing, making it look like
a flaky peripheral rather than a cache bug.

## Double buffering: avoiding the "who owns this buffer" race

Continuous DMA capture needs a way to hand a full buffer to application code
without either side touching a buffer the other is using:

```c
#define BUF_LEN 256
static uint32_t bufA[BUF_LEN], bufB[BUF_LEN];
static volatile int active_buf_is_a = 1;
static volatile int buffer_ready = 0;

/* Called from the DMA "transfer complete" ISR */
void DMA_TC_IRQHandler(void) {
    active_buf_is_a = !active_buf_is_a;     /* DMA now fills the other buffer */
    DMA1_S0M0AR = active_buf_is_a ? (uint32_t)bufA : (uint32_t)bufB;
    buffer_ready = 1;                        /* signal: the OLD buffer is done */
}

void main_loop(void) {
    if (buffer_ready) {
        buffer_ready = 0;
        uint32_t *finished_buf = active_buf_is_a ? bufB : bufA; /* the one NOT active */
        process_full_buffer(finished_buf);   /* safe: DMA is now filling the other one */
    }
}
```

`buffer_ready` must be `volatile` for the same reason any ISR-shared flag
must be (module 3-06 covers this in depth) — otherwise the compiler may
cache `main_loop`'s read of it in a register and never notice the ISR
changed it.

## Verifying the double-buffer handoff logic in portable C

The buffer-swap bookkeeping is pure logic and can be checked without any
real DMA hardware. Compiled and run with `gcc`:

```c
#include <stdio.h>
#include <assert.h>

int active_is_a = 1, ready = 0;

void on_transfer_complete(void) {
    active_is_a = !active_is_a;
    ready = 1;
}

int main(void) {
    /* simulate three completed transfers */
    for (int i = 0; i < 3; i++) {
        int was_active_a = active_is_a;
        on_transfer_complete();
        assert(ready == 1);
        assert(active_is_a == !was_active_a);      /* flipped */
        int finished_is_a = !active_is_a;           /* the one NOT now active */
        assert(finished_is_a == was_active_a);      /* consumer reads the buffer DMA just filled */
        ready = 0;
    }
    printf("double-buffer handoff model OK\n");
    return 0;
}
```

## Traps specific to DMA and high-speed peripherals

- **Cache coherency**, as above — only on cached cores (M7, not M0/M3/M4),
  but silent and load-dependent when it bites.
- **Buffer alignment**: many DMA controllers require the destination address
  aligned to the transfer width (4-byte aligned for 32-bit transfers);
  an unaligned buffer either faults or silently corrupts adjacent memory.
- **Racing the "transfer complete" flag**: reading `DMA1_S0NDTR` to check
  "is it done yet" instead of using the completion interrupt is a classic
  polling race — the count can read nonzero on one read and hit zero a
  cycle later without ever observing the exact boundary reliably.
- **Peripheral FIFO underrun/overrun**: if DMA can't keep up with the
  peripheral's data rate (bus contention from another high-priority DMA
  stream, for example), samples are silently dropped at the peripheral level
  — the DMA transfer count is *not* evidence that every conversion was
  captured.

## Cheat sheet

| Concept | Detail |
|---|---|
| DMA controller | Independent hardware that moves data peripheral<->memory without CPU load/store per byte |
| Circular mode | DMA re-arms automatically at the end of the buffer — true continuous capture |
| D-Cache coherency | Only relevant on cached cores (e.g. M7); invalidate before CPU read, clean before DMA read |
| `SCB_InvalidateDCache_by_Addr` | CMSIS call to drop stale cached copies before reading DMA'd data |
| Double buffering | Two buffers, DMA fills one while code processes the other — avoids torn reads |
| `volatile` on shared flags | Required wherever an ISR sets a flag `main_loop` polls |
| Verification here | Buffer-swap bookkeeping compiled/run with `gcc`; cache/DMA hardware behavior reviewed, not executed |

## Exercise

Extend the double-buffer model into a three-buffer ring (to tolerate the
consumer occasionally running one cycle behind) with an explicit
"buffer overrun" detection: if the DMA ISR fires a third time before the
consumer has freed the oldest buffer, set an `overrun_count` instead of
silently overwriting data the consumer hasn't read yet. Write assertions
covering the non-overrun case and the overrun case, compile and run with
`gcc`, and in a comment explain why this ring needs at least one more buffer
than the simple double-buffer scheme to detect overrun instead of just
delaying it.
