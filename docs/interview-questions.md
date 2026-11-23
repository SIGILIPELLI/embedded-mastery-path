---
title: "Embedded Systems Interview Questions and Answers (179 Questions)"
description: "179 Embedded Systems interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Embedded Systems Mastery Path."
---

# Embedded Systems Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Embedded Systems Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; Toolchain](level-1/01-setup-toolchain.md)

**Q1.** What is a microcontroller?

**Answer:** A small self-contained chip with CPU, RAM, flash and peripherals

Programs are cross-compiled for it.

**Q2.** What does the toolchain do on your laptop?

**Answer:** Compile and link code into machine code for the target MCU

Cross compilation.

**Q3.** What is a sketch's `setup()` for?

**Answer:** One-time initialization before loop() runs repeatedly

Arduino model.

**Q4.** Why upload via a bootloader or programmer?

**Answer:** The compiled image must be written into the MCU's flash

USB serial or SWD.

### [02 · C/C++ for Embedded](level-1/02-c-cpp-for-embedded.md)

**Q5.** What does this print?

```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>
int main(void){ uint8_t x=250; x+=10; int8_t y=127; y++; printf("%u %d", x, y); }
```

**Answer:** 4 -128

Fixed-width types wrap at their size (signed overflow is technically undefined; shown for typical two's complement). Verified by compiling and running.

**Q6.** What does this print?

```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>
int main(void){ volatile uint32_t reg=0; reg|=(1u<<3); reg|=(1u<<5); reg&=~(1u<<3); printf("0x%02X", reg); }
```

**Answer:** 0x20

Set bit 3 and 5, then clear bit 3. Verified by compiling and running.

**Q7.** What does `volatile` tell the compiler?

**Answer:** The value can change outside the program's flow, so don't cache or optimize away accesses

Hardware registers and ISR-shared variables.

**Q8.** Why prefer fixed-width integers like uint8_t on MCUs?

**Answer:** Sizes of int vary by platform; fixed width makes behavior predictable

Registers have exact widths.

### [03 · Digital I/O](level-1/03-digital-io.md)

**Q9.** What does pinMode(pin, OUTPUT) do?

**Answer:** Configures the pin to drive high or low

INPUT reads.

**Q10.** What's a floating input?

**Answer:** An unconnected input that picks up noise; use a pull-up or pull-down resistor

INPUT_PULLUP.

**Q11.** Why debounce a mechanical button?

**Answer:** Contacts bounce for milliseconds producing multiple transitions

Software or hardware.

**Q12.** What current limit applies to GPIO pins?

**Answer:** A few to tens of mA; use a transistor/driver for larger loads

Check the datasheet.

### [04 · Analog I/O &amp; PWM](level-1/04-analog-io-pwm.md)

**Q13.** A 10-bit ADC reads 512 with Vref = 5 V. What voltage is that?

```python
print(round(512/1023*5,3))
```

**Answer:** 2.502

V = raw/(2^n - 1) x Vref. Verified by running it.

**Q14.** What's the ADC resolution step of a 12-bit ADC at 3.3 V?

```python
print(round(3.3/4096*1000,3))
```

**Answer:** 0.806

LSB = Vref / 2^n (in mV). Verified by running it.

**Q15.** What does PWM do?

**Answer:** Switches a pin on and off quickly; the duty cycle sets the average power

Dimming and motors.

**Q16.** How does a duty cycle of 25% at 5 V behave on an LED?

**Answer:** About 1.25 V average, so it looks dimmer

Average of the waveform.

### [05 · Serial/UART Communication](level-1/05-serial-uart.md)

**Q17.** What does UART need to communicate?

**Answer:** Matching baud rate and framing on TX/RX plus a common ground

Asynchronous.

**Q18.** At 9600 baud with 8N1 framing, how many characters per second at most?

```python
print(9600//10)
```

**Answer:** 960

10 bits per character (start + 8 data + stop). Verified by running it.

**Q19.** What happens if baud rates mismatch?

**Answer:** Garbled characters

Check both ends.

**Q20.** Why print debug output over serial sparingly?

**Answer:** It blocks/takes time and can disturb real-time behavior

Log levels.

### [06 · Sensors, I2C &amp; SPI](level-1/06-sensors-i2c-spi.md)

**Q21.** What's the difference between I2C and SPI?

**Answer:** I2C uses two shared wires with addresses; SPI uses a clock, data lines and a chip-select per device and is faster

Choose by need.

**Q22.** Why do I2C lines need pull-up resistors?

**Answer:** The bus is open-drain: devices only pull low

Typically 4.7k.

**Q23.** What identifies a device on I2C?

**Answer:** A 7-bit address

i2cdetect finds them.

**Q24.** What does SPI mode (CPOL/CPHA) define?

**Answer:** Clock polarity and phase for sampling data

Must match the device.

### [07 · Timers &amp; Interrupts](level-1/07-timers-interrupts.md)

**Q25.** Why avoid delay() in real firmware?

**Answer:** It blocks everything else; use timers or millis() state machines

Non-blocking design.

**Q26.** What does an interrupt do?

**Answer:** Pauses normal code to run a handler when an event occurs

Keep it short.

**Q27.** What should an ISR do?

**Answer:** Minimal work: set a flag or buffer data and return

Defer to main loop.

**Q28.** What does this print?

```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>
volatile int flag=0;
void isr(void){ flag=1; }
int main(void){ isr(); if(flag) printf("handled"); else printf("missed"); }
```

**Answer:** handled

Share state with an ISR through a volatile flag. Verified by compiling and running.

### [08 · Memory &amp; Power Basics](level-1/08-memory-power.md)

**Q29.** Which memories does an MCU typically have?

**Answer:** Flash (code), SRAM (data) and EEPROM/NVS for settings

Fixed at design time.

**Q30.** Where do string literals go on AVR unless told otherwise?

**Answer:** Copied to SRAM at startup; use F()/PROGMEM to keep them in flash

Save RAM.

**Q31.** What dominates battery life?

**Answer:** How long and how often the MCU and radios are awake

Sleep modes.

**Q32.** What does deep sleep do?

**Answer:** Powers down most of the chip, waking on a timer or pin

Microamps.

### [09 · ESP32 WiFi Intro](level-1/09-esp32-wifi.md)

**Q33.** What does an ESP32 add over a basic MCU?

**Answer:** Built-in WiFi and Bluetooth radios

IoT-ready.

**Q34.** What is an HTTP GET used for in a device?

**Answer:** Fetching data from a server

Check status codes.

**Q35.** Why handle WiFi disconnects in firmware?

**Answer:** Links drop; reconnect and retry rather than hang

Robustness.

**Q36.** What should you never hardcode in source?

**Answer:** WiFi passwords and API keys

Use config/NVS.

### [10 · Capstone — Environment Monitor](level-1/10-capstone-project.md)

**Q37.** What does the environment monitor combine?

**Answer:** Sensor reading, display, alarm and WiFi in one device

Level 1 capstone.

**Q38.** Why use millis() scheduling in the project?

**Answer:** Multiple tasks need non-blocking timing

Cooperative scheduling.

**Q39.** How do you handle a failed sensor read?

**Answer:** Detect and report/skip it rather than using garbage values

Check for NaN.

**Q40.** What would you add for production?

**Answer:** Watchdog, logging and OTA

Level 2 topics.

## Level 2 · Intermediate

### [FreeRTOS Tasks &amp; Queues](level-2/01-freertos-tasks-queues.md)

**Q41.** What does FreeRTOS provide?

**Answer:** Tasks, queues, semaphores and timers on an MCU

Preemptive scheduling.

**Q42.** What's a queue for?

**Answer:** Passing data safely between tasks or from an ISR

Thread-safe.

**Q43.** What's priority inversion?

**Answer:** A high-priority task blocked by a low-priority one holding a resource

Use priority inheritance mutexes.

**Q44.** Why not call blocking APIs from an ISR?

**Answer:** They can't wait in interrupt context; use FromISR variants

xQueueSendFromISR.

### [ESP-IDF Development](level-2/02-esp-idf.md)

**Q45.** What is ESP-IDF?

**Answer:** Espressif's official development framework underlying Arduino-ESP32

More control.

**Q46.** What does menuconfig do?

**Answer:** Configures chip, RTOS and component options

sdkconfig.

**Q47.** What's a component in ESP-IDF?

**Answer:** A reusable module with its own CMake and sources

Modular.

**Q48.** Why use ESP-IDF over Arduino?

**Answer:** Finer control of resources, partitions and features

Production firmware.

### [Bluetooth Low Energy (BLE)](level-2/03-ble.md)

**Q49.** Why use BLE over WiFi?

**Answer:** Much lower power and no router needed for phone connections

Short data bursts.

**Q50.** What is a GATT characteristic?

**Answer:** A data value a BLE peripheral exposes for read/write/notify

Services group them.

**Q51.** What is advertising in BLE?

**Answer:** Broadcasting presence so central devices can discover and connect

Interval affects power.

**Q52.** What does notify do?

**Answer:** Pushes updates to a subscribed client without polling

Efficient.

### [MQTT &amp; IoT Messaging](level-2/04-mqtt-iot-messaging.md)

**Q53.** What does MQTT use?

**Answer:** Publish/subscribe messaging through a broker

Lightweight for IoT.

**Q54.** What are MQTT QoS levels 0, 1, 2?

**Answer:** At most once, at least once, exactly once

Trade-off with overhead.

**Q55.** What does a retained message do?

**Answer:** The broker stores the last value for new subscribers

Status topics.

**Q56.** What's a Last Will message?

**Answer:** Published by the broker if a client disconnects unexpectedly

Detect offline devices.

### [OTA Firmware Updates](level-2/05-ota-updates.md)

**Q57.** Why use OTA updates?

**Answer:** Devices in the field can't be reached with a cable

Remote maintenance.

**Q58.** What's needed for a safe OTA?

**Answer:** Two app partitions, validation and rollback

Fail-safe.

**Q59.** Why sign firmware images?

**Answer:** Prevent installing malicious or corrupted firmware

Verify before boot.

**Q60.** What does the 'mark app valid' step do?

**Answer:** Confirms the new firmware booted OK so it isn't rolled back

Anti-brick.

### [Advanced Peripherals (I2S, RMT, CAN)](level-2/06-advanced-peripherals.md)

**Q61.** What does I2S carry?

**Answer:** Continuous digital audio streams

Mics and DACs.

**Q62.** What's RMT used for?

**Answer:** Generating/receiving precisely timed pulse trains (IR, LED strips)

Hardware timing.

**Q63.** When use DMA with peripherals?

**Answer:** To move data without CPU involvement at high rates

Frees the CPU.

**Q64.** Why not bit-bang a protocol when hardware support exists?

**Answer:** Timing is jittery and wastes CPU

Use peripherals.

### [Filesystems &amp; Storage (LittleFS, SD)](level-2/07-filesystems-storage.md)

**Q65.** What's NVS for?

**Answer:** Storing small key/value settings that survive reboot

Configuration.

**Q66.** What's LittleFS suited to?

**Answer:** Wear-leveled, power-loss-resilient files on flash

Logs.

**Q67.** Why batch flash writes?

**Answer:** Flash has limited erase cycles and slow writes

Reduce wear.

**Q68.** What happens on a power cut during write without protection?

**Answer:** Corruption; use journaling/atomic updates

Design for it.

### [Watchdogs &amp; Robust Firmware](level-2/08-watchdogs-robust-firmware.md)

**Q69.** What does a watchdog do?

**Answer:** Resets the device if the firmware stops 'feeding' it

Recovers from hangs.

**Q70.** Where should you feed the watchdog?

**Answer:** Only when the system is truly healthy, not blindly in a timer ISR

Otherwise it hides hangs.

**Q71.** What is a brownout?

**Answer:** Supply voltage dropping below a safe level, causing erratic behavior

Brownout detectors reset safely.

**Q72.** What should firmware do after a watchdog reset?

**Answer:** Record the cause and enter a safe state or retry

Diagnose.

### [Debugging &amp; Unit Testing](level-2/09-debugging-unit-testing.md)

**Q73.** What's a core dump/panic handler useful for?

**Answer:** Capturing crash state for post-mortem debugging

Save to flash.

**Q74.** How do you unit test embedded logic?

**Answer:** Compile hardware-independent code on the host with mocks

Fast tests.

**Q75.** What does JTAG/SWD provide?

**Answer:** Hardware debugging: breakpoints, memory and register access

OpenOCD, gdb.

**Q76.** Why add assertions?

**Answer:** Catch invalid states early near the cause

Fail fast in development.

### [Project — WiFi Data Logger](level-2/10-project-wifi-data-logger.md)

**Q77.** What does the WiFi data logger combine?

**Answer:** Scheduled sampling, storage, WiFi/MQTT upload, watchdog and OTA

A year-long device.

**Q78.** How do you handle offline periods?

**Answer:** Buffer data locally and upload when the link returns

Store and forward.

**Q79.** What validates robustness?

**Answer:** Fault injection: power cuts, WiFi loss and bad sensors

Test failures.

**Q80.** How do you keep power low?

**Answer:** Sleep between samples and batch transmissions

Duty cycle.

## Level 3 · Advanced

### [Bare-Metal ARM Cortex-M (STM32)](level-3/01-bare-metal-arm-stm32.md)

**Q81.** What does bare-metal programming mean?

**Answer:** Running directly on the hardware without an OS or framework

You set clocks and registers.

**Q82.** What's the first code that runs after reset?

**Answer:** The reset handler from the vector table, which sets up memory and calls main

Startup code.

**Q83.** Why configure the clock tree manually?

**Answer:** Peripherals and CPU need correct frequencies from PLLs/dividers

Datasheet.

**Q84.** What does HAL provide?

**Answer:** A vendor abstraction layer over registers

LL is lower level.

### [Registers, Linker Scripts &amp; Startup Code](level-3/02-registers-linker-startup.md)

**Q85.** What does the linker script define?

**Answer:** Memory regions and where sections (.text, .data, .bss) are placed

FLASH/RAM layout.

**Q86.** What is .bss?

**Answer:** Zero-initialized static data (cleared at startup)

Not stored in flash.

**Q87.** What does startup code do with .data?

**Answer:** Copies initial values from flash to RAM

Before main.

**Q88.** What's the vector table?

**Answer:** An array of addresses for the stack pointer and exception/interrupt handlers

At the start of flash.

### [RTOS Internals](level-3/03-rtos-internals.md)

**Q89.** What does a scheduler's context switch save?

**Answer:** CPU registers and stack pointer of the running task

PendSV on Cortex-M.

**Q90.** What's a tick?

**Answer:** A periodic timer interrupt driving scheduling and delays

SysTick.

**Q91.** What's a semaphore for?

**Answer:** Signaling and resource counting between tasks/ISRs

Binary or counting.

**Q92.** What's stack overflow's effect in an RTOS?

**Answer:** Corrupts memory and crashes; size stacks and enable checks

uxTaskGetStackHighWaterMark.

### [DMA &amp; High-Speed Peripherals](level-3/04-dma-high-speed-peripherals.md)

**Q93.** What does DMA do?

**Answer:** Moves data between memory and peripherals without CPU copies

Frees CPU.

**Q94.** When is DMA essential?

**Answer:** High-speed streams like ADC sampling or audio

Throughput.

**Q95.** What is a circular DMA buffer?

**Answer:** A buffer refilled continuously with half/complete interrupts

Double buffering.

**Q96.** What hazard do DMA buffers add?

**Answer:** Cache coherency and ownership: CPU and DMA must not touch the same data

Use barriers.

### [Low-Power Design Deep Dive](level-3/05-low-power-design.md)

**Q97.** What's the biggest lever for low power?

**Answer:** Spend most time in deep sleep and wake rarely

Duty cycle.

**Q98.** What wakes an MCU from sleep?

**Answer:** Timers, GPIO interrupts and peripherals

Configure sources.

**Q99.** Why disable unused peripherals and clocks?

**Answer:** They draw current even when idle

Clock gating.

**Q100.** How do you measure low power accurately?

**Answer:** A current profiler/analyzer capturing sleep and active phases

Wake spikes matter.

### [Interrupts &amp; the NVIC](level-3/06-interrupts-nvic.md)

**Q101.** What is NVIC?

**Answer:** Cortex-M's nested vectored interrupt controller handling priorities

Preemption.

**Q102.** What does interrupt priority mean?

**Answer:** Higher-priority interrupts can preempt lower-priority handlers

Lower number = higher priority on Cortex-M.

**Q103.** What should an ISR avoid?

**Answer:** Long operations and blocking calls

Keep latency low.

**Q104.** Why clear the pending flag in an ISR?

**Answer:** Otherwise it re-enters continuously

Acknowledge the source.

### [PCB Design Basics (KiCad)](level-3/07-pcb-design-kicad.md)

**Q105.** What's a typical first step in KiCad?

**Answer:** Draw the schematic, then assign footprints and lay out the PCB

ERC then DRC.

**Q106.** Why add decoupling capacitors near ICs?

**Answer:** They supply transient current and filter noise

100 nF close to pins.

**Q107.** What does DRC check?

**Answer:** Design rules like clearances and trace widths

Before fabrication.

**Q108.** Why consider return paths and ground planes?

**Answer:** Signal integrity and EMI depend on them

Solid ground.

### [Protocol Deep Dive (UART/I2C/SPI/CAN)](level-3/08-protocol-deep-dive.md)

**Q109.** What does a logic analyzer do?

**Answer:** Captures digital signals over time to decode protocols

Debug I2C/SPI.

**Q110.** What's clock stretching in I2C?

**Answer:** A slave holding SCL low to delay the master

Some masters mishandle it.

**Q111.** What does CAN provide?

**Answer:** A robust multi-master differential bus with arbitration (automotive/industrial)

Error detection.

**Q112.** Why add checksums/CRC to custom protocols?

**Answer:** Detect corrupted frames

Reliability.

### [Bootloaders &amp; Firmware Updates](level-3/09-bootloaders.md)

**Q113.** What does a bootloader do?

**Answer:** Runs first, can update application firmware, then jumps to it

Update path.

**Q114.** What must a bootloader verify before jumping?

**Answer:** A valid application image (CRC/signature) and vector table

Avoid bricking.

**Q115.** Why keep the bootloader minimal and protected?

**Answer:** A bug or erase in it can brick the device

Write-protect flash.

**Q116.** How does the app vector table relocate?

**Answer:** VTOR points to the app's table

Cortex-M.

### [Project — Custom Sensor Board Firmware](level-3/10-project-sensor-board.md)

**Q117.** What does the sensor board project combine?

**Answer:** Custom PCB, bare-metal firmware, DMA/low power and a bootloader

Level 3 capstone.

**Q118.** How do you validate the board?

**Answer:** Bring-up checklist: power rails, clocks, communications and sensor accuracy

Stage the bring-up.

**Q119.** Why add test points?

**Answer:** Probing signals for debug and manufacturing test

DFT.

**Q120.** What documentation helps?

**Answer:** Schematics, BOM, pinouts and firmware notes

Handover.

## Level 4 · Master

### [Production Firmware Architecture](level-4/01-production-firmware-architecture.md)

**Q121.** What does a production firmware architecture include?

**Answer:** Layered modules, HAL abstraction, state machines, logging and update support

Maintainable.

**Q122.** Why layer HAL, drivers and application?

**Answer:** Testability and portability across hardware

Mock hardware.

**Q123.** Why use state machines?

**Answer:** Explicit, testable behavior for event-driven devices

Avoid spaghetti.

**Q124.** What should logging support?

**Answer:** Levels and persistent crash logs without blocking

Field diagnosis.

### [Secure Boot &amp; Encrypted OTA at Scale](level-4/02-secure-boot-ota-scale.md)

**Q125.** What does secure boot at scale require?

**Answer:** Key management, signed images and anti-rollback

Chain of trust.

**Q126.** What does anti-rollback prevent?

**Answer:** Installing older vulnerable firmware

Version counters.

**Q127.** Why use staged OTA rollouts?

**Answer:** Limit damage from a bad release

Canary.

**Q128.** Where are signing keys kept?

**Answer:** In an HSM or secured offline infrastructure

Never in the repo.

### [TinyML &amp; Edge AI](level-4/03-tinyml-edge-ai.md)

**Q129.** What's TinyML?

**Answer:** Running machine learning inference on microcontrollers

Quantized models.

**Q130.** What are TinyML constraints?

**Answer:** Memory, compute and power budgets

Small models.

**Q131.** Why quantize models?

**Answer:** Smaller size and faster integer inference

int8.

**Q132.** What data matters most?

**Answer:** Representative data from the target sensors

Field accuracy.

### [RF &amp; Wireless Protocol Design](level-4/04-rf-protocol-design.md)

**Q133.** What does link budget account for?

**Answer:** Transmit power, antenna gains, path loss and receiver sensitivity

Range planning.

**Q134.** Why choose LoRa for long range?

**Answer:** Low data rate with high sensitivity and low power

Trade-offs.

**Q135.** What's a packet format design concern?

**Answer:** Framing, CRC, versioning and security

Interoperability.

**Q136.** Why consider regulatory limits (duty cycle)?

**Answer:** Radio rules limit transmit time and power

Compliance.

### [HIL Testing &amp; CI for Firmware](level-4/05-hil-testing-ci.md)

**Q137.** What does HIL testing do?

**Answer:** Runs real firmware on hardware with simulated inputs automatically

Hardware-in-the-loop.

**Q138.** Why integrate HIL into CI?

**Answer:** Catch hardware-dependent regressions on every change

Test farms.

**Q139.** What does a HIL rig need?

**Answer:** Device under test, power/reset control, signal injection and result capture

Automation.

**Q140.** What stays in host unit tests?

**Answer:** Pure logic without hardware dependencies

Fast feedback.

### [Functional Safety &amp; MISRA](level-4/06-functional-safety-misra.md)

**Q141.** What is MISRA C?

**Answer:** A coding standard for safe C in critical systems

Reduces undefined behavior.

**Q142.** What does functional safety (IEC 61508/ISO 26262) require?

**Answer:** Hazard analysis, defined processes, verification and traceability

Safety integrity levels.

**Q143.** What's a safe state?

**Answer:** A defined condition the system enters on fault to avoid harm

Design for it.

**Q144.** Why static analysis?

**Answer:** Find rule violations and defects without running the code

Part of the process.

### [Ultra-Low-Power &amp; Energy Harvesting](level-4/07-ultra-low-power.md)

**Q145.** What's the target in ultra-low-power design?

**Answer:** Microamp average current via deep sleep and minimal wake time

Energy budget.

**Q146.** What often dominates energy?

**Answer:** Radio transmissions and wake overhead

Batch and shorten.

**Q147.** How to estimate battery life?

**Answer:** Average current from duty cycle, battery capacity and derating

Measure.

**Q148.** What's leakage current?

**Answer:** Current drawn by components even when idle, such as regulators and pull-ups

Choose parts carefully.

### [Manufacturing, DFM &amp; Factory Test](level-4/08-manufacturing-dfm-test.md)

**Q149.** What is DFM?

**Answer:** Design for manufacturing: making the board easy and cheap to build

Footprints, panelization.

**Q150.** Why provide test fixtures and a factory test mode?

**Answer:** Verify every unit and provision keys/IDs on the line

Production test.

**Q151.** What belongs in the BOM?

**Answer:** Parts, values, footprints and approved alternatives

Supply chain.

**Q152.** Why provision unique identities at manufacturing?

**Answer:** Per-device keys and IDs enable secure fleet management

Secure provisioning.

### [Fleet Management &amp; Device Clouds](level-4/09-fleet-management.md)

**Q153.** What does fleet management provide?

**Answer:** Monitoring, configuration and updates of many devices

Scale.

**Q154.** What telemetry matters?

**Answer:** Version, health, battery, errors and connectivity

Observability.

**Q155.** How do you handle devices offline for months?

**Answer:** Resumable updates and compatibility across versions

Version skew.

**Q156.** What's a device twin/shadow?

**Answer:** A cloud representation of desired and reported device state

Sync later.

### [Capstone — Production IoT Product](level-4/10-capstone-production-iot.md)

**Q157.** What does the production IoT capstone integrate?

**Answer:** Secure boot, OTA, telemetry, low power, testing and manufacturing

Full product.

**Q158.** How do you prove readiness?

**Answer:** HIL/CI, field trials, fault injection and a staged rollout plan

Evidence.

**Q159.** What documentation matters?

**Answer:** Architecture, runbooks, release and support procedures

Lifecycle.

**Q160.** What guards long-term support?

**Answer:** Version policy, security updates and compatibility testing

Years in the field.

## Mixed questions across all levels

**Q161.** What does a microcontroller's toolchain do on your laptop?

**Answer:** Cross-compiles code into machine code for the target MCU

Then flash via a bootloader or SWD.

**Q162.** What does this print?

```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>
int main(void){ volatile uint32_t r=0; r|=(1u<<3); r|=(1u<<5); r&=~(1u<<3); r^=1u; printf("0x%02X", r); }
```

**Answer:** 0x21

Set bits 3 and 5, clear 3, toggle bit 0. Verified by compiling and running.

**Q163.** What does this print?

```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>
int main(void){ uint8_t a=200,b=100; uint8_t s=a+b; uint16_t w=(uint16_t)a+b; printf("%u %u", s, w); }
```

**Answer:** 44 300

8-bit arithmetic wraps; widen before adding. Verified by compiling and running.

**Q164.** A 12-bit ADC at 3.3 V reads 2048. What voltage is that?

```python
print(round(2048/4095*3.3,3))
```

**Answer:** 1.65

V = raw/(2^12-1) x Vref. Verified by running it.

**Q165.** At 115200 baud, 8N1, how long does one character take in microseconds (rounded)?

```python
print(round(10/115200*1e6))
```

**Answer:** 87

10 bits per character. Verified by running it.

**Q166.** Why is a floating digital input a problem?

**Answer:** It picks up noise and reads random values; add a pull-up or pull-down

INPUT_PULLUP.

**Q167.** A button produces several transitions per press. What fixes it?

**Answer:** Debounce in hardware (RC) or software (stable-sample counting)

Contacts bounce for milliseconds.

**Q168.** Why shouldn't an ISR call printf or delay?

**Answer:** ISRs must be short and non-blocking; set a flag or buffer data and return

Defer work to the main loop.

**Q169.** Which qualifier must a flag shared with an ISR have?

**Answer:** volatile (and ensure atomic access)

Prevent caching in a register.

**Q170.** Two tasks on FreeRTOS pass sensor data. What's the right primitive?

**Answer:** A queue (thread-safe, can be used from ISRs with FromISR variants)

Avoid data races.

**Q171.** A low-priority task holds a mutex a high-priority task needs while a medium task runs. What is this and the fix?

**Answer:** Priority inversion; use mutexes with priority inheritance

FreeRTOS mutexes provide inheritance.

**Q172.** Which MQTT QoS gives at-least-once delivery?

**Answer:** QoS 1

QoS 2 is exactly once.

**Q173.** A field device loses WiFi for hours. What should firmware do?

**Answer:** Buffer readings locally and upload when reconnected, with reconnect/backoff

Store and forward.

**Q174.** A firmware update is interrupted by a power cut. Which design survives?

**Answer:** Dual application partitions with validation and rollback

OTA safety.

**Q175.** Where should the watchdog be fed?

**Answer:** In the main control path only when all critical tasks report healthy

Otherwise it masks hangs.

**Q176.** What does the startup code do before main()?

**Answer:** Copies .data from flash to RAM, zeroes .bss and sets up the stack and vector table

Reset handler.

**Q177.** Why use DMA for a 1 MS/s ADC stream?

**Answer:** The CPU can't service every sample by interrupt; DMA moves data to memory

Use half/full-transfer interrupts.

**Q178.** Battery life is much worse than calculated. What do you measure first?

**Answer:** Current in sleep and during wake spikes with a current profiler, including regulators and pull-ups

Leakage and radio bursts dominate.

**Q179.** A product needs functional safety evidence. What do you need besides passing tests?

**Answer:** Hazard analysis, defined process, traceability, coding standard (MISRA) and independent verification

Standards require process evidence.

## Practical coding questions

### A UART receive ring buffer

Complete rb_put (store at head, advance, drop and count when full) and rb_get (empty check, read from tail, advance). The buffer holds RB_SIZE-1 bytes so full and empty are distinguishable. A burst of 10 bytes arrives, then the main loop drains it.

Starter code:

```
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

Expected output:

```text
dropped=3
1 2 3 4 5 6 7 
after wrap: first=42
```

??? success "Show a solution"

    ```
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
        uint8_t next = (uint8_t)((r->head + 1) % RB_SIZE);
        if (next == r->tail) { r->dropped++; return false; }   /* full: drop, never overwrite */
        r->buf[r->head] = b;
        r->head = next;
        return true;
    }

    static bool rb_get(ring_t *r, uint8_t *out) {
        if (r->head == r->tail) return false;                  /* empty */
        *out = r->buf[r->tail];
        r->tail = (uint8_t)((r->tail + 1) % RB_SIZE);
        return true;
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

    The ISR only writes head and the main loop only writes tail, so a single-producer/single-consumer ring buffer needs no lock; dropping on overflow is safer than overwriting unread data. Verified by compiling and running the solution with gcc.

### Debounce and register packing

Implement debounce(): the stable state flips only after STABLE_COUNT identical consecutive samples that differ from it, and `*changed` is true only on that sample. The program also packs a mode/enable/channel register; observe the output.

Starter code:

```
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

Expected output:

```text
sample 6: stable -> PRESSED
sample 12: stable -> RELEASED
reg=0xAD mode=10 en=1 ch=5
```

??? success "Show a solution"

    ```
    #include <stdio.h>
    #include <stdint.h>
    #include <stdbool.h>
    #include <string.h>

    /* Debounce a noisy button: the stable state changes only after STABLE_COUNT identical samples. */
    #define STABLE_COUNT 3

    typedef struct { bool stable; bool last; uint8_t count; } debounce_t;

    static bool debounce(debounce_t *d, bool sample, bool *changed) {
        *changed = false;
        if (sample == d->last) {
            if (d->count < STABLE_COUNT) d->count++;
        } else {
            d->last = sample;
            d->count = 1;
        }
        if (d->count >= STABLE_COUNT && d->stable != d->last) {
            d->stable = d->last;
            *changed = true;
        }
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

    Counting consecutive identical samples filters contact bounce without delay(). Masks and shifts pack fields into a register exactly as peripheral registers are laid out. Verified by compiling and running the solution with gcc.

---

More free practice: the [Embedded Systems Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
