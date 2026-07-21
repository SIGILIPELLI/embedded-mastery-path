# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: get comfortable programming microcontrollers in C/C++ using the Arduino
framework on Arduino Uno and ESP32 boards — digital and analog I/O, serial
communication, sensors on I2C/SPI, non-blocking timing with interrupts, and a
first taste of WiFi — finishing with a complete simulated environment-monitor
device.

**You do not need to own any hardware for this level.** Every sketch runs in
the free [Wokwi online simulator](https://wokwi.com/) — a browser-based
simulator with virtual Arduino and ESP32 boards, LEDs, buttons,
potentiometers, DHT22 sensors, and OLED displays. Everything also works
unchanged on real boards.

## Modules

1. [Setup & Toolchain](01-setup-toolchain.md)
2. [C/C++ for Embedded](02-c-cpp-for-embedded.md)
3. [Digital I/O](03-digital-io.md)
4. [Analog I/O & PWM](04-analog-io-pwm.md)
5. [Serial/UART Communication](05-serial-uart.md)
6. [Sensors, I2C & SPI](06-sensors-i2c-spi.md)
7. [Timers & Interrupts](07-timers-interrupts.md)
8. [Memory & Power Basics](08-memory-power.md)
9. [ESP32 WiFi Intro](09-esp32-wifi.md)
10. [Capstone — Environment Monitor](10-capstone-project.md)

By the end of this level you'll be able to read buttons and sensors, drive
LEDs and displays, communicate over serial, schedule work without blocking
using `millis()` and interrupts, connect an ESP32 to WiFi, and combine all of
it into a working multi-sensor device — entirely in the browser if you want.
