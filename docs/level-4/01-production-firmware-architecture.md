# Production Firmware Architecture

Levels 1-3 built understanding one concept at a time — a task here, a
register there, a bootloader by itself. Shipping firmware means those
pieces coexist in one codebase that a team maintains for years, across
hardware revisions. This module is about the structural decisions that
determine whether that codebase stays workable: layering, hardware
abstraction, and build configuration for multiple product variants from one
source tree.

## The layering problem

Code that mixes "read the raw I2C register" logic directly into
application-level business logic (a control loop, a UI handler) becomes
unportable and untestable — every hardware revision touches application
code, and testing application logic requires real hardware.

```
Application layer      (decides WHAT to do — no register access, no #ifdef HW_REV)
        |
Driver layer            (sensor.h / sensor.c — knows the sensor's protocol)
        |
HAL (Hardware Abstraction Layer)   (i2c_read(), gpio_set() — knows THIS chip's registers)
        |
Silicon
```

```c
/* driver layer — depends only on the HAL interface, never on raw registers */
typedef struct { int (*i2c_read)(uint8_t addr, uint8_t reg, uint8_t *buf, size_t len); } hal_i2c_t;

typedef struct { hal_i2c_t *hal; uint8_t addr; } temp_sensor_t;

int temp_sensor_read_celsius(temp_sensor_t *s, float *out) {
    uint8_t raw[2];
    if (s->hal->i2c_read(s->addr, 0x00, raw, 2) != 0) return -1;
    int16_t counts = (raw[0] << 8) | raw[1];
    *out = counts * 0.0625f;   /* per this sensor's datasheet conversion */
    return 0;
}
```

`temp_sensor_read_celsius` never mentions a peripheral base address — it
depends on the `hal_i2c_t` interface, which a specific board's HAL
implementation provides. Porting to a new MCU means writing a new HAL
implementation; the driver and everything above it is untouched.

## Multiple product variants from one tree

A product line rarely has exactly one hardware revision forever. The
standard pattern is compile-time configuration selecting which HAL/driver
set is linked, not runtime `#ifdef` scattered through application code:

```c
/* board_config.h — one file per variant, selected by the build system */
#if defined(BOARD_REV_A)
  #define TEMP_SENSOR_I2C_ADDR   0x48
  #define HAS_SECONDARY_SENSOR   0
#elif defined(BOARD_REV_B)
  #define TEMP_SENSOR_I2C_ADDR   0x49   /* rev B moved to a different address */
  #define HAS_SECONDARY_SENSOR   1
#endif
```

Application code reads `HAS_SECONDARY_SENSOR` as a feature flag, never a
board name — the distinction matters because a later revision might add the
same sensor via a different mechanism, and code written against "is this
feature present" survives that change; code written against "is this board
Rev B" does not.

## Modeling the layering contract in portable C

The HAL/driver separation is testable without real hardware by substituting
a **fake HAL** — exactly the technique used for unit testing embedded
drivers on a host machine. Compiled and run with `gcc`:

```c
#include <stdio.h>
#include <assert.h>
#include <stdint.h>
#include <string.h>
#include <math.h>

typedef struct { int (*i2c_read)(uint8_t addr, uint8_t reg, uint8_t *buf, size_t len); } hal_i2c_t;
typedef struct { hal_i2c_t *hal; uint8_t addr; } temp_sensor_t;

int temp_sensor_read_celsius(temp_sensor_t *s, float *out) {
    uint8_t raw[2];
    if (s->hal->i2c_read(s->addr, 0x00, raw, 2) != 0) return -1;
    int16_t counts = (int16_t)((raw[0] << 8) | raw[1]);
    *out = counts * 0.0625f;
    return 0;
}

/* fake HAL: returns a fixed byte pattern instead of talking to real I2C */
int fake_i2c_read(uint8_t addr, uint8_t reg, uint8_t *buf, size_t len) {
    (void)addr; (void)reg;
    if (len != 2) return -1;
    buf[0] = 0x01; buf[1] = 0x90;   /* 0x0190 = 400 counts -> 25.0C */
    return 0;
}

int main(void) {
    hal_i2c_t fake_hal = { .i2c_read = fake_i2c_read };
    temp_sensor_t sensor = { .hal = &fake_hal, .addr = 0x48 };

    float celsius;
    int rc = temp_sensor_read_celsius(&sensor, &celsius);
    assert(rc == 0);
    assert(fabsf(celsius - 25.0f) < 0.01f);

    printf("driver-against-fake-HAL test OK: %.2f C\n", celsius);
    return 0;
}
```

This is the actual mechanism behind testing embedded drivers on a laptop
without any target hardware: the driver's contract with the HAL is a plain
C function pointer, and anything satisfying that contract — real I2C
hardware or a fake returning canned bytes — is interchangeable from the
driver's point of view.

## Error handling as an architectural decision, not an afterthought

A driver returning `-1` for every failure loses information the caller
needs to act correctly (retry vs. give up vs. escalate). A small, consistent
error enum used across every layer is worth establishing early:

```c
typedef enum {
    ERR_OK = 0,
    ERR_TIMEOUT,        /* transient — caller may retry */
    ERR_NACK,           /* device not present or busy — caller may retry a few times */
    ERR_BAD_DATA,       /* CRC/range check failed — not a bus problem, don't retry blindly */
    ERR_NOT_INITIALIZED
} err_t;
```

Retrofitting this after a codebase has thousands of `return -1;` call sites
is expensive — it is one of the few decisions genuinely cheaper to make
correctly on day one than to fix later.

## Traps in production firmware architecture

- **Leaky abstraction**: a HAL function that returns a chip-specific error
  code directly (instead of translating to a portable `err_t`) forces every
  caller up the stack to know about that chip's quirks — defeats the point
  of the layer.
- **God headers**: one `board.h` `#include`d everywhere that pulls in every
  peripheral's registers turns any change into a full rebuild and hides real
  dependencies between modules.
- **Runtime board detection with no fallback**: reading a board-ID pin to
  select behavior at runtime is more flexible than compile-time
  `#if`, but every code path needs an explicit "unknown board" case — silent
  fallthrough to whichever `#if` branch happened to compile first is a
  field-failure waiting to happen on a board revision nobody tested against.
- **Testing only the top layer**: application-level tests that always run
  against a fake HAL can hide integration bugs that only appear against real
  timing/electrical behavior — host-side tests catch logic bugs, they do not
  replace hardware-in-the-loop testing (module 4-05).

## How It Actually Works

**Why a function-pointer HAL interface is what makes the layering real
rather than aspirational**: `hal_i2c_t`'s `i2c_read` field is a plain C
function pointer, and at the machine level, calling `s->hal->i2c_read(...)`
compiles to loading that pointer from memory and performing an *indirect*
call — jumping to whatever address the pointer currently holds, resolved at
runtime, not at link time. This is the exact mechanism (not a testing
convenience layered on top, but the underlying compiled behavior) that lets
`fake_i2c_read` and a real hardware implementation be genuinely
interchangeable: the driver code's compiled instructions never encode a
specific target address for that call, only "whatever `s->hal->i2c_read`
currently points to." A direct call to a real register-access function,
by contrast, is resolved by the linker to one fixed address baked into the
binary — which is precisely why layering through function pointers (or,
equivalently in C++, virtual dispatch through a vtable, itself just a
compiler-managed array of the same kind of function pointers) is what
actually enables host-side testing, not merely a stylistic choice.

**Why "leaky abstraction" is a real, mechanical failure and not just a code
smell**: when a HAL function returns a chip-specific error code (say, an
STM32 HAL's `HAL_I2C_ERROR_AF`) directly instead of translating it into a
portable `err_t`, every caller up the stack that wants to branch on the
*meaning* of that error (transient vs. permanent, retry vs. give up) has to
either duplicate the chip-specific knowledge of what that constant means, or
`#include` the chip's own HAL headers just to get the constant's definition
— at which point the "portable" driver layer now has a compile-time
dependency on one specific vendor's headers, and porting to a different MCU
means every one of those call sites needs to change, not just the HAL
implementation underneath. The translation at the HAL boundary is what
actually contains that dependency to one place; skipping it doesn't remove
the coupling, it just spreads it silently through every layer that touches
the return value.

**Why compile-time board selection (`#if defined(BOARD_REV_A)`) avoids a
class of bug runtime detection can't**: with compile-time selection, code for
a board variant that was never built simply isn't present in that build's
binary at all — there's no code path to accidentally execute, because the
preprocessor removed it before the compiler ever saw it. Runtime board
detection (reading an ID pin, branching on its value) keeps *all* variants'
code linked into every build, and the correctness of the branch depends on
that ID pin being read correctly at the right point in boot, on every unit,
forever — a hardware assembly defect or a bit of noise on that pin at the
wrong moment silently selects the wrong code path in a binary that "compiled
fine" for every variant, a failure mode that simply cannot occur when the
choice was resolved by the preprocessor before the binary was even linked.

## Cheat sheet

| Concept | Detail |
|---|---|
| HAL | Bottom layer — knows chip registers, exposes a portable function-pointer interface |
| Driver | Knows a specific peripheral's protocol, depends only on the HAL interface |
| Application | Business logic — no register access, no board `#ifdef` |
| Feature flags | `HAS_X` compile-time flags, not board-name checks, in application code |
| Fake HAL | Swap in a canned-response HAL to unit-test drivers on a host machine, no hardware needed |
| Consistent error enum | Decide once, project-wide — expensive to retrofit later |
| Verification here | Driver-against-fake-HAL logic compiled/run with `gcc`; real hardware timing not represented |

## Exercise

Extend the fake-HAL test harness with a `fake_i2c_read` that can be
configured to simulate a NACK (return nonzero) on the Nth call, and update
`temp_sensor_read_celsius` to accept a `retry_count` parameter that retries
on `ERR_NACK`-equivalent failures up to that count before giving up. Write
assertions for: success on the first try, success after one simulated NACK
and one retry, and permanent failure when NACKs exceed the retry budget.
Compile and run with `gcc`, and in a comment state which of these three
scenarios would be expensive or impossible to test against real hardware on
demand (hint: consider how you'd reliably force a real sensor to NACK on
exactly the second call).
