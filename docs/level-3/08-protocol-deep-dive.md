# Protocol Deep Dive (UART/I2C/SPI/CAN)

Levels 1 and 2 used UART, I2C, and SPI through library calls
(`Serial.print`, `Wire.write`, `SPI.transfer`). This module looks at what
those calls actually put on the wire — framing, timing, and error handling
— and adds **CAN**, the differential bus used throughout automotive and
industrial embedded systems, which none of the earlier modules touched.

## UART framing at the bit level

A UART frame is: a start bit (line pulled low), 8 data bits (LSB first, by
far the most common configuration), an optional parity bit, and one or two
stop bits (line high) — all timed by each side's own clock, with no shared
clock line, which is why matching baud rate matters more here than in any
other protocol in this module.

```c
/* software framing check — models what a UART receiver's state machine does,
   without needing real UART hardware */
typedef enum { UART_IDLE, UART_START, UART_DATA, UART_STOP } uart_state_t;

typedef struct {
    uart_state_t state;
    uint8_t bit_index;
    uint8_t shift_reg;
} uart_rx_t;

/* returns 1 and writes *out_byte when a full byte has been received.
   real UART hardware samples mid-bit via oversampling; this models only
   the framing state machine, one bit per call. */
int uart_rx_sample(uart_rx_t *rx, int line_level, uint8_t *out_byte) {
    if (rx->state == UART_IDLE) {
        if (line_level == 0) { rx->state = UART_DATA; rx->bit_index = 0; rx->shift_reg = 0; }
        return 0;
    }
    if (rx->state == UART_DATA) {
        rx->shift_reg |= (line_level << rx->bit_index);
        rx->bit_index++;
        if (rx->bit_index == 8) rx->state = UART_STOP;
        return 0;
    }
    if (rx->state == UART_STOP) {
        rx->state = UART_IDLE;
        if (line_level != 1) return 0;   /* framing error: stop bit wasn't high */
        *out_byte = rx->shift_reg;
        return 1;
    }
    return 0;
}
```

Baud rate mismatch of even a few percent accumulates bit-to-bit until, by
the last data bit, the receiver samples at the wrong point in the bit period
— explaining why UART "mostly works" at a mismatched rate and fails
intermittently rather than immediately: the error compounds across the byte.

## I2C: clock stretching and the arbitration trap

I2C's open-drain lines let a slow slave hold SCL low to pause the master
mid-transaction (**clock stretching**) — a legitimate part of the protocol,
but a driver that doesn't handle it (fixed-delay bit-banged I2C, for
example) can misread a stretched clock as a bus fault. On multi-master buses
(rare in embedded work but real in some designs), **arbitration** lets
multiple masters drive the bus simultaneously as long as they're sending the
same bits; the moment one master tries to send a `1` while another sends a
`0`, the one sending `1` sees the line low, recognizes it lost arbitration,
and backs off — a mechanism worth knowing exists even on projects that never
use multiple masters, because a bus fault symptom on a single-master design
sometimes turns out to be a phantom second driver (a poorly wired sensor
holding a line).

## SPI: mode mismatches are a clock-polarity/phase bug, not a wiring bug

SPI has four "modes" defined by two bits — CPOL (clock idle level) and CPHA
(which clock edge data is sampled on). Two devices wired correctly but
configured for different modes will often produce plausible-looking garbage
rather than an obvious failure, because the framing (chip select, byte
boundaries) still works — only the bit sampling point is wrong.

```c
/* models SPI mode 0 (CPOL=0, CPHA=0): sample on rising edge, data stable
   before it. Portable — no real SPI hardware needed to check the framing. */
uint8_t spi_shift_byte_mode0(uint8_t tx_byte, uint8_t (*clock_edge_rx)(int bit_out)) {
    uint8_t rx_byte = 0;
    for (int i = 7; i >= 0; i--) {                  /* MSB first is SPI's near-universal convention */
        int bit_out = (tx_byte >> i) & 1;
        int bit_in = clock_edge_rx(bit_out);        /* simulates one clock cycle */
        rx_byte |= (bit_in << i);
    }
    return rx_byte;
}
```

## CAN: arbitration by design, not by accident

CAN (Controller Area Network) is built for exactly the multi-master
contention I2C only tolerates as an edge case. Every node transmits its
message ID as part of arbitration; a `0` ("dominant") bit always wins over a
`1` ("recessive") bit when both are driven simultaneously, because the bus
physically behaves like a wired-AND. Lower numeric ID therefore always wins
arbitration — this is a deliberate design decision (lower ID = higher
priority), not a limitation.

```c
/* models CAN bit-wise arbitration: given two competing IDs, returns which
   one wins the bus (lower ID always wins) and at which bit position the
   loser first detects it lost arbitration */
typedef struct { int winner_is_a; int bit_lost_at; } arb_result_t;

arb_result_t can_arbitrate(uint32_t id_a, uint32_t id_b, int id_bits) {
    for (int i = id_bits - 1; i >= 0; i--) {
        int bit_a = (id_a >> i) & 1;
        int bit_b = (id_b >> i) & 1;
        if (bit_a != bit_b) {
            /* dominant (0) wins; whichever sent 1 loses right here */
            arb_result_t r = { .winner_is_a = (bit_a == 0), .bit_lost_at = i };
            return r;
        }
    }
    arb_result_t r = { .winner_is_a = 1, .bit_lost_at = -1 };  /* identical IDs — shouldn't happen on a real bus */
    return r;
}
```

## Verifying the framing/arbitration logic

Both the UART state machine and the CAN arbitration function are pure logic
and were compiled and run with `gcc`:

```c
#include <stdio.h>
#include <assert.h>
#include <stdint.h>

/* ... uart_rx_t / uart_rx_sample and arb_result_t / can_arbitrate as above ... */

int main(void) {
    /* UART: feed 0x41 ('A') LSB-first: start(0), 1,0,0,0,0,0,1,0, stop(1) */
    uart_rx_t rx = { .state = UART_IDLE };
    int bits[] = {0, 1,0,0,0,0,0,1,0, 1};
    uint8_t out = 0;
    int got = 0;
    for (int i = 0; i < 10; i++) {
        if (uart_rx_sample(&rx, bits[i], &out)) got = 1;
    }
    assert(got == 1 && out == 0x41);

    /* CAN: lower ID wins */
    arb_result_t r = can_arbitrate(0x123, 0x456, 11);
    assert(r.winner_is_a == 1);   /* 0x123 < 0x456 */

    r = can_arbitrate(0x456, 0x123, 11);
    assert(r.winner_is_a == 0);   /* now B (0x123) wins */

    printf("UART frame OK (0x%02X), CAN arbitration OK\n", out);
    return 0;
}
```

## Traps across these protocols

- **Baud mismatch that "mostly works"**: as above, small mismatches degrade
  gradually rather than failing outright — always verify baud rate against
  a known-good transmitter, not just "it printed something."
- **I2C clock stretching ignored by a bit-banged driver**: produces
  intermittent NACKs or truncated reads only under conditions that make the
  slave slow (busy processing, low supply voltage).
- **SPI mode mismatch**: garbage that looks like a wiring problem — check
  CPOL/CPHA against the datasheet before re-checking wiring.
- **CAN bus-off from ID collisions**: two nodes accidentally configured with
  the same message ID pass arbitration "successfully" (identical IDs never
  diverge) but then both transmit simultaneously, corrupting the frame — a
  bug arbitration doesn't protect against by design, only different-ID
  contention.

## Cheat sheet

| Concept | Detail |
|---|---|
| UART framing | Start(0), 8 data bits LSB-first, optional parity, stop(1) — no shared clock |
| I2C clock stretching | Slave holds SCL low to pause the master — legitimate, must be handled |
| I2C arbitration | Multi-master: whoever tries to send `1` while another sends `0` backs off |
| SPI CPOL/CPHA | Four modes; mismatch produces plausible garbage, not an obvious failure |
| CAN arbitration | Dominant bit (`0`) always wins; lower numeric ID = higher priority, by design |
| Verification here | UART/CAN logic compiled/run with `gcc`; real bus electrical behavior reviewed against spec only |

## Exercise

Extend `can_arbitrate` to a `can_arbitrate_n(uint32_t *ids, int n, int id_bits)`
that finds the overall winner among N competing IDs by running pairwise
arbitration bit-by-bit across all of them simultaneously (not just calling
the two-way version repeatedly), and add a test with at least 4 IDs where
the winner isn't the first or last in the array. Compile and run with `gcc`,
then in a comment explain why arbitration only needs one pass over the bit
positions regardless of how many nodes are contending — what property of
the dominant-bit rule makes it scale to N without pairwise comparisons.
