# FreeRTOS Tasks & Queues

Every Level 1 sketch had one `loop()`. It felt single-threaded, but it never
was: the Arduino-ESP32 core boots **FreeRTOS** first and runs your `setup()`
+ `loop()` inside a task called `loopTask`, pinned to core 1 at priority 1,
while the WiFi/Bluetooth stack quietly runs its own tasks on core 0. This
module makes that machinery explicit — you'll create tasks of your own, pin
them to a core, and pass data between them safely with queues and
semaphores. Everything here still runs from a normal Arduino sketch; ESP-IDF's
native project structure is next module.

## Why one `loop()` stops being enough

A single loop forces every job — sampling a sensor, debouncing a button,
running a display, watching the network — into one shared timeline, held
together by `millis()` bookkeeping (module 1-07). That works, but it has
limits: jobs with genuinely different timing needs (sample at exactly 1 kHz;
block for up to 100 ms waiting on a bus) can't cleanly share one scheduling
loop. **Tasks** give each job its own stack, its own priority, and its own
call to `vTaskDelay()` — the FreeRTOS scheduler does the interleaving for
you.

## Creating a task

```cpp
void sensorTask(void *pvParameters) {
  for (;;) {
    Serial.println("sampling...");
    vTaskDelay(pdMS_TO_TICKS(500));   // yields the CPU for ~500 ms
  }
  // a task must never fall off the end of its function —
  // if it can exit, call vTaskDelete(NULL) instead
}

void setup() {
  Serial.begin(115200);

  xTaskCreatePinnedToCore(
    sensorTask,      // function to run
    "sensorTask",    // name, for debugging (shows up in uxTaskGetStackHighWaterMark etc.)
    4096,            // stack size — IN BYTES on ESP-IDF, see the warning below
    NULL,            // parameter passed into the task function
    2,               // priority (higher number = higher priority)
    NULL,            // out: task handle, if you need to reference it later
    1                // core: 0, 1, or tskNO_AFFINITY to let the scheduler choose
  );
}

void loop() {
  vTaskDelay(pdMS_TO_TICKS(1000));   // loopTask itself is just another task
}
```

`xTaskCreatePinnedToCore()` returns `pdPASS` on success or an error code
(commonly `errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY`) if the heap can't supply
the stack — worth checking whenever code creates tasks after boot rather than
only at startup.

!!! warning "Stack size is in bytes, not words"
    Vanilla FreeRTOS measures `usStackDepth` in *words* (4 bytes each on a
    32-bit core). **ESP-IDF's port measures it in bytes.** Copy a stack size
    from a non-ESP FreeRTOS tutorial and your task gets a quarter of the
    stack you meant to give it — a classic silent-corruption trap. `4096`
    below means 4 KB, not 16 KB.

## Task priorities — and priority inversion

Priorities run from `0` (idle, lowest) up to `configMAX_PRIORITIES - 1`
(25 on the default ESP32 configuration). The scheduler always runs the
highest-priority *ready* task; equal-priority tasks time-slice. `loopTask`
sits at priority 1, which is why a task you create at priority 2 or higher
can starve `loop()` entirely if it never blocks.

**Priority inversion** is the classic trap: a low-priority task holds a
mutex a high-priority task needs. The high-priority task blocks waiting for
it — but a *medium*-priority task, unrelated to either, keeps preempting the
low-priority one, so the mutex never gets released and the high-priority
task waits far longer than it should. FreeRTOS's `xSemaphoreCreateMutex()`
solves this with **priority inheritance**: the low-priority holder is
temporarily boosted to the waiter's priority for as long as it holds the
lock. A plain binary semaphore (`xSemaphoreCreateBinary()`) does *not* do
this — use a real mutex whenever a shared resource, not just a signal, is
involved.

## Queues: passing data between tasks

Never write to a shared variable from two tasks and hope for the best — pass
**copies of data** through a queue instead. A queue is a fixed-length,
fixed-item-size FIFO the kernel manages for you:

```cpp
struct Reading {
  uint32_t timestampMs;
  float celsius;
};

QueueHandle_t readingQueue;

void producerTask(void *pv) {
  for (;;) {
    Reading r = { millis(), 21.5f + (esp_random() % 100) / 100.0f };
    if (xQueueSend(readingQueue, &r, pdMS_TO_TICKS(100)) != pdTRUE) {
      Serial.println("queue full — consumer too slow");
    }
    vTaskDelay(pdMS_TO_TICKS(200));
  }
}

void consumerTask(void *pv) {
  Reading r;
  for (;;) {
    if (xQueueReceive(readingQueue, &r, portMAX_DELAY) == pdTRUE) {
      Serial.printf("t=%lu  %.2f C\n", r.timestampMs, r.celsius);
    }
  }
}

void setup() {
  Serial.begin(115200);
  readingQueue = xQueueCreate(10, sizeof(Reading));   // 10 slots of struct Reading

  xTaskCreatePinnedToCore(producerTask, "producer", 2048, NULL, 2, NULL, 1);
  xTaskCreatePinnedToCore(consumerTask, "consumer", 2048, NULL, 2, NULL, 1);
}

void loop() { vTaskDelay(portMAX_DELAY); }
```

`xQueueSend()`'s third argument is how long to *wait* for space if the queue
is full (`0` = don't block, `portMAX_DELAY` = wait forever); `xQueueReceive()`
works the same way waiting for data. `portMAX_DELAY` in the consumer means
"block efficiently until something arrives" — no polling, no wasted CPU.
Structs passed by value are copied into the queue's internal buffer, so the
producer is free to reuse or destroy `r` immediately after `xQueueSend()`
returns.

## Protecting a shared peripheral with a mutex

Two tasks both calling `Wire.` (I2C, module 1-06) or `Serial.print()` at the
same time is a **race condition** — bytes from both calls can interleave on
the bus or the wire, producing garbage neither task sent. A mutex makes the
critical section atomic:

```cpp
SemaphoreHandle_t i2cMutex;

void readSensorSafely() {
  if (xSemaphoreTake(i2cMutex, pdMS_TO_TICKS(50)) == pdTRUE) {
    Wire.beginTransmission(0x3C);
    // ... talk to the bus ...
    Wire.endTransmission();
    xSemaphoreGive(i2cMutex);        // always release, even on early return
  } else {
    Serial.println("i2c busy, skipped this cycle");
  }
}

void setup() {
  i2cMutex = xSemaphoreCreateMutex();
  Wire.begin();
  // ... create tasks that call readSensorSafely() ...
}
```

The rule: **any resource touched from more than one task — a bus, a shared
struct, a file — needs a mutex**, or it needs to live behind a single task
that owns it exclusively (the producer/consumer pattern above follows the
same idea: only the consumer task ever touches the serial port for
readings).

## Detecting stack overflow before it corrupts memory

An overrun task stack silently tramples whatever memory sits next to it —
often another task's stack or heap data — producing symptoms nowhere near
the real cause. Check headroom instead of guessing:

```cpp
void sensorTask(void *pv) {
  for (;;) {
    // ... work ...
    UBaseType_t wordsFree = uxTaskGetStackHighWaterMark(NULL);  // NULL = this task
    if (wordsFree < 200) {
      Serial.printf("WARNING: %s low on stack: %u\n", pcTaskGetName(NULL), wordsFree);
    }
    vTaskDelay(pdMS_TO_TICKS(500));
  }
}
```

`uxTaskGetStackHighWaterMark()` reports the *closest the task ever came* to
running out, not the current free space — a small, unchanging number across
many cycles means the task found its steady-state stack use early; a number
that keeps shrinking means something (often unbounded recursion or a large
local buffer on one code path) is eating more stack over time. Module 2-08
covers the complementary safety net — watchdogs — for when a task stops
checking in at all.

## Cheat sheet

| Concept | Detail |
|---------|--------|
| `xTaskCreatePinnedToCore(fn, name, stack, arg, prio, &handle, core)` | Create + pin a task; returns `pdPASS` |
| Stack size units | **Bytes** on ESP-IDF (vanilla FreeRTOS uses words — a real gotcha) |
| Priority | `0` (idle) .. `configMAX_PRIORITIES-1` (25); higher preempts lower |
| `loopTask` | Arduino's `loop()`, running as a task at priority 1, core 1 |
| `vTaskDelay(pdMS_TO_TICKS(ms))` | Blocks *this* task only, yields CPU to others |
| Priority inversion | Low-prio task holds a lock a high-prio task needs; a mid-prio task starves both |
| Fix: real mutex | `xSemaphoreCreateMutex()` — has priority inheritance; binary semaphores don't |
| `xQueueCreate(len, itemSize)` | Fixed-size FIFO; copies data, not pointers, between tasks |
| `xQueueSend`/`xQueueReceive(q, &item, ticks)` | `0`=don't block, `portMAX_DELAY`=block forever |
| `xSemaphoreTake`/`Give(mutex, ticks)` | Guard any resource shared across tasks |
| `uxTaskGetStackHighWaterMark(NULL)` | Worst-case-ever free stack (words) for the calling task |

## Exercise

Build a two-task system: a `producerTask` (priority 2, core 1) that reads a
potentiometer every 200 ms and pushes `{timestamp, raw}` structs into a
10-slot queue, and a `consumerTask` (priority 1, core 0) that blocks on the
queue with `portMAX_DELAY` and prints each reading. Add a shared `Serial`
mutex even though only the consumer prints today — explain in a comment why
you'd still want it the moment a second task needs to log. Then intentionally
misconfigure the producer's stack to `768` bytes while giving it a local
`float buf[100]`, watch `uxTaskGetStackHighWaterMark()` head toward zero (or
watch it crash outright), and fix it by moving the buffer out of the task's
stack or increasing the stack size — note in a comment which fix you chose
and why.
