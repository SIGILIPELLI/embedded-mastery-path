# PCB Design Basics (KiCad)

Every module so far assumed a board already existed. This module is about
designing one: taking a schematic from "MCU + sensor + power" to
manufacturable Gerber files using KiCad, the free/open-source EDA tool most
independent embedded engineers use. There is no PCB fabricated or tested for
this course — this module is careful, technically accurate **manual
review** of the design workflow and the electrical rules that matter, stated
plainly wherever a claim can't be hardware-verified from here.

## The KiCad workflow, end to end

1. **Schematic capture** (`.kicad_sch`) — place symbols for every component,
   wire logical connections, assign footprints (the physical
   pad/hole pattern each symbol maps to on the board).
2. **Electrical Rules Check (ERC)** — KiCad checks the schematic itself: unconnected
   pins, conflicting outputs driving the same net, missing power symbols.
   Fix every ERC warning before moving on — it's far cheaper to fix here
   than after fabrication.
3. **PCB layout** (`.kicad_pcb`) — import the netlist, place footprints
   physically, route copper traces between them, add a ground plane.
4. **Design Rules Check (DRC)** — checks the physical board: trace-to-trace
   clearance, drill sizes, whether traces are wide enough for their current,
   whether the design matches the fab's manufacturing capabilities.
5. **Gerber + drill file export** — the actual manufacturing output: one
   file per copper layer plus solder mask, silkscreen, and drill data, sent
   to a fab house (JLCPCB, PCBWay, OSH Park, etc).

## Power supply decoupling — the rule that's easy to state, easy to skip

Every IC that switches current (which is every digital IC, every clock
edge) needs a **decoupling capacitor** placed as close as physically
possible to its power pin — typically 100 nF ceramic, sometimes paired with
a larger 1-10 µF bulk capacitor per power rail. The capacitor supplies the
instantaneous current spike a chip draws when its internal logic switches,
faster than current can travel from a power supply several centimeters away
through trace inductance.

**Placement matters as much as value.** A 100 nF cap placed 3 cm from the
IC's power pin, connected by a long, narrow trace, is measurably less
effective than the same capacitor placed directly adjacent with a short,
wide connection — the trace inductance between the cap and the pin it's
protecting is exactly what the cap is there to defeat. This is a genuinely
common beginner PCB mistake: the decoupling cap is *on the schematic* and
*on the board*, just too far away to do its job, and the symptom (a reset
that only happens under load, an MCU that browns out on a motor start) looks
nothing like "capacitor placement."

## Trace width and current capacity

A copper trace has finite resistance and heats up under load; trace width
(and copper thickness, usually 1 oz/ft² for hobbyist boards) sets how much
current it can carry before its temperature rise becomes a problem. This is
governed by IPC-2152 (the modern standard; the older IPC-2221 charts are
still commonly used as a conservative approximation). As a working rule of
thumb for 1 oz copper, external traces: roughly 0.25 mm (10 mil) per amp for
a modest (~10°C) temperature rise — but the real number depends on copper
weight, ambient temperature, internal vs. external layer, and allowed
temperature rise, and should come from an IPC-2152 calculator, not a
memorized ratio. Power and ground traces carrying more than a few hundred
milliamps are the ones worth explicitly checking; signal traces (I2C, SPI,
GPIO) essentially never carry enough current for width to matter — their
constraints are signal integrity (length matching, impedance for
high-speed lines) instead.

## Ground planes and return current

A solid ground plane on at least one layer isn't just "more copper" —
it gives every signal a low-inductance return path directly underneath it,
which matters even for supposedly slow signals whenever a fast edge (any
digital transition, even at 100 kHz I2C, has real high-frequency content in
its edges) is involved. Splitting a ground plane under a component that
routes across the split forces return current to detour around the gap,
increasing loop area and radiated emissions — a specific, well-documented
DRC-adjacent issue that KiCad won't flag automatically because "the ground
net is still connected," just via a much longer path than necessary.

## What a DRC check actually catches, and what it doesn't

DRC catches physical/geometric problems: traces too close together for the
fab's minimum clearance, drill holes too small, unconnected nets, footprint
courtyard overlaps. It does **not** catch: wrong component value chosen for
the circuit, decoupling caps placed too far from their pin (geometrically
"fine," electrically wrong), or a ground plane split in a way that hurts
signal integrity but doesn't violate any clearance rule. DRC passing is a
necessary, not sufficient, condition for a working board.

## Traps in PCB design

- **Footprint mismatch**: the schematic symbol is electrically correct but
  assigned the wrong footprint (wrong pin pitch, wrong package) — ERC and
  DRC both pass, and the board arrives with a component that physically
  cannot be soldered on.
- **Silkscreen vs. copper confusion**: labeling pin 1 on the silkscreen
  without double-checking it against the footprint's actual pin-1 marker
  can produce a board that's populated backwards on the first assembly run.
- **Skipping ERC "warnings"** (not just errors) — an unconnected input pin
  left floating on a CMOS IC is a warning, not an error, but floating CMOS
  inputs draw unpredictable current and can even oscillate.
- **No test points**: a board with no accessible pads for probing key
  signals (power rails, reset, key buses) turns any bring-up issue into a
  much harder debugging session than a few strategically placed 1 mm pads
  would have allowed.

## How It Actually Works

**Why decoupling capacitor placement is a physics problem, not a schematic
one**: every trace, however short, has parasitic inductance (a rough rule of
thumb: ~1 nH per mm for a typical PCB trace), and inductance opposes rapid
changes in current (`V = L × di/dt`). A digital IC's internal logic can
switch many gates simultaneously on a clock edge, demanding a current pulse
that rises in a nanosecond or less — exactly the kind of rapid `di/dt` that
even a few nanohenries of trace inductance turns into a real voltage sag at
the chip's power pin, because the supply several centimeters away simply
cannot deliver current fast enough through that inductance to keep up. A
decoupling capacitor works by being a *local* charge reservoir: it supplies
that instantaneous current from charge already stored close by, over a path
short enough that its own loop inductance is negligible compared to the
distance back to the regulator. Moving the same-value capacitor a few
centimeters away doesn't change its capacitance, but it reintroduces exactly
the trace inductance between the cap and the pin that the capacitor was
placed there to bypass — which is why "the cap is on the board" is not the
same claim as "the cap is doing its job."

**Why splitting a ground plane hurts even when DRC passes**: a fast digital
edge's return current doesn't take "a" path back to the source — it
distributes itself along the path of *least impedance*, which for a trace
routed over a solid ground plane is the strip of plane directly beneath it
(mutual inductance between trace and plane makes that the lowest-impedance
route by a wide margin at the frequencies present in a fast edge). If the
plane has a slot or split under that trace, the return current is forced to
detour around the gap — a physically longer path that encloses a much larger
loop area. Loop area is the direct multiplier in the electromagnetic
radiation from any current loop, so a plane split routed across doesn't
create a DRC-visible defect (the net is still one contiguous electrical
node) but does directly increase radiated emissions and crosstalk — a purely
electromagnetic consequence of loop geometry, invisible to a tool that only
checks clearances and connectivity.

**Why trace width has a current limit at all**: copper has finite
resistivity, so any current through a trace dissipates power as heat
(`P = I²R`), and a narrower trace has higher resistance per unit length —
more heat per amp. That heat has to escape through the PCB's fiberglass
substrate (a poor thermal conductor) and the surrounding air/copper, and if
generation outpaces dissipation, the trace's temperature keeps climbing
until it approaches the copper-to-substrate bond's failure point or the
solder mask's degradation temperature. IPC-2152's charts are empirically
derived exactly from measuring this heat balance across many trace
geometries, copper weights, and layer positions (internal traces run hotter
for the same current because they're insulated by substrate on both sides
instead of open to air) — which is why "0.25 mm per amp" is only ever a
rough starting point: the real constraint is a thermal equilibrium, not a
fixed ratio.

## Cheat sheet

| Concept | Detail |
|---|---|
| ERC | Checks the **schematic**: unconnected pins, conflicting drivers, missing power symbols |
| DRC | Checks the **physical board**: clearance, drill sizes, fab capability limits |
| Decoupling cap | ~100 nF ceramic per IC power pin, placed as close as physically possible |
| Trace width for current | Use an IPC-2152 calculator; ~0.25 mm/A (1oz Cu, external) is only a rough starting estimate |
| Ground plane | Gives every signal a short return path; avoid routing across plane splits |
| Gerber/drill files | The actual manufacturing output sent to a fab house |
| Reality check | This module is manual technical review — no board from this course was fabricated or tested |

## Exercise

Take a schematic for a simple sensor board (MCU + one I2C sensor + one LDO
regulator) and, without opening KiCad, list on paper: every net that needs
a decoupling capacitor and where on the board (relative to which pin) each
one should sit; every net that should route with a matched ground-plane
return path underneath it; and at least two things an ERC/DRC pass would
NOT catch about this design that you'd still want to manually check before
sending it to fabrication. This module's PCB module (3-10, the sensor
board project) will let you apply this list to an actual layout.
