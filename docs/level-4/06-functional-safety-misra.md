# Functional Safety & MISRA

Everything up to this module has optimized for correctness and performance.
**Functional safety** asks a different question: when something does go
wrong — a sensor fails, a bit flips, a programmer makes a mistake — does the
system fail in a way that doesn't hurt anyone? This module covers the
mindset (not a specific certification process) and **MISRA C**, the coding
standard most safety-relevant embedded C actually gets written against.
This is careful manual review of real, citable standards material — no
MISRA rule numbers or safety-standard clauses are invented here, and this
module is not a substitute for a real functional-safety qualification
process on an actual product.

## Fail-safe versus fail-operational: a real design decision

**Fail-safe** means a detected failure puts the system into a known-safe
state (a motor controller cuts power on a sensor fault, rather than guessing
a value and continuing). **Fail-operational** means the system must keep
working correctly even after a failure (an aircraft flight computer, where
"stop and go safe" isn't an option mid-flight) — which requires redundancy,
not just detection. Most embedded consumer/industrial products are
fail-safe designs; fail-operational is reserved for genuinely
safety-critical systems where stopping is itself unsafe. Deciding which
category a given design falls into happens at the requirements stage, not
as an afterthought during implementation.

## MISRA C: what it actually is

MISRA C (currently MISRA C:2012, with amendments) is a coding standard
published by the Motor Industry Software Reliability Association,
restricting the C language to a subset chosen to eliminate undefined,
unspecified, or easily-misused behavior. It doesn't make code "safe" by
itself — it removes a specific, well-documented category of bug (language
misuse) that safety analysis would otherwise have to account for separately
for every construct.

A few real, well-known MISRA C:2012 rules, cited accurately:

- **Rule 15.5** (advisory): "A function should have a single point of exit
  at the end" — motivates structuring functions to avoid multiple early
  `return` statements scattered through control flow, so cleanup and
  postconditions are easier to reason about and verify.
- **Rule 8.13** (advisory): a pointer parameter should be declared `const`
  if the function doesn't modify what it points to — makes the caller's
  contract explicit and catchable by the compiler.
- **Directive 4.1** (required): run-time failures must be minimized —
  the umbrella directive that most specific numeric/pointer-safety rules
  exist to support.
- The well-known **Rule 10.x series** governs "essential type" conversions
  — restricting implicit conversions between signed/unsigned/different
  widths that are a common source of the exact silent-corruption bugs this
  course has flagged repeatedly (module 3-06's bit-width traps, for
  example).

Citing rule numbers precisely matters here: safety audits check compliance
against specific numbered rules, and getting a rule number wrong in
documentation is itself a compliance-process failure, independent of the
code's actual correctness.

## A concrete before/after against real MISRA guidance

```c
/* before: multiple exit points (Rule 15.5), implicit signed/unsigned
   comparison (Rule 10.x family), no const on a read-only pointer (Rule 8.13) */
int validate_reading(float *buf, int len) {
    if (len == 0) return -1;
    for (unsigned int i = 0; i < len; i++) {   /* signed len vs unsigned i */
        if (buf[i] < 0.0f) return -2;
    }
    return 0;
}

/* after: single exit, const-correct, matched signedness */
int validate_reading_v2(const float *buf, int len) {
    int result = 0;
    if (len <= 0) {
        result = -1;
    } else {
        for (int i = 0; i < len; i++) {         /* both signed, no implicit conversion */
            if (buf[i] < 0.0f) { result = -2; break; }
        }
    }
    return result;
}
```

## Defensive programming beyond what MISRA mandates

Functional safety design generally goes further than a language subset —
watchdog timers (module 2-08's mechanism, now framed as a safety mitigation
rather than just a robustness feature), redundant sensor checks, and
range/plausibility validation on every external input are standard practice
in safety-conscious embedded design, independent of any specific
certification:

```c
/* plausibility check, not just a range check: does this reading make
   physical sense given the last reading and elapsed time? */
int reading_is_plausible(float new_val, float last_val, uint32_t dt_ms,
                          float max_rate_per_sec) {
    float dt_s = dt_ms / 1000.0f;
    float max_change = max_rate_per_sec * dt_s;
    float actual_change = fabsf(new_val - last_val);
    return actual_change <= max_change;
}
```

A temperature sensor reporting a physically impossible 40°C jump in 10 ms is
a strong signal of a fault (loose connection, ADC glitch, EMI) even though
the value itself might be within the sensor's normal operating range — a
plausibility check catches this class of failure that a simple min/max
range check cannot.

## Verifying the validation and plausibility logic

Both are pure logic and were compiled and run with `gcc`:

```c
#include <stdio.h>
#include <assert.h>
#include <math.h>

int validate_reading_v2(const float *buf, int len) {
    int result = 0;
    if (len <= 0) {
        result = -1;
    } else {
        for (int i = 0; i < len; i++) {
            if (buf[i] < 0.0f) { result = -2; break; }
        }
    }
    return result;
}

int reading_is_plausible(float new_val, float last_val, unsigned dt_ms, float max_rate_per_sec) {
    float dt_s = dt_ms / 1000.0f;
    float max_change = max_rate_per_sec * dt_s;
    float actual_change = fabsf(new_val - last_val);
    return actual_change <= max_change;
}

int main(void) {
    float good[] = {1.0f, 2.0f, 3.0f};
    float bad[]  = {1.0f, -2.0f, 3.0f};
    assert(validate_reading_v2(good, 3) == 0);
    assert(validate_reading_v2(bad, 3) == -2);
    assert(validate_reading_v2(good, 0) == -1);

    assert(reading_is_plausible(21.0f, 20.5f, 100, 5.0f) == 1);   /* 0.5C in 100ms, plausible */
    assert(reading_is_plausible(60.0f, 20.0f, 10, 5.0f) == 0);     /* 40C in 10ms, implausible */

    printf("MISRA-style validation + plausibility model OK\n");
    return 0;
}
```

## Traps in functional-safety-oriented design

- **Treating MISRA compliance as proof of safety**: MISRA reduces language-
  misuse bugs; it says nothing about whether the algorithm or requirements
  are correct — a MISRA-compliant implementation of the wrong control law
  is still wrong.
- **Watchdog that only proves the main loop is spinning**: a watchdog kicked
  unconditionally at the top of every loop iteration proves the loop is
  running, not that it's doing anything correct — a genuinely useful
  watchdog check should be tied to evidence of actual forward progress
  (a specific critical task having completed its work), not mere CPU
  activity.
- **Range checks without plausibility checks**: a value within its
  sensor's valid range can still be physically impossible given recent
  history — the two checks catch different failure classes and both matter.
- **No documented rationale for a MISRA deviation**: some advisory rules are
  legitimately violated in specific, justified cases — MISRA's own process
  requires documenting *why* a deviation is acceptable, not just suppressing
  the warning silently.

## Cheat sheet

| Concept | Detail |
|---|---|
| Fail-safe | On detected failure, go to a known-safe state — most common design goal |
| Fail-operational | Must keep working correctly through a failure — needs redundancy, reserved for critical systems |
| MISRA C:2012 | Restricts C to eliminate undefined/unspecified/misuse-prone constructs |
| Rule 15.5 | Single point of exit per function (advisory) |
| Rule 8.13 | `const`-correctness for read-only pointer parameters (advisory) |
| Rule 10.x | Essential-type conversion rules — governs signed/unsigned/width mismatches |
| Plausibility check | Validates a reading against recent history/rate limits, not just its static range |
| Verification here | Validation/plausibility logic compiled/run with `gcc`; MISRA rule text and safety-process claims reviewed against real, citable published material only |

## Exercise

Take the "before" `validate_reading` function above and rewrite it a second
way that also fixes a bug not yet addressed: it has no NULL-pointer check on
`buf`. Add that check as an explicit early branch consistent with the
single-exit style (set an error result and skip the loop, rather than an
early `return`), write assertions for a NULL buffer, a valid buffer, and an
invalid-value buffer, and compile/run with `gcc`. In a comment, state which
specific real MISRA directive or rule category (naming the general
family — pointer/null-related rules, not inventing a number if you don't
have the standard in front of you) this kind of check exists to support.
