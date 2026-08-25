# Interrupts & the NVIC

Level 1's `attachInterrupt()` and Level 2's ISR-safe FreeRTOS calls both sit
on top of the Cortex-M **NVIC** (Nested Vectored Interrupt Controller). This
module covers what the NVIC actually does — priority levels, nesting rules,
and the specific ways shared state between an ISR and normal code goes
wrong — using bare registers instead of a framework's `attachInterrupt`.

## Enabling and prioritizing an interrupt

```c
#define NVIC_ISER0  (*(volatile uint32_t *)0xE000E100)  /* Set-Enable */
#define NVIC_IPR0   (*(volatile uint32_t *)0xE000E400)  /* Priority, 4 regs/word on many parts */

#define EXTI0_IRQn  6   /* example: EXTI line 0 on many STM32F4 parts */

void enable_button_interrupt(void) {
    NVIC_ISER0 |= (1u << EXTI0_IRQn);          /* unmask the interrupt */

    /* priority register layout is chip/family specific — this pattern sets
       IRQ6's priority byte within IPR1 (IRQ 4-7), upper bits used depending
       on how many priority bits the implementation supports */
    volatile uint8_t *pri_byte = (volatile uint8_t *)(0xE000E400 + EXTI0_IRQn);
    *pri_byte = (2 << 4);   /* priority 2, assuming 4 implemented priority bits */
}

void EXTI0_IRQHandler(void) {
    EXTI_PR |= (1u << 0);        /* clear pending bit FIRST — see trap below */
    button_pressed_flag = 1;      /* keep the handler itself short */
}
```

## Nesting: why priority *numbers* are backwards

On Cortex-M, a **lower priority number means higher urgency** — priority 0
can preempt priority 5, not the other way around. A running ISR can be
interrupted by a higher-priority (lower-numbered) one; it cannot be
interrupted by an equal or lower-priority one, which instead waits and runs
after. This inversion trips up nearly everyone the first time: "priority 1"
sounds low-urgency in everyday language but is one of the most urgent levels
on the chip.

## The trap: clearing the pending flag after reading data, not before

```c
/* WRONG — a new edge between the read and the clear is lost */
void EXTI0_IRQHandler_buggy(void) {
    int level = read_pin_level();
    EXTI_PR |= (1u << 0);      /* if another edge arrived during read_pin_level(),
                                   its pending bit is now cleared without ever
                                   having been serviced */
}
```

The fix in the earlier example — clear-then-read for a level, or track a
counter that's incremented rather than a level that's overwritten — depends
on what the ISR actually needs, but the ordering question ("could a new
event arrive between my read and my clear?") has to be asked for every ISR
that clears its own pending bit.

## ISR/main-loop shared state needs `volatile`, and often more

```c
volatile uint32_t edge_count = 0;    /* volatile: main loop must re-read, not cache */

void EXTI0_IRQHandler(void) {
    EXTI_PR |= (1u << 0);
    edge_count++;                     /* NOT atomic on most cores for >8-bit values
                                          in general, but a single 32-bit RMW that
                                          only the ISR ever writes is safe here */
}

uint32_t read_edge_count_safely(void) {
    /* if main-loop code also WROTE edge_count, this would need to disable
       the interrupt around the read — reading a volatile is not the same
       as reading it atomically with respect to a concurrent writer */
    __disable_irq();
    uint32_t snapshot = edge_count;
    __enable_irq();
    return snapshot;
}
```

`volatile` guarantees the compiler won't cache or reorder the access — it
says nothing about atomicity. A 32-bit read on a 32-bit bus is naturally
atomic on Cortex-M *if* it's a single instruction and nothing splits it, but
a 64-bit counter or a multi-field struct shared with an ISR is not, and
needs an explicit critical section (`__disable_irq()`/`__enable_irq()`, kept
as short as possible) around any access from normal code.

## Modeling the priority-preemption rule in portable C

The nesting rule itself — lower number preempts higher, equal doesn't — is
pure logic, checkable without a real NVIC. Compiled and run with `gcc`:

```c
#include <stdio.h>
#include <assert.h>

/* returns 1 if a request at new_priority may preempt a handler
   currently running at current_priority */
int can_preempt(int current_priority, int new_priority) {
    return new_priority < current_priority;   /* lower number = higher urgency */
}

int main(void) {
    assert(can_preempt(5, 2) == 1);    /* priority 2 preempts priority 5 */
    assert(can_preempt(2, 5) == 0);    /* priority 5 does NOT preempt priority 2 */
    assert(can_preempt(3, 3) == 0);    /* equal priority never preempts */
    printf("NVIC preemption model OK\n");
    return 0;
}
```

## Traps in interrupt handling

- **Priority-number inversion**, as above — a very common source of "why
  isn't my important interrupt preempting the other one" bugs.
- **Long ISRs**: an ISR that does real work (parsing, printing, floating
  point) instead of setting a flag and returning blocks every equal/lower
  priority interrupt for its entire duration — keep ISRs to "record the
  event, wake a task" and do the work in task context.
- **Forgetting to clear the pending bit** (or clearing the wrong one) leaves
  the interrupt re-firing immediately after return, sometimes appearing as
  an infinite-loop hang with no useful stack trace.
- **Priority Group configuration** (`SCB_AIRCR`) splits the priority field
  into "preempt priority" and "subpriority" bits; getting this split wrong
  changes which interrupts can nest versus merely queue in order, in ways
  that don't show up until two specific interrupts race in the field.

## Cheat sheet

| Concept | Detail |
|---|---|
| NVIC | Cortex-M's interrupt controller — enable, priority, and pending state per IRQ |
| Priority numbers | **Lower number = higher urgency**; can preempt any strictly higher-numbered priority |
| Equal priority | Never preempts — queues and runs after the current handler returns |
| `EXTI_PR` / pending bit | Must be cleared in the handler, with care about ordering vs. reading data |
| `volatile` | Prevents compiler caching/reordering; does **not** by itself guarantee atomicity |
| `__disable_irq()`/`__enable_irq()` | Critical section for accesses that must be atomic w.r.t. an ISR |
| Verification here | Preemption-rule logic compiled/run with `gcc`; real NVIC register behavior reviewed against ARM docs only |

## Exercise

Write a portable C simulator that models a small NVIC: an array of pending
IRQs each with a priority, a "currently running priority" stack (since
handlers can nest), and a function `next_to_run()` that returns the
highest-urgency pending IRQ only if it's strictly more urgent than whatever
is on top of the running stack (or the first pending IRQ if nothing is
running). Simulate a sequence: IRQ at priority 5 starts, IRQ at priority 3
arrives and should preempt it, IRQ at priority 4 arrives during that and
should **not** preempt priority 3, then priority-3 finishes and priority-4
should run next. Assert each step and compile/run with `gcc`.
