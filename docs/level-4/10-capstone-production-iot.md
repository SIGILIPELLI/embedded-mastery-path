# Capstone — Production IoT Product

This capstone combines every module across Level 4 (and, through it, every
prior level) into one design: taking the Level 3 sensor board project from
a working prototype to a fleet-deployable production IoT product — layered
firmware, secure updates, edge inference, safety-conscious validation,
tested in CI, manufacturable at volume, and managed as a fleet. As with
every hardware-adjacent module in this course, no physical unit was built
or shipped; this is an architecture-level design combining reviewed,
citable practice with every piece of portable logic actually compiled and
run.

## System architecture

```
                     [ Fleet backend / device cloud ]
                              ^ TLS + device cert (4-09)
                              |
   +----------------------------------------------------+
   |  Production Firmware (4-01 layering)                |
   |                                                       |
   |  Application layer                                   |
   |    - reads sensor via Driver -> HAL (4-01)            |
   |    - runs quantized on-device anomaly model (4-03)     |
   |    - validates readings: range + plausibility (4-06)   |
   |    - aggregates telemetry summaries (4-09)              |
   |    - adapts duty cycle to power state (4-07)             |
   |                                                            |
   |  Bootloader: dual-bank, signature-verified (3-09, 4-02)     |
   |  HAL/drivers: I2C+DMA sensor read, UART report (3-04, 3-08)  |
   +------------------------------------------------------------+
                              |
                    [ Sensor board hardware ]
                    DFM-reviewed layout, factory
                    test firmware at manufacture (4-08)
```

## Bringing modules together: the sensor-read-to-cloud path

```c
/* application layer — orchestrates modules from every earlier level;
   depends only on driver/HAL interfaces, per module 4-01's layering rule */

#include <stdint.h>
#include <stdbool.h>

typedef struct { float celsius; uint32_t timestamp_ms; } sample_t;

extern int  temp_sensor_read_celsius(void *sensor, float *out);       /* 4-01 driver */
extern int  reading_is_plausible(float new_val, float last_val,
                                  uint32_t dt_ms, float max_rate);      /* 4-06 safety check */
extern void telemetry_summary_add(void *summary, float value);         /* 4-09 aggregation */
extern int8_t anomaly_model_infer(const int8_t *quantized_input);      /* 4-03 edge inference */
extern power_state_t classify_power_state(double stored_uj, double cap_uj); /* 4-07 power state */

typedef struct {
    float last_reading;
    uint32_t last_timestamp_ms;
    void *telemetry_summary;
    void *sensor;
} app_state_t;

/* one full cycle: read -> validate -> infer -> aggregate -> adapt */
int app_cycle(app_state_t *st, double stored_energy_uj, double capacity_uj,
              uint32_t now_ms) {
    float reading;
    if (temp_sensor_read_celsius(st->sensor, &reading) != 0) {
        return -1;   /* sensor read failure: caller decides retry/escalate policy */
    }

    uint32_t dt = now_ms - st->last_timestamp_ms;
    if (!reading_is_plausible(reading, st->last_reading, dt, 5.0f)) {
        return -2;   /* implausible jump: flag rather than silently trust it */
    }

    telemetry_summary_add(st->telemetry_summary, reading);

    st->last_reading = reading;
    st->last_timestamp_ms = now_ms;

    power_state_t power = classify_power_state(stored_energy_uj, capacity_uj);
    (void)power;   /* real firmware uses this to pick the next sleep interval (4-07) */

    return 0;
}
```

## Verifying the orchestration logic against fakes

The application-layer control flow is testable exactly the way module
4-01 introduced — real sensor/model/telemetry functions replaced with
fakes that let the *logic* (not the hardware) be verified with `gcc`:

```c
#include <stdio.h>
#include <assert.h>
#include <stdint.h>
#include <math.h>

typedef struct { float celsius; uint32_t timestamp_ms; } sample_t;
typedef enum { POWER_CRITICAL, POWER_LOW, POWER_NORMAL, POWER_ABUNDANT } power_state_t;

static float g_fake_reading = 21.0f;
static int g_fake_sensor_fail = 0;

int temp_sensor_read_celsius(void *sensor, float *out) {
    (void)sensor;
    if (g_fake_sensor_fail) return -1;
    *out = g_fake_reading;
    return 0;
}
int reading_is_plausible(float new_val, float last_val, uint32_t dt_ms, float max_rate) {
    float dt_s = dt_ms / 1000.0f;
    return fabsf(new_val - last_val) <= max_rate * dt_s;
}
static float g_summary_sum = 0; static int g_summary_count = 0;
void telemetry_summary_add(void *summary, float value) { (void)summary; g_summary_sum += value; g_summary_count++; }
power_state_t classify_power_state(double stored_uj, double cap_uj) {
    double frac = stored_uj / cap_uj;
    return frac < 0.1 ? POWER_CRITICAL : POWER_NORMAL;
}

typedef struct { float last_reading; uint32_t last_timestamp_ms; void *telemetry_summary; void *sensor; } app_state_t;

int app_cycle(app_state_t *st, double stored_energy_uj, double capacity_uj, uint32_t now_ms) {
    float reading;
    if (temp_sensor_read_celsius(st->sensor, &reading) != 0) return -1;
    uint32_t dt = now_ms - st->last_timestamp_ms;
    if (!reading_is_plausible(reading, st->last_reading, dt, 5.0f)) return -2;
    telemetry_summary_add(st->telemetry_summary, reading);
    st->last_reading = reading;
    st->last_timestamp_ms = now_ms;
    power_state_t power = classify_power_state(stored_energy_uj, capacity_uj);
    (void)power;
    return 0;
}

int main(void) {
    app_state_t st = { .last_reading = 20.0f, .last_timestamp_ms = 0 };

    /* normal cycle succeeds */
    g_fake_reading = 20.5f; g_fake_sensor_fail = 0;
    assert(app_cycle(&st, 500.0, 1000.0, 1000) == 0);
    assert(g_summary_count == 1);

    /* sensor failure propagates as -1 */
    g_fake_sensor_fail = 1;
    assert(app_cycle(&st, 500.0, 1000.0, 2000) == -1);
    g_fake_sensor_fail = 0;

    /* implausible jump rejected as -2, telemetry NOT updated */
    g_fake_reading = 200.0f;
    int rc = app_cycle(&st, 500.0, 1000.0, 3000);
    assert(rc == -2);
    assert(g_summary_count == 1);   /* still 1 — the bad reading was never aggregated */

    printf("capstone orchestration model OK\n");
    return 0;
}
```

## Traps this capstone exercises across the whole course

- **Layering discipline breaking down under integration pressure**: it's
  tempting, when wiring modules together for the first time, to let the
  application layer reach past the driver interface "just this once" for a
  quick fix — exactly the leak module 4-01 warned against, and exactly
  where it tends to actually happen in real projects.
- **Safety checks silently skipped on the "happy path"**: the plausibility
  check must run on every cycle, not just when a developer remembers to
  test it — the fake-sensor test above deliberately checks that a bad
  reading never reaches telemetry, not just that good readings do.
- **Power-state logic computed but never acted on**: the orchestration
  above computes `power` and discards it — a reminder that this capstone's
  sketch is architecture, not a finished product; real firmware must
  actually feed that state into the sleep-interval decision from module
  4-07, not just compute and discard it.
- **Testing only individual modules, never the orchestration**: each
  module's own tests (4-01 through 4-09) verify that module in isolation;
  this capstone's orchestration test is what catches bugs in how they're
  wired together, which neither side's isolated tests can see.

## Cheat sheet

| Module | Role in the capstone |
|---|---|
| 4-01 | Layering: application depends only on driver/HAL interfaces |
| 4-02 | Bootloader signature verification + staged rollout for updates |
| 4-03 | Quantized on-device anomaly inference on sensor readings |
| 4-04 | (If wireless) link budget and packet design for the report path |
| 4-05 | HIL tier catches what this capstone's host-side tests structurally cannot |
| 4-06 | Plausibility/range validation before any reading is trusted or aggregated |
| 4-07 | Power-state classification driving adaptive duty cycling |
| 4-08 | DFM-reviewed layout and factory test firmware at manufacturing time |
| 4-09 | Telemetry aggregation and device identity for fleet reporting |

## Stretch goals

- Wire the discarded `power` state from `app_cycle` into an actual
  `sleep_interval_ms_for_state` call (module 4-07) and extend the test to
  assert the correct interval is chosen as simulated stored energy drops
  across a sequence of cycles.
- Add a fake OTA-check step to the cycle that calls a module-4-02-style
  `should_install` function once every N cycles, and test that an
  in-progress sensor read is never interrupted mid-cycle by an update check
  (an ordering requirement, not just an existence check).
- Extend the orchestration test to simulate a full canary-rollout scenario
  (module 4-02/4-09): a fleet of simulated `app_state_t` instances, a
  fraction receiving a "bad" config that fails `validate_config`, and assert
  that the fleet-level logic halts further rollout once the simulated
  failure rate crosses a threshold.
- Take this capstone's architecture diagram and produce a one-page DFM
  checklist (module 4-08 style) for the physical board this firmware would
  run on, listing every test point the factory test firmware above would
  need physical access to.
