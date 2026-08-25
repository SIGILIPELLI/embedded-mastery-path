# Ultra-Low-Power & Energy Harvesting

Module 3-05 covered sleep modes and duty-cycle arithmetic for a
battery-powered device. This module goes further: designs that run
indefinitely on harvested energy (solar, thermal, vibration) instead of a
finite battery, where the power budget isn't "how long until it dies" but
"does average harvested power exceed average consumed power, ever, under
worst-case conditions."

## The harvesting design problem is fundamentally different

A battery-powered design optimizes for **total energy** — minimize
consumption, maximize a fixed budget's lifetime. A harvesting design
optimizes for **power balance** — the system must never consume, on
average and even during worst-case harvesting conditions (night, no wind,
no vibration), more than it harvests, or a storage element (a supercapacitor
or small rechargeable cell) eventually depletes regardless of how efficient
individual operations are.

```c
#include <stdio.h>
#include <math.h>

/* returns 1 if the design survives indefinitely under the given worst-case
   harvesting scenario, 0 if average consumption exceeds average harvest */
int power_budget_sustainable(double avg_harvest_uw, double avg_consume_uw) {
    return avg_harvest_uw >= avg_consume_uw;
}

/* time until a storage cap depletes if harvest can't keep up, given its
   usable energy and the (negative) net power deficit */
double time_to_depletion_hours(double stored_energy_uj, double deficit_uw) {
    if (deficit_uw <= 0.0) return INFINITY;         /* harvest keeps up: never depletes */
    double seconds = stored_energy_uj / deficit_uw;
    return seconds / 3600.0;
}
```

## Adaptive duty cycling: consumption that responds to available energy

A harvesting design that duty-cycles at a fixed rate regardless of current
harvest conditions either wastes available energy (too conservative when the
sun is out) or drains its storage (too aggressive at night) — the standard
approach is to sample harvested energy level and adjust the operating duty
cycle accordingly:

```c
typedef enum { POWER_CRITICAL, POWER_LOW, POWER_NORMAL, POWER_ABUNDANT } power_state_t;

power_state_t classify_power_state(double stored_uj, double capacity_uj) {
    double frac = stored_uj / capacity_uj;
    if (frac < 0.10) return POWER_CRITICAL;   /* only the most essential task runs */
    if (frac < 0.30) return POWER_LOW;         /* reduce sample rate, skip radio */
    if (frac < 0.70) return POWER_NORMAL;      /* normal operating duty cycle */
    return POWER_ABUNDANT;                     /* can afford extra work: more frequent radio, etc */
}

uint32_t sleep_interval_ms_for_state(power_state_t state) {
    switch (state) {
        case POWER_CRITICAL: return 3600000u;   /* once an hour, minimal work */
        case POWER_LOW:       return 600000u;    /* every 10 minutes */
        case POWER_NORMAL:    return 60000u;     /* every minute */
        case POWER_ABUNDANT:  return 10000u;     /* every 10 seconds */
    }
    return 60000u;
}
```

This is the mechanism behind every "smart" harvesting-powered sensor —
graceful degradation under scarcity rather than a hard cutoff, and the
storage element (supercapacitor, typically, for its charge-cycle longevity
versus a rechargeable chemical cell) never fully depletes under the design's
own logic unless harvest is truly and persistently below the critical-state
consumption floor.

## Supercapacitor voltage, not battery voltage, drives regulator design

A Li-ion battery holds a roughly flat voltage across most of its discharge
curve; a supercapacitor's voltage drops **linearly** with stored charge
(`V = Q/C`), which means the downstream regulator must tolerate a much wider
input voltage range and the firmware must actively measure voltage (a cheap
proxy for stored energy, since `E = 0.5*C*V^2`) to make duty-cycle decisions
— there's no equivalent of a battery "fuel gauge IC" doing this
transparently in most low-cost harvesting designs; it's computed from a
voltage ADC reading in firmware.

## Verifying the power-budget and state-classification logic

Pure arithmetic and decision logic, compiled and run with `gcc`:

```c
#include <stdio.h>
#include <assert.h>
#include <math.h>

/* power_budget_sustainable / time_to_depletion_hours / classify_power_state
   / sleep_interval_ms_for_state as above */

int main(void) {
    assert(power_budget_sustainable(50.0, 30.0) == 1);   /* harvesting more than consuming */
    assert(power_budget_sustainable(20.0, 30.0) == 0);   /* deficit */

    double hours = time_to_depletion_hours(1000.0, 10.0);   /* 1000uJ stored, 10uW deficit */
    assert(fabs(hours - (1000.0/10.0/3600.0)) < 1e-9);

    assert(isinf(time_to_depletion_hours(1000.0, 0.0)));   /* no deficit -> never depletes */

    assert(classify_power_state(5.0, 100.0) == POWER_CRITICAL);
    assert(classify_power_state(50.0, 100.0) == POWER_NORMAL);
    assert(classify_power_state(90.0, 100.0) == POWER_ABUNDANT);

    assert(sleep_interval_ms_for_state(POWER_CRITICAL) > sleep_interval_ms_for_state(POWER_ABUNDANT));

    printf("power-budget and duty-cycle-state model OK\n");
    return 0;
}
```

## Traps in ultra-low-power and harvesting design

- **Sizing storage for average harvest instead of worst-case**: a design
  that balances on *average* daily solar input still fully depletes during
  a multi-day cloudy stretch unless storage capacity and the critical-state
  floor were sized against a realistic worst case, not the average.
- **Regulator quiescent current dominating the budget**: at the microwatt
  scale ultra-low-power harvesting designs target, the regulator's own
  quiescent current (often tens of nA to low µA for parts designed for this)
  can be a meaningful fraction of total budget — a regulator chosen for a
  battery design's convenience, not its quiescent draw, can single-handedly
  break a harvesting power budget.
- **No hysteresis between power states**: switching `classify_power_state`
  right at a threshold with noisy voltage readings causes rapid oscillation
  between duty-cycle settings — real designs add hysteresis (different
  thresholds for entering vs. leaving a state) to avoid this.
- **Ignoring supercapacitor leakage current**: unlike a chemical battery,
  supercapacitors self-discharge measurably over time — a design budget that
  omits this loses real energy the arithmetic didn't account for.

## Cheat sheet

| Concept | Detail |
|---|---|
| Power balance | Harvesting designs must satisfy avg(harvest) >= avg(consume), not a fixed total budget |
| Adaptive duty cycling | Sample stored energy, classify state, adjust sleep interval accordingly |
| `E = 0.5 * C * V^2` | Supercapacitor stored energy — voltage is the measurable proxy for energy level |
| Regulator quiescent current | Can dominate a microwatt-scale power budget — choose parts for this, not battery convenience |
| Hysteresis | Needed between power states to avoid oscillation near a threshold |
| Worst-case sizing | Storage capacity and critical-state behavior must survive worst case, not average, harvest conditions |
| Verification here | Power-balance and state-classification logic compiled/run with `gcc`; real harvester/regulator behavior needs lab measurement |

## Exercise

Add hysteresis to `classify_power_state`: rewrite it as a stateful function
`power_state_t classify_power_state_hyst(double stored_uj, double capacity_uj,
power_state_t previous_state)` that requires crossing a threshold by an
extra 5% margin to transition to a *worse* state than `previous_state`
(easy to enter a better state, sticky about leaving a good one) — this
prevents oscillation near a boundary. Write assertions showing a voltage
reading that flickers around the POWER_LOW/POWER_NORMAL boundary stays in
one state with hysteresis, and would have oscillated without it (compare
both versions in the same test). Compile and run with `gcc`.
