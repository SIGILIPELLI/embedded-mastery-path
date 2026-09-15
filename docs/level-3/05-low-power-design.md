---
description: "Low-Power Design Deep Dive — Module 1-08 covered ESP32 sleep modes at the API level: call esp_deep_sleep_start(), wake on a timer or GPIO. This module…"
---

# Low-Power Design Deep Dive

Module 1-08 covered ESP32 sleep modes at the API level: call
`esp_deep_sleep_start()`, wake on a timer or GPIO. This module goes one
level down — the actual mechanisms (clock gating, power domains, wake-up
logic) that make sleep modes work on any Cortex-M chip, and the arithmetic
for deciding which mode a real battery-powered design actually needs.

## The power/wake-latency tradeoff

Every low-power mode trades retained state and wake speed for current draw:

| Mode | What's off | Wake latency | Typical current (STM32-class MCU) |
|---|---|---|---|
| Run | Nothing | — | ~5-20 mA (clock-speed dependent) |
| Sleep | CPU clock only | ~microseconds | ~1-3 mA |
| Stop | Most clocks; RAM/registers retained | ~microseconds-ms | ~1-10 µA |
| Standby | Nearly everything; RAM lost | ~ms (behaves like reset) | ~1-2 µA |
| Shutdown | Everything except a few wake pins | ~ms, full reboot | tens of nA |

Deeper sleep is not free even ignoring current: **Standby loses RAM**,
meaning the application must persist any state it needs across the sleep
(to backup registers or external flash) and re-initialize everything on
wake, including re-running clock and peripheral setup that Run/Sleep modes
never need to repeat.

## Configuring Stop mode and the wake-up interrupt

```c
#define PWR_CR   (*(volatile uint32_t *)0x40007000)
#define SCB_SCR  (*(volatile uint32_t *)0xE000ED10)

void enter_stop_mode(void) {
    PWR_CR |= (1u << 0);          /* LPDS: regulator in low-power mode during Stop */
    SCB_SCR |= (1u << 2);         /* SLEEPDEEP bit — WFI now enters Stop, not just Sleep */

    __asm volatile ("wfi");       /* Wait For Interrupt: halts CPU clock until a wake event */

    /* execution resumes HERE after wake — clocks must be reconfigured:
       Stop mode drops the system clock back to the internal RC oscillator */
    system_clock_reconfigure();
}
```

`WFI` is the actual mechanism — a single ARM instruction that halts
instruction execution until any enabled interrupt fires. `SLEEPDEEP` in the
System Control Block decides *how deep* `WFI` goes: clear, it's ordinary
Sleep (CPU clock gated, everything else running); set, the power controller
also gates peripheral clocks and possibly drops the core voltage, per the
`PWR_CR` configuration. The exact combination of bits needed for a given
depth of sleep is chip-specific and always comes from the reference manual
— this is one of the areas where copying a bit pattern from a different
STM32 family's example code silently does the wrong thing.

## The trap: any pending interrupt cancels sleep instantly

`WFI` returns the instant *any* enabled interrupt becomes pending — not just
the one meant to wake the device. A `SysTick` interrupt left enabled at
1 kHz makes `WFI` wake up 1000 times a second and immediately go back to
sleep, burning far more power than a design that disables `SysTick` (or
switches it to a slow "tick-less idle" scheme) before entering a deep sleep
mode. This is the single most common reason a "low power" design measures
nowhere near its datasheet current: something is still ticking.

```c
void enter_stop_mode_safely(void) {
    uint32_t saved_systick_ctrl = SYSTICK_CTRL;
    SYSTICK_CTRL &= ~(1u << 0);         /* disable SysTick before sleeping */

    PWR_CR |= (1u << 0);
    SCB_SCR |= (1u << 2);
    __asm volatile ("wfi");

    SYSTICK_CTRL = saved_systick_ctrl;  /* restore on wake */
    system_clock_reconfigure();
}
```

## Modeling the current-budget math in portable C

The one piece of this that's genuinely just arithmetic — and worth
verifying — is battery-life estimation from a duty-cycled current profile.
Compiled and run with `gcc`:

```c
#include <stdio.h>
#include <assert.h>

/* returns estimated battery life in hours for a periodic wake/sleep cycle */
double estimate_battery_hours(double battery_mah,
                               double active_ma, double active_ms,
                               double sleep_ua, double sleep_ms) {
    double cycle_ms = active_ms + sleep_ms;
    double avg_ma = (active_ma * active_ms + (sleep_ua / 1000.0) * sleep_ms) / cycle_ms;
    return battery_mah / avg_ma;
}

int main(void) {
    /* wake every 10s, active 50ms at 15mA, sleep the rest at 3uA, 220mAh coin cell */
    double hours = estimate_battery_hours(220.0, 15.0, 50.0, 3.0, 9950.0);
    printf("estimated life: %.0f hours (%.1f days)\n", hours, hours / 24.0);
    assert(hours > 1000);   /* duty cycle this light should comfortably exceed a month */

    /* sanity check: always-on at the active current should be far worse */
    double always_on_hours = 220.0 / 15.0;
    assert(always_on_hours < hours);
    printf("always-on comparison: %.1f hours\n", always_on_hours);
    return 0;
}
```

This is the calculation every "will this run a year on a coin cell" question
reduces to — get the duty cycle and the two current numbers from a
datasheet/measurement, and the arithmetic itself is simple and verifiable
independent of any specific chip.

## Traps in low-power design

- **Leaving a peripheral clock enabled that isn't used** costs current in
  every mode down to the one that gates it — audit `RCC_*ENR` registers
  before shipping, not just GPIO configuration.
- **Floating input pins** draw unexpected current from internal pull
  contention; every unused pin should be explicitly configured (input with
  pull, or analog to disconnect the digital input buffer entirely) before
  sleep.
- **Debug interfaces left enabled** (SWD) can prevent some low-power modes
  from reaching their rated current, or prevent them from being entered at
  all on some silicon — a classic "why is my production current 10x the
  datasheet" bug traced to a debug connection or a debug-mode fuse.
- **Wake source not actually enabled in the EXTI/interrupt controller** —
  configuring a pin as a wake source in the power controller is necessary
  but not sufficient; the corresponding interrupt line must also be unmasked
  in the NVIC (module 3-06), or the event never reaches the CPU to end WFI.

## How It Actually Works

**What `WFI` actually does at the transistor level**: `WFI` is a real ARM
instruction, but its effect is to signal the core's clock-and-power
controller logic that the pipeline has nothing left to execute — that
controller then physically gates (stops toggling) the clock signal feeding
the CPU's sequential logic. Since dynamic power in CMOS digital circuits is
dominated by switching activity (every clock edge charges and discharges
gate capacitances), a stopped clock means the CPU core's dynamic power drops
to near zero almost immediately — this is the actual physical mechanism
behind "halts the CPU," not a software loop or a low-power firmware trick.
`SLEEPDEEP` extends this same gating decision to additional clock domains
and, in Stop/Standby, to the voltage regulator itself: the power controller
can request the internal LDO or SMPS regulator supplying the core drop to a
lower-current mode or shut off entirely, which is why deeper modes need
longer wake latency — bringing a regulator or PLL-based clock back to a
stable, usable state takes real settling time measured in microseconds to
milliseconds, unlike simply re-enabling a gated clock signal.

**Why losing RAM in Standby is a hardware retention decision, not a software
one**: SRAM cells hold their state as long as they receive a minimum supply
voltage — even a small continuous leakage current — through cross-coupled
inverters that need to stay powered to keep reinforcing their stored bit.
Sleep and Stop modes keep the RAM's supply rail active (at reduced voltage in
Stop, to cut leakage while still meeting the SRAM's retention voltage
minimum) specifically so those cells never lose their charge. Standby and
Shutdown cut power to the RAM's supply domain entirely to reach their much
lower current figures, and an unpowered SRAM cell has no mechanism to
remember anything — which is precisely why waking from Standby is
architecturally indistinguishable from a power-on reset: there is no saved
state left to resume, only a small always-on "backup domain" (a separate,
tiny, continuously-powered SRAM/register block with its own supply pin) that
survives specifically because it's wired to a rail the power controller
never gates.

**Why any pending interrupt — not just the intended one — ends `WFI`**: `WFI`
is architecturally defined to resume execution when the processor's
interrupt-pending logic (the same NVIC pending-bit mechanism from module
3-06) has any bit set that isn't masked by `PRIMASK`/`BASEPRI` — it's a
single hardware condition check ("is anything pending that I'm allowed to
service"), not a per-source semantic distinction between "the wake source"
and "everything else." The CPU's wake logic has no concept of which
interrupt you *intended* as the wake trigger; from the hardware's point of
view, `SysTick` firing at 1 kHz and a deliberate GPIO wake pin look
identical — both flip a pending bit the WFI condition is watching — which is
exactly why every unrelated periodic interrupt has to be explicitly disabled
before entering deep sleep, not merely deprioritized.

## Cheat sheet

| Concept | Detail |
|---|---|
| `WFI` | Halts CPU until any enabled interrupt is pending — the core sleep primitive |
| `SLEEPDEEP` (SCB_SCR) | Selects how deep `WFI` goes: Sleep vs Stop/Standby |
| Sleep | CPU clock off, peripherals/RAM retained, µs wake |
| Stop | Most clocks off, RAM retained, µs-ms wake, reconfigure clocks on resume |
| Standby | RAM lost, behaves like reset on wake, must persist state externally |
| Any pending IRQ wakes | Disable/mask anything not meant to wake the device (e.g. `SysTick`) before deep sleep |
| Wake source setup | Must be enabled in **both** the power controller and the NVIC/EXTI |
| Verification here | Battery-life arithmetic compiled/run with `gcc`; actual sleep-mode current is hardware-measured, not testable here |

## Exercise

Extend `estimate_battery_hours` to accept a list of N duty-cycle phases
(each with its own current and duration) instead of just active/sleep, and
compute a weighted average current across a full cycle that includes a
third phase — e.g. a brief "radio transmit" burst at 80 mA for 20 ms once
per cycle. Add an assertion that adding the transmit phase measurably
shortens the estimated life compared to the two-phase version, compile and
run with `gcc`, and in a comment state which of the three phases dominates
the average and why (hint: compare each phase's `current x duration`
contribution, not just its current).
