# HIL Testing & CI for Firmware

Module 4-01's fake-HAL pattern lets driver and application logic run as
host-side unit tests — fast, but blind to real timing, electrical noise, and
actual chip behavior. **Hardware-in-the-Loop (HIL)** testing closes that
gap: real target hardware, automated in a CI pipeline, exercised by test
code the same way a human would with a bench supply and a logic analyzer,
but repeatable and run on every commit.

## The testing pyramid for embedded firmware

```
        /\
       /  \    HIL tests (real hardware, slow, few, catches timing/electrical bugs)
      /----\
     /      \  Integration tests (target simulator/emulator, e.g. QEMU, medium speed)
    /--------\
   /          \ Unit tests (fake HAL, host machine, fast, many — module 4-01's pattern)
  /____________\
```

Most of a firmware test suite should be unit tests, for the same reason
this applies everywhere: they're fast enough to run on every save, and catch
the majority of logic bugs cheaply. HIL tests are the smallest, slowest tier
— reserved for what genuinely needs real silicon: timing-sensitive
peripheral behavior, actual power measurements, and end-to-end validation
that the full stack, including the parts no simulator models, actually
works.

## What a HIL rig actually automates

```
CI runner
   |
   +-- flashes firmware to target board via ST-Link/OpenOCD
   +-- controls a programmable power supply (measure current, inject brownouts)
   +-- reads target UART output over USB-serial
   +-- controls a GPIO relay/multiplexer to simulate sensor inputs
   +-- asserts on captured UART output / measured current / GPIO state
```

A HIL test script (Python, commonly, driving the lab equipment over USB/
serial/GPIB) treats the target board as a black box exercised the same way
a person at a bench would, but scripted:

```python
# conceptual HIL test — talks to real hardware, not runnable without the rig
def test_low_power_current_draw():
    flash_firmware("build/sensor_node.elf")
    reset_target()
    time.sleep(2)                      # let it settle into its sleep cycle
    current_ma = power_supply.measure_current_ma()
    assert current_ma < 0.05, f"expected <50uA average, measured {current_ma}mA"

def test_uart_report_format():
    flash_firmware("build/sensor_node.elf")
    reset_target()
    line = uart.read_line(timeout_s=15)   # wait for one sample report
    assert line.startswith("0xAA")        # module 3-08's framing byte
```

This is exactly the kind of thing module 3-05's battery-life arithmetic
cannot verify — the arithmetic proves the *design* should draw 50 µA
average; only a real measurement on real hardware proves the *implementation*
actually does.

## What CI can verify without real hardware, and how far that goes

The parts of a firmware pipeline runnable in ordinary CI (GitHub Actions,
etc, no lab equipment attached):

```yaml
# conceptual CI stages
build:        arm-none-eabi-gcc build of the actual firmware image
static-analysis:  cppcheck / clang-tidy / MISRA checker (module 4-06)
unit-test:    host-gcc build of the SAME driver code against fake HALs (module 4-01)
size-check:   fail if the built image exceeds flash/RAM budget
# HIL stage runs on a separate self-hosted runner with real boards attached,
# typically gated to run on merge to main rather than every PR, given cost/rig availability
```

The size-check stage is worth calling out specifically: a firmware image
that silently grows past its flash budget over many small commits is a
common, avoidable failure mode that a simple `arm-none-eabi-size` check in
CI catches immediately, for free, on every build.

## Modeling a CI-style size/health check in portable C

While a real build-size gate needs the actual toolchain, the *policy logic*
— pass/fail decision against a budget — is pure logic and testable here:

```c
#include <stdio.h>
#include <assert.h>

typedef struct { const char *name; long flash_bytes, ram_bytes; } build_report_t;
typedef struct { long flash_budget, ram_budget; } budget_t;

/* returns 0 if within budget, negative error code otherwise */
int check_budget(build_report_t report, budget_t budget) {
    if (report.flash_bytes > budget.flash_budget) return -1;
    if (report.ram_bytes   > budget.ram_budget)   return -2;
    return 0;
}

int main(void) {
    budget_t budget = { .flash_budget = 128 * 1024, .ram_budget = 32 * 1024 };

    build_report_t ok_build   = { "sensor_node", 100 * 1024, 20 * 1024 };
    build_report_t over_flash = { "sensor_node", 130 * 1024, 20 * 1024 };
    build_report_t over_ram   = { "sensor_node", 100 * 1024, 40 * 1024 };

    assert(check_budget(ok_build, budget) == 0);
    assert(check_budget(over_flash, budget) == -1);
    assert(check_budget(over_ram, budget) == -2);

    printf("CI size-gate policy model OK\n");
    return 0;
}
```

## Traps in HIL/CI for firmware

- **Flaky HIL rigs treated as flaky tests**: a HIL failure that only
  reproduces intermittently is often a real hardware timing bug (a race the
  fake-HAL unit tests structurally cannot catch), not test infrastructure
  noise — silencing/retrying past it hides exactly the class of bug HIL
  exists to find.
- **No hardware reset between tests**: state leaking from one HIL test into
  the next (a peripheral left mid-transaction, a persisted flag) produces
  order-dependent test results that pass or fail depending on what ran
  before them.
- **Running every test tier on every commit regardless of cost**: HIL rigs
  are a scarce, slow resource; gating them to merge-to-main (or a manual
  trigger) while running the fast unit-test tier on every push is a
  deliberate, common tradeoff, not a compromise on quality.
- **Testing only the happy path on real hardware**: HIL's unique value is
  testing conditions a simulator can't — brownout injection, real timing
  margins, actual current draw — under-using the rig for only "does it boot"
  wastes its most useful property.

## Cheat sheet

| Concept | Detail |
|---|---|
| Testing pyramid | Mostly host-side unit tests (fake HAL), fewer integration/emulator tests, fewest HIL tests |
| HIL | Real target hardware, automated (flash, power, UART, GPIO) — catches timing/electrical bugs simulators can't |
| CI without hardware | Build, static analysis, host unit tests, flash/RAM size gate |
| Size gate | Cheap, automatic protection against silent flash/RAM budget creep |
| HIL scope | Timing margins, real power measurement, end-to-end — not a replacement for fast unit tests |
| Verification here | Size-budget policy logic compiled/run with `gcc`; HIL rig automation is architecture reviewed, not executed (no rig available) |

## Exercise

Extend `check_budget` to accept a *history* of builds (an array of
`build_report_t`) and compute whether flash usage is trending upward at a
rate that will breach budget within N future builds if the trend continues
(simple linear extrapolation from the last few data points is enough) —
returning a warning distinct from an outright failure. Write assertions for
a flat trend (no warning), a slow creeping trend (warning but not yet
failing), and a trend already over budget (failure). Compile and run with
`gcc`, and in a comment explain why catching the *trend* early is more
valuable in a real CI pipeline than only gating on the current build's
absolute numbers.
