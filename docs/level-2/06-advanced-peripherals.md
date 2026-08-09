# Advanced Peripherals (I2S, RMT, CAN)

Module 1-06 covered I2C and SPI — the two buses that cover most sensors.
This module covers three peripherals that exist because those two can't do
the job: **I2S** for continuous audio streams, **RMT** for pulse trains
timed to a fraction of a microsecond, and **TWAI** (Espressif's name for
CAN) for multi-node industrial and automotive buses. What unites them is
that all three are *hardware state machines with DMA* — the CPU sets them up
and then stays out of the way, which is exactly what bit-banging from a
FreeRTOS task can never achieve.

## I2S: continuous audio

I2C moves a few bytes when you ask. Audio needs 32,000–96,000 samples per
second, forever, with no gaps — a missed sample is an audible click. **I2S**
is a synchronous serial format built for exactly that: a bit clock (BCLK), a
word-select line (WS/LRCLK) that toggles once per sample to mark left vs.
right channel, and a data line (DIN or DOUT).

ESP-IDF v5 replaced the old `driver/i2s.h` with per-mode drivers. For
standard Philips-format devices — MEMS microphones like the INMP441, DACs
like the MAX98357A — that's `driver/i2s_std.h`:

```c
#include "driver/i2s_std.h"

static i2s_chan_handle_t rx_chan;

void i2s_mic_init(void)
{
    i2s_chan_config_t chan_cfg =
        I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);
    /* NULL for tx: this channel only receives */
    ESP_ERROR_CHECK(i2s_new_channel(&chan_cfg, NULL, &rx_chan));

    i2s_std_config_t std_cfg = {
        .clk_cfg  = I2S_STD_CLK_DEFAULT_CONFIG(16000),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
                        I2S_DATA_BIT_WIDTH_32BIT, I2S_SLOT_MODE_MONO),
        .gpio_cfg = {
            .mclk = I2S_GPIO_UNUSED,
            .bclk = GPIO_NUM_26,
            .ws   = GPIO_NUM_25,
            .dout = I2S_GPIO_UNUSED,
            .din  = GPIO_NUM_33,
            .invert_flags = { .mclk_inv = false, .bclk_inv = false, .ws_inv = false },
        },
    };
    ESP_ERROR_CHECK(i2s_channel_init_std_mode(rx_chan, &std_cfg));
    ESP_ERROR_CHECK(i2s_channel_enable(rx_chan));
}

void mic_task(void *pv)
{
    static int32_t buf[512];
    size_t bytes_read = 0;

    for (;;) {
        /* blocks until the DMA buffer is full — no polling, no busy-wait */
        if (i2s_channel_read(rx_chan, buf, sizeof(buf),
                             &bytes_read, portMAX_DELAY) == ESP_OK) {
            int samples = bytes_read / sizeof(int32_t);
            int64_t sum = 0;
            for (int i = 0; i < samples; i++) {
                sum += llabs(buf[i] >> 14);   /* 24-bit sample in a 32-bit slot */
            }
            ESP_LOGI(TAG, "level %lld", sum / samples);
        }
    }
}
```

The `>> 14` is not arbitrary: a 24-bit microphone left-aligns its sample
inside a 32-bit slot, so the raw `int32_t` is enormous until you shift the
padding out. Reading a mic as `int16_t` and wondering why everything is
silence or noise is the classic first I2S bug.

## RMT: precise pulse trains

The RMT (Remote Control) peripheral was built for infrared remotes, but it
generalizes to anything that is "a list of (level, duration) pairs emitted
with hardware timing". Its unit is the `rmt_symbol_word_t`: two levels with
two durations, measured in ticks of a resolution you choose.

WS2812 addressable LEDs are the standard example — they encode bits as pulse
widths around 1.25 µs, far too tight to bit-bang reliably while FreeRTOS is
scheduling other tasks:

```c
#include "driver/rmt_tx.h"

#define RMT_RES_HZ 10000000   /* 10 MHz → 1 tick = 0.1 µs */

static rmt_channel_handle_t led_chan;
static rmt_encoder_handle_t led_encoder;

void ws2812_init(void)
{
    rmt_tx_channel_config_t tx_cfg = {
        .gpio_num          = GPIO_NUM_18,
        .clk_src           = RMT_CLK_SRC_DEFAULT,
        .resolution_hz     = RMT_RES_HZ,
        .mem_block_symbols = 64,
        .trans_queue_depth = 4,
    };
    ESP_ERROR_CHECK(rmt_new_tx_channel(&tx_cfg, &led_chan));

    rmt_bytes_encoder_config_t enc_cfg = {
        /* WS2812: '0' = 0.3 µs high + 0.9 µs low; '1' = 0.9 µs high + 0.3 µs low */
        .bit0 = { .level0 = 1, .duration0 = 3, .level1 = 0, .duration1 = 9 },
        .bit1 = { .level0 = 1, .duration0 = 9, .level1 = 0, .duration1 = 3 },
        .flags.msb_first = 1,
    };
    ESP_ERROR_CHECK(rmt_new_bytes_encoder(&enc_cfg, &led_encoder));
    ESP_ERROR_CHECK(rmt_enable(led_chan));
}

void ws2812_write(const uint8_t *grb, size_t len)   /* note: G, R, B order */
{
    rmt_transmit_config_t tx_conf = { .loop_count = 0 };
    ESP_ERROR_CHECK(rmt_transmit(led_chan, led_encoder, grb, len, &tx_conf));
    ESP_ERROR_CHECK(rmt_tx_wait_all_done(led_chan, portMAX_DELAY));
    /* WS2812 latches after >50 µs of idle line */
    esp_rom_delay_us(60);
}
```

`rmt_transmit()` queues the transfer and returns immediately — the
peripheral clocks the bits out on its own. That is the entire point: the
timing is unaffected by a higher-priority task preempting you mid-frame.

## TWAI: the CAN bus

CAN is what cars, tractors, and a lot of industrial equipment run on.
Espressif calls its controller **TWAI** (Two-Wire Automotive Interface) for
trademark reasons; it is CAN 2.0. Its defining properties are worth
understanding before the API:

- **Multi-master, no addresses.** Frames carry a *message ID* describing the
  content ("engine RPM"), not a destination. Everyone hears everything and
  filters locally.
- **Lower ID wins arbitration.** If two nodes transmit at once, the one with
  the numerically lower ID takes the bus without a collision or a retry —
  priority is baked into the ID space.
- **Differential and long-haul.** Up to 1 Mbit/s over tens of metres, with
  hardware error counters and automatic retransmission.

The ESP32 has the controller but not the transceiver — you need an external
SN65HVD230 or MCP2551 between the GPIOs and the actual CAN_H/CAN_L pair,
plus 120 Ω termination at both ends of the bus.

```c
#include "driver/twai.h"

void can_init(void)
{
    twai_general_config_t g = TWAI_GENERAL_CONFIG_DEFAULT(
        GPIO_NUM_21, GPIO_NUM_22, TWAI_MODE_NORMAL);   /* tx, rx, mode */
    twai_timing_config_t t = TWAI_TIMING_CONFIG_500KBITS();
    twai_filter_config_t f = TWAI_FILTER_CONFIG_ACCEPT_ALL();

    ESP_ERROR_CHECK(twai_driver_install(&g, &t, &f));
    ESP_ERROR_CHECK(twai_start());
}

void can_tx_task(void *pv)
{
    twai_message_t msg = {
        .identifier       = 0x123,   /* 11-bit standard ID */
        .extd             = 0,       /* 1 for 29-bit extended IDs */
        .data_length_code = 4,
        .data             = { 0xDE, 0xAD, 0xBE, 0xEF },
    };
    for (;;) {
        if (twai_transmit(&msg, pdMS_TO_TICKS(1000)) != ESP_OK) {
            ESP_LOGW(TAG, "tx queue full or bus not ready");
        }
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

void can_rx_task(void *pv)
{
    twai_message_t rx;
    for (;;) {
        if (twai_receive(&rx, portMAX_DELAY) == ESP_OK) {
            ESP_LOGI(TAG, "id=0x%03lX dlc=%d", rx.identifier, rx.data_length_code);
        }
    }
}
```

!!! warning "A CAN node alone on the bus goes bus-off, not silent"
    CAN requires at least one *other* node to acknowledge each frame. A
    single ESP32 transmitting with nothing else attached gets no ACK,
    increments its transmit error counter on every retry, and at 256 enters
    **bus-off** — the controller takes itself offline and stops transmitting
    entirely. This looks exactly like "my code stopped working" and is
    almost always missing termination, a missing second node, or swapped
    CAN_H/CAN_L. Use `TWAI_MODE_NO_ACK` for solo bench testing, and poll
    `twai_get_status_info()` for the real state.

## Traps worth knowing

- **I2S DMA buffers set your latency floor.** `dma_desc_num × dma_frame_num`
  determines how much audio is buffered; too small and you get underruns
  when a higher-priority task runs long, too large and you add tens of
  milliseconds of delay. Read with `portMAX_DELAY` and let the driver pace
  you rather than adding `vTaskDelay()` into an audio loop.
- **RMT memory blocks are shared and finite.** Each channel claims
  `mem_block_symbols` from a common pool; allocating several wide channels
  fails at `rmt_new_tx_channel()` with `ESP_ERR_NOT_FOUND`. Check the return
  code instead of assuming the channel exists.
- **WS2812 wants GRB, not RGB**, and 5 V logic — a 3.3 V ESP32 data line
  often works but is out of spec; a level shifter or a sacrificial first LED
  is the reliable fix.
- **Bit-rate mismatches on CAN are invisible.** A node at 250 kbit/s on a
  500 kbit/s bus doesn't warn you; it just accumulates bus errors. Every
  node must agree exactly.
- **All three peripherals are limited in count** (ESP32: 2 I2S, 1 TWAI,
  8 RMT channels split between TX and RX). Check the chip's datasheet before
  designing a board that needs three of anything.

## Cheat sheet

| Concept | Detail |
|---------|--------|
| I2S | Continuous synchronous audio: BCLK + WS (left/right) + data, DMA-backed |
| `i2s_new_channel(&cfg, &tx, &rx)` | Pass `NULL` for the direction you don't need |
| `I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(bits, mode)` | Standard format for most mics/DACs |
| `i2s_channel_read(h, buf, len, &got, timeout)` | Blocks until a DMA buffer is full |
| 24-bit mic in 32-bit slot | Shift the padding out (`>> 14`) or the numbers look absurd |
| RMT | Hardware (level, duration) pulse generator — IR, WS2812, any tight timing |
| `resolution_hz` | Sets the tick; 10 MHz → 1 tick = 0.1 µs |
| `rmt_symbol_word_t` | `{level0, duration0, level1, duration1}` — one bit's waveform |
| `rmt_transmit()` | Queues and returns; hardware clocks it out, immune to preemption |
| TWAI | ESP32's CAN 2.0 controller — needs an external transceiver + 120 Ω ends |
| Message ID | Describes content, not destination; **lower ID wins arbitration** |
| `TWAI_GENERAL_CONFIG_DEFAULT(tx, rx, mode)` | Plus timing + filter configs into `twai_driver_install()` |
| Bus-off | 256 TX errors (often: no second node to ACK) → controller stops transmitting |
| `TWAI_MODE_NO_ACK` | Self-test mode for a solo node on the bench |

## Exercise

Pick two of the three and build them side by side in one ESP-IDF project,
each in its own FreeRTOS task (module 2-01).

**RMT + I2S:** a sound-reactive light. Task A reads the microphone with
`i2s_channel_read()` and computes an average level over each buffer; it
pushes that level into a queue. Task B drains the queue and drives eight
WS2812 LEDs as a VU meter. Confirm the LEDs still update smoothly when you
add a deliberately slow priority-3 task that spins for 50 ms every second —
proof the RMT timing is hardware-driven and not at the scheduler's mercy.

**TWAI:** wire two ESP32s to one bus with two transceivers and 120 Ω at both
ends. Node A transmits a counter at ID `0x100` every 100 ms; node B receives
it and echoes it back at ID `0x101`. Then set node A's filter to accept only
`0x101` and confirm it stops seeing its own traffic. Finally, disconnect
node B, log `twai_get_status_info()` every second, and watch the transmit
error counter climb toward bus-off — then recover with
`twai_initiate_recovery()` once B is back.
