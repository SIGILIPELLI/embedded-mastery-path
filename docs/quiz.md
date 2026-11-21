---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Embedded Systems Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does a microcontroller's toolchain do on your laptop?",
    "options": [
      "Edits it",
      "Runs the code",
      "Emulates it",
      "Cross-compiles code into machine code for the target MCU"
    ],
    "correctIndex": 3,
    "explanation": "Then flash via a bootloader or SWD."
  },
  {
    "question": "What does this print?\n```c\n#include <stdio.h>\n#include <stdint.h>\n#include <stdbool.h>\n#include <string.h>\nint main(void){ volatile uint32_t r=0; r|=(1u<<3); r|=(1u<<5); r&=~(1u<<3); r^=1u; printf(\"0x%02X\", r); }\n```\n",
    "options": [
      "0x29",
      "0x21",
      "0x28",
      "0x20"
    ],
    "correctIndex": 1,
    "explanation": "Set bits 3 and 5, clear 3, toggle bit 0. Verified by compiling and running."
  },
  {
    "question": "What does this print?\n```c\n#include <stdio.h>\n#include <stdint.h>\n#include <stdbool.h>\n#include <string.h>\nint main(void){ uint8_t a=200,b=100; uint8_t s=a+b; uint16_t w=(uint16_t)a+b; printf(\"%u %u\", s, w); }\n```\n",
    "options": [
      "44 300",
      "300 300",
      "44 44",
      "255 300"
    ],
    "correctIndex": 0,
    "explanation": "8-bit arithmetic wraps; widen before adding. Verified by compiling and running."
  },
  {
    "question": "A 12-bit ADC at 3.3 V reads 2048. What voltage is that?\n```python\nprint(round(2048/4095*3.3,3))\n```\n",
    "options": [
      "1.65",
      "0.825",
      "3.3",
      "2.048"
    ],
    "correctIndex": 0,
    "explanation": "V = raw/(2^12-1) x Vref. Verified by running it."
  },
  {
    "question": "At 115200 baud, 8N1, how long does one character take in microseconds (rounded)?\n```python\nprint(round(10/115200*1e6))\n```\n",
    "options": [
      "10",
      "8",
      "87",
      "870"
    ],
    "correctIndex": 2,
    "explanation": "10 bits per character. Verified by running it."
  },
  {
    "question": "Why is a floating digital input a problem?",
    "options": [
      "It's faster",
      "It's analog",
      "It picks up noise and reads random values; add a pull-up or pull-down",
      "It saves power"
    ],
    "correctIndex": 2,
    "explanation": "INPUT_PULLUP."
  },
  {
    "question": "A button produces several transitions per press. What fixes it?",
    "options": [
      "Ignore",
      "Debounce in hardware (RC) or software (stable-sample counting)",
      "Delay 5 s",
      "Use PWM"
    ],
    "correctIndex": 1,
    "explanation": "Contacts bounce for milliseconds."
  },
  {
    "question": "Why shouldn't an ISR call printf or delay?",
    "options": [
      "Speed",
      "Style",
      "ISRs must be short and non-blocking; set a flag or buffer data and return",
      "Syntax"
    ],
    "correctIndex": 2,
    "explanation": "Defer work to the main loop."
  },
  {
    "question": "Which qualifier must a flag shared with an ISR have?",
    "options": [
      "static only",
      "register",
      "const",
      "volatile (and ensure atomic access)"
    ],
    "correctIndex": 3,
    "explanation": "Prevent caching in a register."
  },
  {
    "question": "Two tasks on FreeRTOS pass sensor data. What's the right primitive?",
    "options": [
      "A global variable",
      "delay()",
      "printf",
      "A queue (thread-safe, can be used from ISRs with FromISR variants)"
    ],
    "correctIndex": 3,
    "explanation": "Avoid data races."
  },
  {
    "question": "A low-priority task holds a mutex a high-priority task needs while a medium task runs. What is this and the fix?",
    "options": [
      "Priority inversion; use mutexes with priority inheritance",
      "Race; volatile",
      "Deadlock; reboot",
      "Starvation; sleep"
    ],
    "correctIndex": 0,
    "explanation": "FreeRTOS mutexes provide inheritance."
  },
  {
    "question": "Which MQTT QoS gives at-least-once delivery?",
    "options": [
      "QoS 2",
      "QoS 0",
      "QoS 1",
      "None"
    ],
    "correctIndex": 2,
    "explanation": "QoS 2 is exactly once."
  },
  {
    "question": "A field device loses WiFi for hours. What should firmware do?",
    "options": [
      "Reboot loop",
      "Drop data",
      "Block",
      "Buffer readings locally and upload when reconnected, with reconnect/backoff"
    ],
    "correctIndex": 3,
    "explanation": "Store and forward."
  },
  {
    "question": "A firmware update is interrupted by a power cut. Which design survives?",
    "options": [
      "Dual application partitions with validation and rollback",
      "Erase first",
      "No check",
      "Single slot overwrite"
    ],
    "correctIndex": 0,
    "explanation": "OTA safety."
  },
  {
    "question": "Where should the watchdog be fed?",
    "options": [
      "In the main control path only when all critical tasks report healthy",
      "Never",
      "At boot",
      "In a timer ISR always"
    ],
    "correctIndex": 0,
    "explanation": "Otherwise it masks hangs."
  },
  {
    "question": "What does the startup code do before main()?",
    "options": [
      "Loads WiFi",
      "Starts RTOS",
      "Copies .data from flash to RAM, zeroes .bss and sets up the stack and vector table",
      "Flashes"
    ],
    "correctIndex": 2,
    "explanation": "Reset handler."
  },
  {
    "question": "Why use DMA for a 1 MS/s ADC stream?",
    "options": [
      "Style",
      "The CPU can't service every sample by interrupt; DMA moves data to memory",
      "Law",
      "Cheaper"
    ],
    "correctIndex": 1,
    "explanation": "Use half/full-transfer interrupts."
  },
  {
    "question": "Battery life is much worse than calculated. What do you measure first?",
    "options": [
      "Code size",
      "Current in sleep and during wake spikes with a current profiler, including regulators and pull-ups",
      "Pins",
      "Clock"
    ],
    "correctIndex": 1,
    "explanation": "Leakage and radio bursts dominate."
  },
  {
    "question": "A product needs functional safety evidence. What do you need besides passing tests?",
    "options": [
      "Hazard analysis, defined process, traceability, coding standard (MISRA) and independent verification",
      "A demo",
      "Nothing",
      "Marketing"
    ],
    "correctIndex": 0,
    "explanation": "Standards require process evidence."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A UART receive ring buffer

Complete rb_put (store at head, advance, drop and count when full) and rb_get (empty check, read from tail, advance). The buffer holds RB_SIZE-1 bytes so full and empty are distinguishable. A burst of 10 bytes arrives, then the main loop drains it.

**Starter code**

```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>

#define RB_SIZE 8   /* capacity is RB_SIZE - 1 so full and empty differ */

typedef struct {
    uint8_t buf[RB_SIZE];
    volatile uint8_t head;   /* written by the ISR (producer) */
    volatile uint8_t tail;   /* written by the main loop (consumer) */
    uint32_t dropped;
} ring_t;

static bool rb_put(ring_t *r, uint8_t b) {
    /* store b at head and advance; if the buffer is full, count a drop and return false */
    return false;
}

static bool rb_get(ring_t *r, uint8_t *out) {
    /* return false if empty; otherwise read from tail, advance and return true */
    return false;
}

int main(void) {
    ring_t r; memset(&r, 0, sizeof r);
    for (uint8_t i = 1; i <= 10; i++) rb_put(&r, i);       /* simulate a burst of 10 bytes */
    printf("dropped=%u\n", (unsigned)r.dropped);
    uint8_t b;
    while (rb_get(&r, &b)) printf("%u ", b);
    printf("\n");
    rb_put(&r, 42); rb_put(&r, 43);
    rb_get(&r, &b);
    printf("after wrap: first=%u\n", b);
    return 0;
}
```

**Expected output**

```text
dropped=3
1 2 3 4 5 6 7 
after wrap: first=42
```

<details><summary>Show solution and explanation</summary>

<pre><code>#include &lt;stdio.h&gt;
#include &lt;stdint.h&gt;
#include &lt;stdbool.h&gt;
#include &lt;string.h&gt;

#define RB_SIZE 8   /* capacity is RB_SIZE - 1 so full and empty differ */

typedef struct {
    uint8_t buf[RB_SIZE];
    volatile uint8_t head;   /* written by the ISR (producer) */
    volatile uint8_t tail;   /* written by the main loop (consumer) */
    uint32_t dropped;
} ring_t;

static bool rb_put(ring_t *r, uint8_t b) {
    uint8_t next = (uint8_t)((r-&gt;head + 1) % RB_SIZE);
    if (next == r-&gt;tail) { r-&gt;dropped++; return false; }   /* full: drop, never overwrite */
    r-&gt;buf[r-&gt;head] = b;
    r-&gt;head = next;
    return true;
}

static bool rb_get(ring_t *r, uint8_t *out) {
    if (r-&gt;head == r-&gt;tail) return false;                  /* empty */
    *out = r-&gt;buf[r-&gt;tail];
    r-&gt;tail = (uint8_t)((r-&gt;tail + 1) % RB_SIZE);
    return true;
}

int main(void) {
    ring_t r; memset(&amp;r, 0, sizeof r);
    for (uint8_t i = 1; i &lt;= 10; i++) rb_put(&amp;r, i);       /* simulate a burst of 10 bytes */
    printf(&quot;dropped=%u\n&quot;, (unsigned)r.dropped);
    uint8_t b;
    while (rb_get(&amp;r, &amp;b)) printf(&quot;%u &quot;, b);
    printf(&quot;\n&quot;);
    rb_put(&amp;r, 42); rb_put(&amp;r, 43);
    rb_get(&amp;r, &amp;b);
    printf(&quot;after wrap: first=%u\n&quot;, b);
    return 0;
}
</code></pre>

<p>The ISR only writes head and the main loop only writes tail, so a single-producer/single-consumer ring buffer needs no lock; dropping on overflow is safer than overwriting unread data. Verified by compiling and running the solution with gcc.</p>
</details>

### Debounce and register packing

Implement debounce(): the stable state flips only after STABLE_COUNT identical consecutive samples that differ from it, and `*changed` is true only on that sample. The program also packs a mode/enable/channel register; observe the output.

**Starter code**

```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>

/* Debounce a noisy button: the stable state changes only after STABLE_COUNT identical samples. */
#define STABLE_COUNT 3

typedef struct { bool stable; bool last; uint8_t count; } debounce_t;

static bool debounce(debounce_t *d, bool sample, bool *changed) {
    /* *changed = true only on the sample where the stable state flips. The stable state flips
       only after STABLE_COUNT identical consecutive samples that differ from the current stable state. */
    *changed = false;
    return d->stable;
}

/* Pack/unpack a register: bits [7:4] = mode, bit 3 = enable, bits [2:0] = channel. */
#define MODE_SHIFT 4
#define EN_BIT     3
static uint8_t reg_make(uint8_t mode, bool en, uint8_t ch) {
    return (uint8_t)(((mode & 0xF) << MODE_SHIFT) | ((en ? 1u : 0u) << EN_BIT) | (ch & 0x7));
}

int main(void) {
    int samples[] = {0,1,0,0,1,1,1,1,0,1,0,0,0,0};   /* noisy press then noisy release */
    debounce_t d = {false, false, 0};
    for (unsigned i = 0; i < sizeof samples / sizeof *samples; i++) {
        bool ch;
        debounce(&d, samples[i] != 0, &ch);
        if (ch) printf("sample %u: stable -> %s\n", i, d.stable ? "PRESSED" : "RELEASED");
    }
    uint8_t reg = reg_make(0xA, true, 5);
    printf("reg=0x%02X mode=%u en=%u ch=%u\n", reg, reg >> MODE_SHIFT, (reg >> EN_BIT) & 1u, reg & 7u);
    return 0;
}
```

**Expected output**

```text
sample 6: stable -> PRESSED
sample 12: stable -> RELEASED
reg=0xAD mode=10 en=1 ch=5
```

<details><summary>Show solution and explanation</summary>

<pre><code>#include &lt;stdio.h&gt;
#include &lt;stdint.h&gt;
#include &lt;stdbool.h&gt;
#include &lt;string.h&gt;

/* Debounce a noisy button: the stable state changes only after STABLE_COUNT identical samples. */
#define STABLE_COUNT 3

typedef struct { bool stable; bool last; uint8_t count; } debounce_t;

static bool debounce(debounce_t *d, bool sample, bool *changed) {
    *changed = false;
    if (sample == d-&gt;last) {
        if (d-&gt;count &lt; STABLE_COUNT) d-&gt;count++;
    } else {
        d-&gt;last = sample;
        d-&gt;count = 1;
    }
    if (d-&gt;count &gt;= STABLE_COUNT &amp;&amp; d-&gt;stable != d-&gt;last) {
        d-&gt;stable = d-&gt;last;
        *changed = true;
    }
    return d-&gt;stable;
}

/* Pack/unpack a register: bits [7:4] = mode, bit 3 = enable, bits [2:0] = channel. */
#define MODE_SHIFT 4
#define EN_BIT     3
static uint8_t reg_make(uint8_t mode, bool en, uint8_t ch) {
    return (uint8_t)(((mode &amp; 0xF) &lt;&lt; MODE_SHIFT) | ((en ? 1u : 0u) &lt;&lt; EN_BIT) | (ch &amp; 0x7));
}

int main(void) {
    int samples[] = {0,1,0,0,1,1,1,1,0,1,0,0,0,0};   /* noisy press then noisy release */
    debounce_t d = {false, false, 0};
    for (unsigned i = 0; i &lt; sizeof samples / sizeof *samples; i++) {
        bool ch;
        debounce(&amp;d, samples[i] != 0, &amp;ch);
        if (ch) printf(&quot;sample %u: stable -&gt; %s\n&quot;, i, d.stable ? &quot;PRESSED&quot; : &quot;RELEASED&quot;);
    }
    uint8_t reg = reg_make(0xA, true, 5);
    printf(&quot;reg=0x%02X mode=%u en=%u ch=%u\n&quot;, reg, reg &gt;&gt; MODE_SHIFT, (reg &gt;&gt; EN_BIT) &amp; 1u, reg &amp; 7u);
    return 0;
}
</code></pre>

<p>Counting consecutive identical samples filters contact bounce without delay(). Masks and shifts pack fields into a register exactly as peripheral registers are laid out. Verified by compiling and running the solution with gcc.</p>
</details>
