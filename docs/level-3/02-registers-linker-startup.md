# Registers, Linker Scripts & Startup Code

Module 3-01 wrote to registers and referenced `_estack` and `Reset_Handler`
without explaining where they come from. Those symbols are defined by two
files every bare-metal project needs and every framework normally hides: a
**linker script** (`.ld`) that tells the linker where flash and RAM live and
how to lay out sections, and **startup code** (`startup_stm32.c` or `.s`)
that runs between reset and `main()` to make C's assumptions about global
variables actually true.

## What C assumes that hardware doesn't provide for free

C guarantees that a zero-initialized global starts at `0`, and an
initialized global starts at its initializer value:

```c
int counter = 0;        /* .bss  — zero-initialized */
int max_retries = 5;    /* .data — non-zero initialized */
```

On flash-based microcontrollers, `.data`'s *initial value* (`5`) is stored in
flash alongside your code — flash is not writable at runtime the way RAM is,
so the variable itself must live in RAM, but its startup value has to come
from somewhere. Nothing does this automatically. Startup code must **copy**
`.data` from flash to RAM and **zero** `.bss` in RAM before `main()` runs, or
`max_retries` reads as garbage.

## The linker script: describing memory to the linker

```ld
MEMORY
{
  FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
  RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS
{
  .isr_vector : { *(.isr_vector) } > FLASH

  .text : {
    *(.text*)
    *(.rodata*)
  } > FLASH

  .data : {
    _sdata = .;
    *(.data*)
    _edata = .;
  } > RAM AT> FLASH          /* lives in RAM, initial values loaded from FLASH */

  .bss : {
    _sbss = .;
    *(.bss*)
    _ebss = .;
  } > RAM

  _estack = ORIGIN(RAM) + LENGTH(RAM);   /* stack grows down from top of RAM */
}
```

`AT> FLASH` is the detail that makes `.data` initialization possible: the
section's *run-time* address is in RAM (where code reads and writes it), but
its *load-time* address — where the linker actually places the initial
bytes — is in FLASH. The linker exposes both addresses as symbols
(`LOADADDR(.data)` and `.data`'s own address), which startup code uses to
copy one to the other.

## Startup code: making the assumptions true

```c
extern uint32_t _sdata, _edata, _sbss, _ebss;
extern uint32_t _sidata;      /* LOADADDR(.data) — where init values live in flash */
extern int main(void);

void Reset_Handler(void) {
    uint32_t *src = &_sidata;
    uint32_t *dst = &_sdata;
    while (dst < &_edata) *dst++ = *src++;      /* copy .data: flash -> RAM */

    dst = &_sbss;
    while (dst < &_ebss) *dst++ = 0;            /* zero .bss */

    main();
    while (1) { /* main() must never return on bare metal */ }
}
```

This is the single piece of code every C runtime on every platform has in
some form — on a hosted OS it's part of the C library's `_start`; here you
write it. Skipping it doesn't produce a compile error or even a crash
necessarily — it produces globals that silently hold whatever bytes were
already in RAM at power-on, which is a uniquely painful class of bug because
the code *looks* correct.

## Simulating the copy/zero logic in portable C

The mechanics — not the real flash/RAM split — are verified below by
modeling "flash" and "RAM" as two arrays and running the same loop shape as
`Reset_Handler`. Compiled and run with `gcc`:

```c
#include <stdio.h>
#include <string.h>
#include <assert.h>

int main(void) {
    unsigned char flash_image[8] = {5, 0, 0, 0, 7, 0, 0, 0}; /* two uint32 inits: 5, 7 */
    unsigned char ram[16];
    memset(ram, 0xAA, sizeof(ram));   /* simulate power-on garbage */

    /* copy .data (first 8 bytes of "ram") from "flash" */
    memcpy(ram, flash_image, 8);
    /* zero .bss (remaining 8 bytes) */
    memset(ram + 8, 0, 8);

    int *data_vars = (int *)ram;
    int *bss_vars  = (int *)(ram + 8);
    assert(data_vars[0] == 5 && data_vars[1] == 7);
    assert(bss_vars[0] == 0 && bss_vars[1] == 0);

    printf("startup simulation OK: data={%d,%d} bss={%d,%d}\n",
           data_vars[0], data_vars[1], bss_vars[0], bss_vars[1]);
    return 0;
}
```

```
$ gcc -Wall -o startupsim startupsim.c && ./startupsim
startup simulation OK: data={5,7} bss={0,0}
```

## Traps in linker scripts and startup code

- **Forgetting `AT> FLASH`** on `.data` makes the linker place both the
  section *and* its load image in RAM — it will link cleanly, then every
  `.data` value reads as whatever RAM happened to hold at boot, because
  nothing ever copies from flash.
- **`_estack` pointing outside RAM** (off-by-one on `ORIGIN + LENGTH`,
  or reusing a linker symbol name that collides with another script) crashes
  before a single line of `main()` runs, with no diagnostic beyond "nothing
  happens."
- **Stack/heap collision**: the stack grows down from `_estack`, the heap
  (if you use `malloc`) grows up from the end of `.bss`. Neither the linker
  nor the startup code stops them from meeting in the middle — this is a
  purely runtime failure mode with no static check.
- **Missing `KEEP()` on the vector table section** lets an aggressive linker
  garbage-collect `.isr_vector` because nothing in `.text` visibly
  references it, producing a binary that boots without a vector table at
  address `0`.

## Cheat sheet

| Concept | Detail |
|---|---|
| `.text`/`.rodata` | Code and constants — stay in FLASH, never copied |
| `.data` | Non-zero globals — live in RAM, load image in FLASH, copied by startup code |
| `.bss` | Zero-initialized globals — live in RAM only, zeroed by startup code, occupy no flash |
| `AT> FLASH` | Tells linker the *load* address differs from the *run* address |
| `_sdata`/`_edata`/`_sbss`/`_ebss` | Linker-defined symbols marking section boundaries, used by startup code |
| Stack direction | Grows **down** from `_estack` (top of RAM) |
| `KEEP()` | Prevents the linker from discarding a section nothing references directly (e.g. `.isr_vector`) |
| Verification here | Copy/zero logic simulated and run with `gcc`; real flash/RAM layout not testable off-chip |

## Exercise

Write a portable C program that models a linker's `.data`/`.bss` split with
three arrays: `flash_image[]` (holds the `.data` load values), `ram[]`
(the target), and a `struct { size_t data_off, data_len, bss_off, bss_len; }`
describing the layout — then write a single function `apply_startup(...)`
that performs the copy-then-zero using only those offsets (no hardcoded
sizes). Compile and run it with assertions proving arbitrary `.data`/`.bss`
sizes work, then add a deliberate bug — swap the copy and zero order — and
explain in a comment exactly which values end up wrong and why order matters
here even though the two operations touch disjoint memory.
