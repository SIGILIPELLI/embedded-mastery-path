# Bare-Metal ARM Cortex-M (STM32)

Every board in Level 1 and 2 ran on top of a framework — Arduino-ESP32 or
ESP-IDF — that booted FreeRTOS, set up the clock tree, and handed you a
`setup()`/`loop()` or `app_main()` already running in a sane environment.
This module strips all of that away. On an STM32 (a Cortex-M0/M3/M4/M7 chip
from ST), you'll write the code that runs *before* `main()`, understand
exactly what memory looks like at reset, and talk to peripherals as raw
memory-mapped registers with no library in between. The toolchain is
`arm-none-eabi-gcc` instead of the Arduino/IDF build system, and the flash
tool is OpenOCD or ST-Link instead of `esptool.py`.

There is no STM32 board wired up for this course, so nothing in this module
was flashed to real silicon. What follows is verified two ways: the C is
technically accurate against the ARM Cortex-M architecture reference and
ST's reference manuals, and every piece of portable logic (bit math, framing,
state machines) is actually compiled and run with `gcc` on this machine —
called out explicitly wherever that happens.

## What "bare-metal" removes

Arduino's `digitalWrite(13, HIGH)` compiles down to a handful of instructions
that: look up which port pin 13 maps to, compute the bit position, and
finally write to a GPIO register. Bare metal skips straight to that last
step — you write to the register yourself:

```c
#include <stdint.h>

#define GPIOA_BASE   0x40020000UL
#define RCC_BASE     0x40023800UL

#define RCC_AHB1ENR  (*(volatile uint32_t *)(RCC_BASE + 0x30))
#define GPIOA_MODER  (*(volatile uint32_t *)(GPIOA_BASE + 0x00))
#define GPIOA_ODR    (*(volatile uint32_t *)(GPIOA_BASE + 0x14))

void led_init(void) {
    RCC_AHB1ENR |= (1u << 0);          /* enable GPIOA clock */
    GPIOA_MODER &= ~(0x3u << (5 * 2)); /* clear mode bits for PA5 */
    GPIOA_MODER |=  (0x1u << (5 * 2)); /* set PA5 to general-purpose output */
}

void led_toggle(void) {
    GPIOA_ODR ^= (1u << 5);
}
```

Every peripheral on the chip — GPIO, UART, timers, ADC — is a block of
registers at a fixed address in the memory map (given in the reference
manual's memory map table, not the datasheet). `volatile` is not optional
here: without it, the compiler is free to assume `GPIOA_ODR` never changes
behind its back and can cache the read or eliminate what looks like a
redundant write. That single keyword is the boundary between C that models
memory and C that models hardware.

## Enable-before-configure, and why the datasheet order matters

`RCC_AHB1ENR` gates the clock to GPIOA. Cortex-M peripherals are clock-gated
to save power at reset — writing to `GPIOA_MODER` before enabling its clock
in `RCC_AHB1ENR` is undefined in practice: on some silicon revisions the
write is silently dropped, on others it stalls the bus. This ordering bug is
one of the most common bare-metal traps, and it never shows up in a
framework because the HAL's `GPIO_Init()` always enables the clock for you.

## The vector table and reset entry

A Cortex-M chip doesn't run a bootloader that jumps to `main()` the way a PC
does. At reset, the CPU reads exactly two 32-bit words from address `0x0`:
the initial stack pointer, then the reset handler's address. It loads `SP`
with the first and jumps to the second — no code runs before that.

```c
extern uint32_t _estack;      /* from the linker script, module 3-02 */
void Reset_Handler(void);
void Default_Handler(void);

void NMI_Handler(void)        __attribute__((weak, alias("Default_Handler")));
void HardFault_Handler(void)  __attribute__((weak, alias("Default_Handler")));

__attribute__((section(".isr_vector")))
const void *vector_table[] = {
    &_estack,           /* [0]  initial SP */
    Reset_Handler,      /* [1]  reset */
    NMI_Handler,         /* [2] */
    HardFault_Handler,   /* [3] */
    /* ... remaining exception and IRQ slots ... */
};
```

`Default_Handler` is a `weak` alias so any vector you don't implement still
resolves to something (typically an infinite loop) instead of a link error
or a jump to garbage. This table lives at address `0` via the `.isr_vector`
linker section — module 3-02 covers the linker script and startup code
(`Reset_Handler`'s body: copy `.data`, zero `.bss`, call `main`) in full.

## Simulating register logic in portable C

The bit-manipulation patterns above don't need real hardware to verify —
only the addresses do. This mock harness models the same MODER/ODR
read-modify-write logic against a plain array standing in for memory, and
was compiled and run with `gcc` on this machine:

```c
#include <stdio.h>
#include <stdint.h>
#include <assert.h>

static uint32_t moder, odr;

static void set_mode_output(uint32_t *moder_reg, unsigned pin) {
    *moder_reg &= ~(0x3u << (pin * 2));
    *moder_reg |=  (0x1u << (pin * 2));
}

int main(void) {
    moder = 0xFFFFFFFFu;   /* reset state: all pins analog/reset mode */
    set_mode_output(&moder, 5);
    assert(((moder >> (5 * 2)) & 0x3u) == 0x1u);   /* PA5 now output */
    assert(((moder >> (4 * 2)) & 0x3u) == 0x3u);   /* PA4 untouched */

    odr ^= (1u << 5);
    assert(odr == (1u << 5));
    odr ^= (1u << 5);
    assert(odr == 0);

    printf("register model OK: MODER=0x%08x ODR=0x%08x\n", moder, odr);
    return 0;
}
```

```
$ gcc -Wall -o regtest regtest.c && ./regtest
register model OK: MODER=0xfffff7ff ODR=0x00000000
```

This confirms the bit math is correct in isolation; it says nothing about
whether `0x40020000` is really GPIOA on a given STM32 part — that comes only
from the reference manual for that exact chip.

## Traps specific to bare-metal register work

- **Read-modify-write on shared registers isn't atomic.** `GPIOA_ODR |= bit`
  is a load, an OR, and a store as three separate instructions. If an ISR
  also touches `ODR` between the load and the store, the ISR's change is
  lost. STM32 GPIO provides `BSRR` (bit set/reset register) specifically to
  make single-bit set/clear atomic — always prefer it over read-modify-write
  on `ODR` once an ISR is in the picture (module 3-06).
- **Wrong peripheral base address compiles cleanly.** `0x40020000` vs
  `0x40020400` (GPIOA vs GPIOB on many STM32F4 parts) is just a number to
  the compiler — the bug only shows up as "the wrong LED blinks" or "nothing
  happens" on real hardware.
- **Forgetting the clock enable** produces a write that appears to succeed
  in code but has no effect on the pin — a classic silent bug with no
  compiler warning.
- **Bit-width mismatches**: some registers pack multiple 2-bit or 4-bit
  fields (MODER, AFR); a shift computed with the wrong field width silently
  corrupts an adjacent pin's configuration.

## Cheat sheet

| Concept | Detail |
|---|---|
| Memory-mapped register | `*(volatile T *)ADDRESS` — `volatile` prevents caching/reordering |
| Enable order | Clock enable (`RCC_*ENR`) before touching any peripheral register |
| Reset entry | CPU loads SP from `0x0`, PC from `0x4` — no bootloader code runs first |
| Weak alias | Unimplemented vectors fall back to `Default_Handler` instead of link errors |
| `BSRR` vs `ODR |=` | `BSRR` is a true atomic set/clear; `ODR` read-modify-write is not |
| Toolchain | `arm-none-eabi-gcc` + OpenOCD/ST-Link, not Arduino/ESP-IDF build system |
| Verification here | Portable bit-logic compiled/run with `gcc`; addresses reviewed against ARM/ST docs only |

## Exercise

Write a standalone C file (compilable with plain `gcc`, no ARM toolchain
needed) that models a 32-bit `GPIOx_MODER`-style register as a `uint32_t`
plus a `set_pin_mode(reg, pin, mode)` function supporting mode values `00`
(input), `01` (output), `10` (alternate function), `11` (analog). Add
assertions that setting one pin's mode never disturbs the two bits belonging
to any other pin, then extend it with a `set_pin_output_atomic(bsrr_reg,
pin, level)` function that models `BSRR` (writing to the low 16 bits sets,
the high 16 bits resets) and prove in a comment why this form is
ISR-safe where `ODR |= /|= ~` is not.
