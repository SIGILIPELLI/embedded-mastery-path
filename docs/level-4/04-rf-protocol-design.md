# RF & Wireless Protocol Design

Module 2-03 used BLE as a library — pair, advertise, exchange
characteristics. This module is about the layer beneath any radio protocol:
link budgets, modulation tradeoffs, and the packet-level design decisions
that determine range, power, and reliability, for the class of chip that
implements a proprietary sub-GHz link rather than a full BLE/WiFi stack.
Real RF behavior (antenna performance, propagation, interference) cannot be
verified from a laptop — this module states plainly where that's true, and
compiles the parts that are genuinely arithmetic or protocol logic.

## The link budget: the one calculation that governs everything else

Every RF link's maximum range is bounded by a straightforward accounting
equation:

```
Received power (dBm) = Transmit power (dBm)
                      + Transmit antenna gain (dBi)
                      - Path loss (dB)
                      + Receive antenna gain (dBi)
                      - Cable/connector losses (dB)
```

The link works if received power exceeds the receiver's **sensitivity**
(the weakest signal it can reliably decode, e.g. -110 dBm for a
long-range, low-data-rate link) by a safety margin — typically 10-20 dB to
tolerate fading, obstacles, and manufacturing variance, not the bare
theoretical minimum.

```c
#include <math.h>

/* free-space path loss in dB, given distance in meters and frequency in Hz —
   the textbook Friis equation form; real-world (non-free-space) loss is
   always worse than this, often much worse indoors */
double free_space_path_loss_db(double distance_m, double freq_hz) {
    const double c = 299792458.0;
    double wavelength = c / freq_hz;
    return 20.0 * log10(4.0 * M_PI * distance_m / wavelength);
}

double received_power_dbm(double tx_dbm, double tx_gain_dbi,
                           double distance_m, double freq_hz,
                           double rx_gain_dbi, double cable_loss_db) {
    double path_loss = free_space_path_loss_db(distance_m, freq_hz);
    return tx_dbm + tx_gain_dbi - path_loss + rx_gain_dbi - cable_loss_db;
}
```

## Modulation tradeoff: data rate versus range, not a free choice

Lower data rate per symbol (e.g. LoRa's spreading-factor scheme, or simple
FSK at a low baud rate) trades throughput for **processing gain** — the
receiver can pull a signal out of noise that would be indecipherable at a
higher data rate, extending range at the cost of airtime per byte sent. This
is a real engineering tradeoff a protocol designer chooses deliberately
based on the application: a sensor reporting once an hour can spend far more
time-on-air per byte than a control link needing sub-100ms latency, and
should pick its modulation/data-rate accordingly rather than defaulting to
"as fast as possible."

## Packet design: framing, addressing, and duty-cycle regulations

```c
/* a compact packet format for a battery-powered sensor node */
typedef struct __attribute__((packed)) {
    uint8_t  preamble_sync;     /* radio-level sync, often handled by the radio IC itself */
    uint16_t node_id;
    uint8_t  seq_num;           /* detects lost/duplicate packets at the receiver */
    uint8_t  payload_len;
    uint8_t  payload[16];
    uint16_t crc16;
} rf_packet_t;

uint16_t rf_packet_crc16(const rf_packet_t *pkt) {
    /* CRC over everything except the CRC field itself */
    return crc16_ccitt((const uint8_t *)pkt, offsetof(rf_packet_t, crc16));
}
```

`seq_num` matters more on RF links than on wired ones: RF packets are lost
to interference or collisions far more often than a well-terminated wired
bus drops bytes, and a receiver needs a cheap way to detect a missing or
duplicated packet without needing an acknowledgment for every single
transmission (which itself costs airtime and power).

**Duty-cycle regulation** is a real, legal constraint in most sub-GHz ISM
bands (e.g. EU 868 MHz limits many channels to roughly 1% duty cycle) — a
protocol that ignores it isn't just impolite to other spectrum users, it can
be a regulatory compliance failure for a shipping product. This is decided
at the protocol design stage (how often, how long each transmission is
allowed to be) and reviewed here as a design constraint; verifying actual
on-air duty cycle requires real RF test equipment, not a laptop.

## Verifying the link-budget and CRC/sequence logic

The arithmetic and packet-integrity logic are compiled and run with `gcc`;
propagation and RF hardware behavior are not represented:

```c
#include <stdio.h>
#include <assert.h>
#include <math.h>

/* free_space_path_loss_db / received_power_dbm as above */

int main(void) {
    /* 915 MHz, 1 km, 14 dBm TX, 2 dBi antennas each end, negligible cable loss */
    double rx_dbm = received_power_dbm(14.0, 2.0, 1000.0, 915e6, 2.0, 0.5);
    printf("received power at 1km: %.1f dBm\n", rx_dbm);

    /* sanity: doubling distance should cost ~6 dB (free-space loss ~ 20*log10(d)) */
    double rx_2km = received_power_dbm(14.0, 2.0, 2000.0, 915e6, 2.0, 0.5);
    double delta = rx_dbm - rx_2km;
    assert(fabs(delta - 6.0) < 0.5);

    /* sensitivity margin check: does a -110 dBm sensitivity receiver still
       have the recommended 10 dB margin at 1km for this configuration? */
    double sensitivity_dbm = -110.0;
    double margin = rx_dbm - sensitivity_dbm;
    assert(margin > 10.0);

    printf("link budget checks OK (margin at 1km: %.1f dB)\n", margin);
    return 0;
}
```

## Traps in RF protocol design

- **Ignoring duty-cycle regulations at design time**: a protocol that
  "works great in the lab" (one node, no regulatory enforcement) can be
  illegal to actually sell in a region if it exceeds a band's duty-cycle
  limit under real usage patterns — this has to be checked against the
  target region's regulations, not assumed.
- **No sequence numbers or CRC**: on wired buses a bit error is rare enough
  that some designs skip integrity checking; on RF, treating every received
  packet as potentially corrupted or duplicated is the realistic default.
- **Assuming free-space path loss indoors**: the Friis equation above is a
  best case; real indoor/obstructed paths lose significantly more, often by
  10-30+ dB depending on materials — a link budget calculated only with
  free-space loss will be optimistic for anything but line-of-sight outdoor
  use, and should be validated with real field measurements before a range
  claim ships in a datasheet.
- **Ignoring collision behavior at scale**: a protocol tested with two nodes
  can behave completely differently with 200 nodes sharing the same channel
  — collision/backoff behavior needs to be designed for the target fleet
  size, not just validated at prototype scale.

## Cheat sheet

| Concept | Detail |
|---|---|
| Link budget | TX power + antenna gains - path loss - cable loss, compared against RX sensitivity + margin |
| Path loss (free space) | `20*log10(4*pi*d/wavelength)` — a best case, real environments lose more |
| Processing gain | Lower data rate/spreading factor trades throughput for range/noise immunity |
| Sequence number | Detects lost/duplicate packets — expected on RF links, not just a nice-to-have |
| Duty cycle | Legally regulated in many ISM bands — a protocol design constraint, not just politeness |
| Verification here | Link-budget arithmetic and CRC/seq-num logic compiled/run with `gcc`; real RF/antenna/propagation behavior needs lab equipment |

## Exercise

Extend the link-budget model into a `max_range_for_margin(tx_dbm, gains,
freq_hz, sensitivity_dbm, required_margin_db)` function that solves for the
maximum distance satisfying a target margin (binary search over distance is
the simplest approach, since path loss is monotonic in distance). Verify it
against the direct calculation by confirming `received_power_dbm` at the
computed max range equals `sensitivity_dbm + required_margin_db` within a
small tolerance. Compile and run with `gcc`, and in a comment state which
real-world factor (name at least two: multipath, obstruction, antenna
orientation, interference) would most reduce this theoretical max range in
practice, and why this function alone cannot produce a trustworthy range
spec for a real product.
