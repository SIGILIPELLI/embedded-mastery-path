---
description: "RTOS Internals — Level 2 used FreeRTOS as a tool: create a task, send it through a queue, protect a resource with a mutex. This module opens the box. On…"
---

# RTOS Internals

Level 2 used FreeRTOS as a tool: create a task, send it through a queue,
protect a resource with a mutex. This module opens the box. On top of the
bare-metal foundation from 3-01/3-02, you'll see what a "task" actually is
in memory, how the scheduler decides which one runs, and how a context
switch physically happens on a Cortex-M core — the same mechanism ESP-IDF's
FreeRTOS port was built on.

## A task is a saved CPU state plus a stack

Every task the kernel knows about is represented by a **Task Control Block**
(TCB) — conceptually just this, stripped to essentials:

```c
typedef struct tcb {
    uint32_t *stack_pointer;   /* saved SP — where this task's registers live */
    uint32_t  priority;
    struct tcb *next;          /* ready-list linkage */
    const char *name;
} TCB_t;
```

Creating a task doesn't start it running — it allocates a stack, and
pre-fills that stack with a **fake initial context**: register values
arranged exactly as if the task had been interrupted right after starting,
so the very first context switch into it "resumes" it at its entry function.

```c
uint32_t *task_stack_init(uint32_t *stack_top, void (*entry)(void)) {
    /* Cortex-M exception return pops these 8 words automatically */
    *(--stack_top) = 0x01000000;          /* xPSR — Thumb bit must be set */
    *(--stack_top) = (uint32_t)entry;     /* PC   — where the task starts */
    *(--stack_top) = 0xFFFFFFFD;          /* LR   — dummy, task never "returns" */
    *(--stack_top) = 0; /* R12 */
    *(--stack_top) = 0; /* R3  */
    *(--stack_top) = 0; /* R2  */
    *(--stack_top) = 0; /* R1  */
    *(--stack_top) = 0; /* R0  */
    /* R4-R11 pushed/popped manually by the context switch code below */
    for (int i = 0; i < 8; i++) *(--stack_top) = 0;
    return stack_top;
}
```

This is exactly what `xTaskCreate()` does internally — the "magic" of a task
starting to run the first time it's scheduled is just a stack frame shaped
to look like a return address.

## The context switch: PendSV

Cortex-M reserves an exception specifically for this: `PendSV`, given the
*lowest* priority of any exception on purpose, so it always runs last —
after any higher-priority interrupt has finished — never in the middle of
one. `SysTick` (a periodic timer) fires the scheduler tick; if it decides a
different task should run, it doesn't switch contexts itself — it sets
`PendSV` pending and returns. The CPU then runs `PendSV_Handler`:

```asm
PendSV_Handler:
    MRS   r0, psp            @ get current task's stack pointer
    STMDB r0!, {r4-r11}      @ save remaining registers (r0-r3,r12,lr,pc,xpsr
                              @ were already auto-saved by hardware on entry)
    LDR   r1, =current_tcb
    LDR   r2, [r1]
    STR   r0, [r2]           @ save SP into the outgoing task's TCB

    BL    scheduler_pick_next_task   @ updates current_tcb to the new task

    LDR   r2, [r1]
    LDR   r0, [r2]           @ load SP from the incoming task's TCB
    LDMIA r0!, {r4-r11}      @ restore its registers
    MSR   psp, r0
    BX    lr                 @ exception return — hardware restores the rest
```

Every context switch is: save eight registers to the old task's stack,
switch which stack pointer is active, restore eight registers from the new
task's stack. The other eight registers (R0-R3, R12, LR, PC, xPSR) are saved
and restored automatically by the Cortex-M exception hardware — this split
is why the handler only touches R4-R11 explicitly.

## Scheduling policy: why priority alone isn't the whole story

`scheduler_pick_next_task` in real FreeRTOS is a bitmap search: one bit per
priority level marks "at least one ready task at this priority," a
count-leading-zeros instruction finds the highest set bit in O(1), then it
round-robins within that priority's ready list. This is why raising a task's
priority above another task at the same priority as it *doesn't* mean the
first task always wins — same-priority tasks time-slice every tick; only a
strictly higher priority guarantees preemption.

## Modeling the ready-queue pick in portable C

The priority-bitmap idea can be verified without any ARM assembly — it's
just bit-scanning. Compiled and run with `gcc`:

```c
#include <stdio.h>
#include <assert.h>
#include <stdint.h>

/* one bit per priority level (0..31), set = at least one ready task there */
static int highest_ready_priority(uint32_t ready_bitmap) {
    if (ready_bitmap == 0) return -1;          /* nothing ready -> idle task */
    for (int p = 31; p >= 0; p--) {
        if (ready_bitmap & (1u << p)) return p;
    }
    return -1;
}

int main(void) {
    uint32_t bitmap = 0;
    bitmap |= (1u << 2);   /* a task ready at priority 2 */
    bitmap |= (1u << 5);   /* a task ready at priority 5 */
    assert(highest_ready_priority(bitmap) == 5);

    bitmap &= ~(1u << 5);  /* priority-5 task blocks on a queue */
    assert(highest_ready_priority(bitmap) == 2);

    bitmap = 0;
    assert(highest_ready_priority(bitmap) == -1);

    printf("scheduler pick model OK\n");
    return 0;
}
```

## Traps in RTOS internals

- **Calling a blocking API from `PendSV` or any ISR** — the scheduler code
  itself must never block; this is why FreeRTOS has separate `...FromISR()`
  variants for every queue/semaphore call.
- **Getting the PendSV priority wrong**: if `PendSV` isn't configured as the
  *lowest* priority exception, a context switch can preempt a higher-priority
  interrupt handler mid-flight, corrupting its assumptions about atomicity.
- **Stack frame layout mismatches**: if the fake initial context pushed by
  `task_stack_init` doesn't exactly match what the hardware expects to pop
  on exception return, the new task starts with garbage registers — a bug
  that is silicon/ABI-specific and nearly impossible to debug from symptoms
  alone.
- **Race on `current_tcb`**: reading/updating the "which task is running"
  pointer must itself be protected from interrupts, or a timer tick during
  the update can hand the CPU to a task mid-switch.

## How It Actually Works

**Why PendSV, specifically, and not SysTick itself, does the switch**: on
Cortex-M, exceptions can preempt each other by priority, and the hardware
exception-entry sequence (auto-saving R0-R3/R12/LR/PC/xPSR) can itself be
interrupted and resumed if a higher-priority exception arrives mid-entry —
this is called "tail-chaining" and it's a real hardware optimization. If the
context-switch logic ran directly inside `SysTick_Handler` (which typically
runs at a fairly high priority so the tick itself is reliable), a
higher-priority peripheral interrupt arriving during the switch could
observe an inconsistent stack — half old task, half new task — with no clean
way to unwind. By instead having `SysTick_Handler` merely set
`PendSV`'s pending bit (a single register write to `ICSR`) and return, the
actual register-juggling only ever executes when the CPU has no higher-priority
exception pending — because `PendSV` is configured at the numerically
lowest priority, meaning it will *always* be interrupted first and only
resumed once every genuinely urgent handler has finished, guaranteeing the
switch itself is never seen half-done by anything else in the system.

**Why only R4-R11 need explicit saving**: this is a direct consequence of
the Cortex-M **exception entry/exit hardware contract**. When any exception
fires — including the one that triggered `PendSV` — the core automatically
pushes R0-R3, R12, LR, PC, and xPSR onto whatever stack was active (this is
called "stacking," and it happens in hardware before the handler's first
instruction even executes), and automatically pops the same set back off on
return via the special `EXC_RETURN` value loaded into `LR`. The
`PendSV_Handler` code is running *after* that automatic stacking already
happened for the outgoing task and *before* the automatic unstacking happens
for the incoming task — so the eight already-stacked registers are already
exactly where they need to be on each task's own stack; the handler's only
remaining job is the eight registers the hardware contract doesn't cover.
This split isn't an implementation shortcut — it's what makes a context
switch on Cortex-M cheap enough to run on every scheduler tick without
meaningfully affecting throughput.

**Why the ready-bitmap scan is O(1) and not O(32)**: real FreeRTOS
implementations don't loop from bit 31 down to 0 as the teaching model above
does — they issue a single **CLZ (count leading zeros)** instruction, present
in the Cortex-M instruction set specifically for exactly this kind of
priority-encoding use case, which returns the bit position of the
highest set bit in one CPU cycle regardless of which bit it is. This is what
makes "which task should run next" a constant-time operation independent of
`configMAX_PRIORITIES` — a scheduler that had to loop through priority
levels one at a time would make context-switch cost scale with how many
priority levels your project happens to define, which is exactly the kind of
non-deterministic overhead a real-time kernel is built to avoid.

## Cheat sheet

| Concept | Detail |
|---|---|
| TCB | Task Control Block — saved SP + priority + list linkage, not the stack itself |
| Initial fake context | Stack pre-filled so the first switch "resumes" the task at its entry point |
| `PendSV` | Lowest-priority exception, dedicated to context switches, never preempts real IRQs |
| Auto-saved by hardware | R0-R3, R12, LR, PC, xPSR — pushed/popped by exception entry/return |
| Manually saved in handler | R4-R11 — the context-switch code's own job |
| Ready bitmap | One bit per priority; highest set bit = next task to run (O(1) via CLZ) |
| Same-priority tasks | Time-slice via `SysTick`; only strictly higher priority guarantees preemption |
| Verification here | Bitmap scheduling logic compiled/run with `gcc`; PendSV assembly reviewed, not executed |

## Exercise

Extend the `highest_ready_priority` model into a small round-robin simulator:
keep an array of ready task IDs per priority level and a `next_task()`
function that returns the next task to run, advancing round-robin within the
highest occupied priority and leaving lower-priority tasks untouched. Add a
test where a priority-5 task blocks (removed from the bitmap) mid-run and
confirm control correctly falls to priority 2 without skipping any
priority-2 task in the rotation. Compile and run it with `gcc`, and in a
comment explain what would go wrong (starvation) if the model exposed no
priority levels at all and every task just took turns.
