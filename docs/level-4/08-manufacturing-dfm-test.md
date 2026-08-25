# Manufacturing, DFM & Factory Test

Module 3-07 designed a single, hand-assembled board. This module is about
what changes when a design has to be built by a contract manufacturer at
volume: **Design for Manufacturing (DFM)** — layout choices that make a
board reliably assemblable by machine — and factory test firmware, the code
that runs once on every unit before it ships, never again in the field. This
is manual technical review of manufacturing practice; no board from this
course was actually built at a CM, and that's stated plainly wherever a
claim depends on real fab/assembly-line behavior.

## DFM is a different discipline from schematic/electrical design

A board can be electrically perfect and still be expensive or unreliable to
build:

- **Component placement for pick-and-place**: parts need adequate spacing
  for a nozzle to place them without disturbing neighbors, and a
  consistent orientation reference (silkscreen pin-1 marker, matching the
  actual pin-1 pad) that the CM's placement program relies on.
- **Panelization**: boards are usually built in a panel (an array of many
  copies) with breakout tabs or v-score lines, then separated after
  assembly — a board designed without panelization in mind (components too
  close to the edge, no tooling holes) costs more per unit or requires
  layout rework before it can be manufactured at all.
- **Solder paste stencil design**: a good stencil aperture design (matching
  pad size and adding paste-reduction for fine-pitch parts) meaningfully
  affects first-pass yield — an unmodified 1:1 stencil-to-pad ratio on
  fine-pitch ICs is a well-known source of solder bridging.
- **Testability**: every net a factory test needs to probe should have an
  accessible test point — retrofitting this after layout is far more
  expensive than including it from the start (module 3-07's test-point
  point applies here at greater stakes).

## Design for Test: the boundary-scan and bed-of-nails reality

Two dominant approaches to factory electrical test:

- **Bed-of-nails / flying probe**: physical pogo pins contact test points on
  the board to inject signals and measure responses — requires the
  test-point accessibility called out above, and works for verifying
  solder joints, shorts, and basic circuit function.
- **Boundary-scan (JTAG)**: for parts that support it, lets a test fixture
  verify pin-level connectivity through the chip's own scan chain without
  needing physical probe access to every net — valuable on dense boards
  where physical test points for every signal aren't feasible.

Neither replaces functional test — a board that passes electrical
continuity test can still fail to run correct firmware, which is why
factory test firmware (below) is a separate, additional stage.

## Factory test firmware: not the same firmware that ships

A dedicated factory test image, flashed once during manufacturing and
overwritten by the final production image before the unit ships, typically
exercises every peripheral in a way normal operation never does — testing
things application firmware has no reason to check on every boot:

```c
/* conceptual factory test sequence — architecture reviewed, not run on real hardware */
typedef struct { const char *name; int (*test_fn)(void); } factory_test_t;

int test_i2c_sensor_present(void)  { /* probe expected I2C address, expect ACK */ return 0; }
int test_flash_write_read(void)    { /* write pattern, read back, verify */ return 0; }
int test_led_visual_confirm(void)  { /* blink pattern; operator/camera confirms */ return 0; }
int test_button_input(void)        { /* wait for operator press within timeout */ return 0; }

factory_test_t factory_tests[] = {
    { "i2c_sensor",   test_i2c_sensor_present },
    { "flash_wr",     test_flash_write_read },
    { "led_visual",   test_led_visual_confirm },
    { "button_input", test_button_input },
};

/* runs every test, reports a single pass/fail plus per-test detail over UART
   for the factory operator/fixture to log — every unit gets a permanent
   test record tied to its serial number */
int run_factory_tests(void) {
    int all_pass = 1;
    for (size_t i = 0; i < sizeof(factory_tests)/sizeof(factory_tests[0]); i++) {
        int rc = factory_tests[i].test_fn();
        report_test_result(factory_tests[i].name, rc);
        if (rc != 0) all_pass = 0;
    }
    return all_pass;
}
```

Every unit's serial number and test result should be logged permanently
(not just pass/fail on a screen the operator glances at) — a field failure
investigation months later often needs to know exactly which factory tests
that specific serial number passed and what its measured values were, not
just that it shipped.

## Modeling the factory test aggregation logic

The pass/fail aggregation and reporting logic is pure control flow,
compiled and run with `gcc` against fake test functions standing in for
real hardware probes:

```c
#include <stdio.h>
#include <assert.h>
#include <string.h>

typedef struct { const char *name; int (*test_fn)(void); } factory_test_t;

int fake_pass(void) { return 0; }
int fake_fail(void) { return -1; }

typedef struct { char name[32]; int rc; } test_record_t;
static test_record_t records[16];
static int record_count = 0;

void report_test_result(const char *name, int rc) {
    strncpy(records[record_count].name, name, sizeof(records[0].name) - 1);
    records[record_count].rc = rc;
    record_count++;
}

int run_factory_tests(factory_test_t *tests, size_t n) {
    int all_pass = 1;
    for (size_t i = 0; i < n; i++) {
        int rc = tests[i].test_fn();
        report_test_result(tests[i].name, rc);
        if (rc != 0) all_pass = 0;
    }
    return all_pass;
}

int main(void) {
    factory_test_t all_ok[] = { {"a", fake_pass}, {"b", fake_pass} };
    record_count = 0;
    assert(run_factory_tests(all_ok, 2) == 1);
    assert(record_count == 2);

    factory_test_t one_bad[] = { {"a", fake_pass}, {"b", fake_fail}, {"c", fake_pass} };
    record_count = 0;
    assert(run_factory_tests(one_bad, 3) == 0);       /* overall fail */
    assert(record_count == 3);                         /* but ALL tests still ran and logged */
    assert(records[1].rc == -1);

    printf("factory test aggregation model OK\n");
    return 0;
}
```

The assertion that `record_count == 3` even when test `b` fails encodes a
real, deliberate design choice: a factory test sequence should run every
test and log every result rather than stopping at the first failure —
diagnosing a real hardware fault usually benefits from knowing everything
that failed, not just the first thing.

## Traps in manufacturing and DFM

- **No test points for signals the factory test actually needs**: designed
  in module 3-07's terms, but the stakes are higher at volume — a missing
  test point means either an expensive layout revision or a compromised
  factory test for the entire production run.
- **Factory test firmware accidentally shipped to customers**: a build
  pipeline that doesn't clearly separate factory-test and production images
  can ship a unit still running diagnostic firmware — embarrassing at best,
  a real functional or security problem at worst.
- **Stopping at first test failure**: as above, loses diagnostic information
  that would help root-cause a real yield problem at the CM.
- **No serial-number-to-test-record traceability**: without it, a field
  return months later can't be cross-referenced against what passed at
  manufacturing time, turning every field failure investigation into
  guesswork.

## Cheat sheet

| Concept | Detail |
|---|---|
| DFM | Layout choices for reliable machine assembly — placement, panelization, stencil design |
| Bed-of-nails / flying probe | Physical test-point contact for electrical test — needs accessible test points |
| Boundary-scan (JTAG) | Pin-level connectivity test through the chip's own scan chain, no physical probe needed |
| Factory test firmware | Separate image, exercises every peripheral once, never ships to customers |
| Run-all-tests policy | Log every result even after a failure — preserves diagnostic information |
| Serial-number traceability | Permanent record linking each unit to its exact factory test results |
| Verification here | Test-aggregation/logging logic compiled/run with `gcc`; real DFM/assembly-line behavior reviewed against manufacturing practice only |

## Exercise

Extend the factory test harness with a `test_result_t` that includes a
numeric measurement (not just pass/fail) for tests where that matters —
e.g. a supply voltage reading that must fall within a tolerance band rather
than a simple pass/fail probe. Add a test that reports 3.3V ± 0.1V
tolerance, write assertions for a measurement inside the band (pass), just
outside it (fail), and exactly at the boundary (decide and justify in a
comment whether boundary-inclusive is the right choice for a voltage
tolerance check). Compile and run with `gcc`.
