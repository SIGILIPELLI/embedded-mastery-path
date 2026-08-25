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
