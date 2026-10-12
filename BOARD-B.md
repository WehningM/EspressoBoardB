# Board B — Mains Switching and Independent Protection

**This board carries 230 V. Treat every block below as hazardous unless it is explicitly
marked SELV.** Nodes HN-3 (mains switching), the mains side of HN-4 (isolation barrier) and
the electronic part of HN-5 (independent protection, LC-20: PR-1…PR-9). It is the
counterpart of board A's J9 control header (`PCB-B` in `espresso.gaphor`).

Board A must stay testable from USB alone (REQ-SYS-106). Board B is the opposite: none of it
works without mains, so it is bring-up module M2 and needs an isolation transformer, an RCD
and a current-limited supply on the bench (see the end of this file).

Drafted 2026-10-12 from model v2.3, `BOARD-A.md` and `Board-A-Audit-2026-10-08.md`. Nothing
here is as-built yet. Items marked **Proposed** are design choices made for this document,
not decisions recorded in the model. Confirm them before layout.

---

## What board B must do (from the model)

| Function | Model source | Block |
|---|---|---|
| Switch the heater by integral cycle, 10 ms granularity, zero-cross SSR | HN-3, PR-4, REQ-SYS-100, REQ-SUB-HYD-009 | 4 |
| Fire the pump SCR once per mains cycle; no single failure gives full-wave | HN-3, CON-1, REQ-SYS-102, REQ-SUB-HYD-001/-006 | 5 |
| Switch the three-way valve, tray path when unpowered | HN-3, REQ-SYS-074, REQ-SUB-HYD-005 | 6 |
| Switch a spare actuator | REQ-SYS-108, REQ-SUB-HYD-011 | 6 |
| Master contactor in series with heater, pump and valve | PR-6, REQ-SYS-004, REQ-SUB-PROT-013/-015/-018 | 3 |
| Windowed watchdog holds the contactor only while the hold signal is valid | PR-5, REQ-SYS-082, REQ-SUB-PROT-010/-011/-012 | 7 |
| Emergency cut, mechanical, latching, not through the compute node | PR-7, REQ-SUB-PROT-014/-016/-017 | 8 |
| E-Fast burden, comparator and trip latch gate the heater in hardware | PR-1/-2/-3, REQ-SYS-001/-002/-083/-097/-098, REQ-SUB-PROT-001…-006 | 9 |
| Protection supply independent of the compute node supply | PR-8, REQ-SUB-PROT-020/-021 | 2 |
| Isolated status back to board A: zero-cross, E-Fast trip, contactor state | HN-4, PR-9, REQ-SYS-080/-091, REQ-SUB-PROT-005/-032 | 10 |
| All five drive signals arrive from board A through optocouplers / reinforced relays | HN-4, `PCB-B` | 4–7 |

Everything **not** on board B: the thermal fuse (inside the FTH II, PR-10), the OPV (PR-11,
mechanical), all sensors and all logic. Board B contains no software (HN-5, LC-20 "hw only").

---

## Board B architecture

Three insulation regions. Getting the boundaries right is the whole job of this board.

```
                 ┌───────────────────── REGION M: mains power ──────────────────────┐
 IEC inlet ──F── │ L/N line side ──┬── SELV PSU (Block 1) ──────────────────────────┼──► board A J1 (+5 V), J2 (+12 V)
 (fuse+switch)   │                 ├── PR-8 protection supply (Block 2) ──┐         │
                 │                 ├── zero-cross sense (Block 10) ──opto─┼─────────┼──► J9 ZERO_CR
                 │                 └── contactor PR-6 (Block 3) ──► L'/N' load side │
                 │                          L'/N' ──► SSR ──► FTH II heater          │
                 │                          L'/N' ──► SCR + diode ──► ULKA pump      │
                 │                          L'/N' ──► valve relay ──► 3-way valve    │
                 │                          L'/N' ──► spare relay ──► spare          │
                 │                          L'/N' ──► load-side sense ──opto────────┼──► J9 CNTCTR_AUX
                 └─────────────────────────────────────────────────────────────────┘
                 ┌──────────── REGION P: protection island (mains-referenced!) ──────┐
                 │ PR-8 supply · E-Fast burden PR-1 · comparator PR-2 · latch PR-3    │
                 │ watchdog PR-5 · coil driver · e-stop loop PR-7 · SSR input drive   │
                 └──────────────────────── reinforced barrier ───────────────────────┘
                 ┌──────────────────────── REGION S: SELV ──────────────────────────┐
                 │ J9 mate to board A · opto LEDs / opto outputs · valve/spare coils  │
                 └───────────────────────────────────────────────────────────────────┘
```

- **Region M** — line and load-side mains conductors.
- **Region P** — low-voltage logic that is **referenced to mains** because the E-Fast
  burden is (PR-1 "mains referenced", PR-2 "reference derived on the mains side"). It looks
  like 12 V logic but it is at mains potential. Treat it as hazardous: no test-clip habits
  from board A.
- **Region S** — SELV, galvanically tied to board A's GND through J9's two GND pins.

| Boundary | Insulation | Creepage / clearance target (230 V, PD2, OVC II, FR-4 IIIb) |
|---|---|---|
| M or P ↔ S | **Reinforced** (REQ-SYS-080) | ≥ 6.4 mm creepage, ≥ 3.0 mm clearance by the numbers; **design to 8 mm and add a milled slot** under every opto and relay that straddles it |
| M ↔ P | Functional (P is already hazardous) | ≥ 3 mm |
| L ↔ N within M | Functional | ≥ 3 mm; wider around the fuse and MOV |

Every part straddling the S boundary must itself be rated for reinforced insulation:
optocouplers to IEC 60747-5-5 / VDE 0884 with wide-lead (≥ 8 mm) option, and relays with
reinforced coil–contact insulation. A standard-pitch PC817 or a sugar-cube relay does **not**
qualify.

---

## Block 1 — SELV power supply (feeds board A) — **Proposed**

The model never places the supply that feeds board A's J1 (+5 V) and J2 (+12 V), but OB-20
and REQ-SYS-091 assume a mains PSU with known hold-up time. Board B is the natural home: it
already has mains, and the PSU's output is what Region S needs anyway to light the
optocoupler LEDs (see Block 4 — J9 carries no supply pin).

| Item | Qty | Spec | Note |
|---|---|---|---|
| AC-DC module, 12 V | 1 | Mean Well IRM-20-12 (12 V / 1.67 A, reinforced, 85 °C derated) or equivalent | Feeds board A J2, the opto LEDs and the valve/spare relay coils |
| 5 V source | 1 | IRM-05-5 module, **or** a 12 → 5 V buck | Feeds board A J1. ESP32 Wi-Fi peaks ≈ 0.5 A |
| Input fuse | 1 | T 1 A (or per module datasheet), 5×20 holder or TR5 | Line side |
| Hold-up bulk cap | 1–2 | 1000–2200 µF on +12 V, sized after OB-20 is set | Buys log-stop time for REQ-SYS-091 |
| Output connectors | 2 | 2-pin, **same family and pinout as board A J1/J2** | One cable per rail; avoid the J1/J11 mis-plug from audit #4 |

**12 V budget:** HMI ≤ 0.5 A + board A K1/K2 2 × 37.5 mA + board B opto LEDs (≈ 6 × 10 mA)
+ board B valve/spare coils (2 × ≈ 35–45 mA) ≈ 0.75 A. 1.67 A leaves headroom.

**Connect on the line side** (before the contactor), after the inlet fuse. The master
disconnect must interrupt pump, heater and valve (REQ-SYS-004), not the compute node.

## Block 2 — Protection supply PR-8 (Region P)

| Item | Qty | Spec | Note |
|---|---|---|---|
| AC-DC module, 12 V | 1 | IRM-05-12 (5 W) or similar, **separate from Block 1** | REQ-SUB-PROT-020: independent of the compute node supply |
| Fuse | 1 | T 500 mA | |
| Local 5 V / 3.3 V LDO | 1 | e.g. 78L05 | For comparator, latch and watchdog IC |
| Decoupling | as needed | | |

- **Line side, before the contactor.** The watchdog must be powered to close the contactor
  in the first place (REQ-SUB-PROT-021: functional while board A is off, resetting or
  flashing).
- **Size it for the contactor coil.** A 25 A 2-pole power relay coil draws ≈ 2 W at 12 V,
  so 5 W is the minimum class.
- The module's 0 V is tied to the E-Fast reference (neutral side, per OI-3), which is what
  makes Region P mains-referenced. Its output terminals therefore need Region-P clearances,
  not SELV ones.

## Block 3 — Master contactor PR-6

| Item | Qty | Spec | Note |
|---|---|---|---|
| Power relay / contactor, **2-pole** | 1 | ≥ 25 A at 250 VAC, 12 VDC coil. **Proposed:** Omron G7L-2A-TUB-J-CB DC12 (25 A, 2 NO, tab terminals, mounting bracket) | Chassis-mounted, wired with 6.3 mm Fastons |
| Coil driver | 1 | logic-level N-MOSFET or NPN on Region P | Driven only by the watchdog output (Block 7) |
| Fast-dropout network | 1 | flyback diode **in series with a 24–33 V Zener** across the coil | A plain flyback diode slows dropout; the Zener is what makes OB-24 short |
| Coil connector | 1 | 2-pin, mains-rated housing | Coil wiring sits in Region P |

- **Two poles, not one.** REQ-SUB-PROT-018 makes the contactor the *only* galvanic
  isolation, because the SSR leaks through its snubber. Breaking only L leaves the loads tied
  to N.
- **Rating.** Total load ≈ 8.9 A heater + pump + valve ≈ 9.5 A. PR-6 asks for > 16 A, which
  the 25 A part covers with margin for the inrush of a cold element.
- **Keep the 9 A path off the PCB.** Proposed wiring: inlet fuse → contactor pole 1 → SSR →
  FTH II (internal thermal fuse) → contactor pole 2 → N, all in 1.5 mm² wire with Fastons.
  The PCB only taps the load side (L′/N′) for pump, valve, spare and sensing, all ≤ 2 A,
  through a T 2 A branch fuse. If you would rather have the contactor on the board, the
  heater tracks need 2 oz copper and ≥ 5 mm width plus tinned solder-mask openings. That is
  doable but it is a different board.
- **The valve and spare are fed from L′/N′.** That is what gives REQ-SUB-PROT-015 (valve
  unpowered, so tray path, whenever the contactor is open) by construction.

## Block 4 — Heater SSR drive PR-4

| Item | Qty | Spec | Note |
|---|---|---|---|
| SSR | 1 | **Zero-cross**, 3–32 VDC input, ≥ 25 A / 600 V, genuine part (Crydom D2425, Omron G3NA-225B). Off-board, panel-mounted | Not a no-name "Fotek" |
| Heatsink | 1 | ≈ 1.3 V × 8.9 A ≈ 11.5 W at full duty → ≤ 3 K/W, thermal paste | REQ-SYS-107: heatsink inside a hot enclosure |
| Optocoupler, reinforced | 1 | wide-lead DC-input opto | S → P barrier |
| LED resistor | 1 | from +12 V (Block 1), ≈ 1 kΩ for ≈ 10 mA | Board A MOSFET sinks it |
| Gate transistor | 1 | in series with the opto output, enabled only while the PR-3 latch is **not** tripped | REQ-SYS-083: hardware gate |
| SSR control connector | 1 | 2-pin, Region P | |

**How the drive is wired.** Board A's SSR output is an open-drain AO3400 (board A Block 8),
so it can only sink current. Board B's +12 V (Block 1) → LED resistor → opto LED → J9 SSR
pin → board A MOSFET → GND. When board A is reset or flashing, its 10 kΩ gate pull-downs keep
the MOSFET off, so the opto is dark and the SSR is off (REQ-SUB-HYD-010).

On the Region P side, PR-8 +12 V → opto transistor → latch-enable transistor → SSR input. Both
must conduct for the heater to see power: firmware asks, and the E-Fast latch allows.

- **Zero-cross SSR is required, not optional.** Integral cycle control at 10 ms granularity
  (REQ-SYS-100) assumes the device only turns on at a zero crossing.
- **The SSR never counts as off.** Its snubber leaks a few mA. Isolation is the contactor's
  job (REQ-SUB-PROT-018).

## Block 5 — Pump SCR drive

| Item | Qty | Spec | Note |
|---|---|---|---|
| SCR | 1 | BT151-800R or similar, TO-220, 800 V / 12 A | Pump current ≈ 0.3–0.4 A, no heatsink needed. Confirm against the CON-1 datasheet |
| Series diode | 1 | 1N5408 (1000 V / 3 A) | **REQ-SYS-102 / REQ-SUB-HYD-006** |
| Gate driver, reinforced | 1 | **random-phase** opto-triac (MOC3023 class, wide-lead/VDE option) | **Not** a zero-cross type (MOC306x) — that would refuse to fire mid-cycle |
| Gate series resistor | 1 | 330 Ω–1 kΩ, 0.5 W | Per driver datasheet |
| Gate–cathode resistor | 1 | 1 kΩ | Prevents dv/dt false triggering |
| RC snubber | 1 | 100 Ω + 47–100 nF X2 across the SCR | Inductive load |
| MOV across pump | 1 | S10K275 or similar | Optional, against coil turn-off spikes |
| LED resistor | 1 | from +12 V, sized for the driver's IFT with margin | Board A sinks it |

**Why two devices in series.** The ULKA has no internal diode (CON-1) and must only ever see
half-wave. If the SCR fails short, the diode still blocks the negative half-cycle. If the
diode fails short, the SCR still blocks it. Either alone would be a single point of failure.

**Firing timing** belongs to board A (hardware-timer ISR on the zero-cross clock, AR-1,
REQ-SYS-081). Board B's only timing duty is to keep its delays small and constant: the
opto-triac turns on in a few µs, which is well inside OB-19 (< 100 µs).

**Gate pulse length.** With an inductive pump coil, anode current rises slowly. The firmware
gate pulse must last until the SCR's latching current is reached — typically a few hundred
µs, possibly longer. Measure this at M2 and record it as a firmware constant.

## Block 6 — Valve and spare actuator

| Item | Qty | Spec | Note |
|---|---|---|---|
| Power relay, reinforced coil–contact | 1 + 1 DNP | 12 VDC coil, ≥ 5 A / 250 VAC, reinforced. **Proposed:** Omron G2RL-1-E DC12 (or Finder 40.31.9.012) | Label the second footprint "spare actuator" |
| Flyback diode | 2 | 1N4148W | Across each coil |
| Branch fuse | 1 | T 2 A, shared with the pump | L′ side |
| Valve connector | 1 | 2-pin (L, N) mains | Plus PE to the valve body if it is metal (REQ-SYS-015) |
| Spare connector | 1 | 2-pin (L, N) mains | |

**The coil loop is fed through board A's floating contact.** Board A's K1/K2 contacts arrive
on J9 as isolated COM/NO pairs with no reference to board A GND (board A Block 8,
REQ-SUB-HYD-011). Board B wires: +12 V (Block 1) → J9 valve COM → board A K1 → J9 valve NO →
board B valve-relay coil → GND. **Do not tie either valve or spare pin to J9's GND.** The
loop is entirely powered by board B, so the board A contact only ever switches ≈ 40 mA at
12 V.

**Use the board-B relay's NO contact.** De-energised means the valve is unpowered, which
means tray path (REQ-SYS-074, OI-1 closed). Check this on the bench with the valve itself,
the same way audit #2 checks K1/K2.

**Why a relay and not a triac here.** A triac with its snubber leaks enough current to keep
some small solenoid valves humming or half-pulled. A relay contact is truly open, so the
unpowered position is reliable.

## Block 7 — Windowed watchdog PR-5 and contactor hold

| Item | Qty | Spec | Note |
|---|---|---|---|
| Optocoupler, reinforced | 1 | S → P | Board A CNTCTR_HOLD sinks the LED |
| Window watchdog | 1 | **Proposed:** TI TPS3430 family **or** a discrete AC-coupled charge pump; see below | On PR-8 supply |
| AND gate / transistor pair | 1 | watchdog OK **and** pulses present **and** e-stop closed → coil on | |
| Timeout setting | — | OB-23 (open) | Window bounds set by capacitor or resistor straps |

Requirements this block must meet:

- Hold the contactor only while a periodic hold signal is present (REQ-SUB-PROT-010), and
  open it within OB-24 once it stops (REQ-SUB-PROT-011).
- A stuck-high **or** stuck-low line must not hold it (REQ-SUB-PROT-012). The input must
  therefore be AC-coupled or edge-sensitive, never level-sensitive.
- A pulse train that is too fast (a free-running timer) must not hold it either. That is
  what the *window* — rejecting early kicks as well as late ones — is for.
- **Verify the power-up behaviour of whatever IC you choose.** Many watchdog ICs release
  their output *before* the first kick. That would close the contactor at power-on with no
  hold signal and violate REQ-SUB-PROT-010. The AND with a "pulses seen" charge pump
  removes this risk regardless of the IC.

**Firmware coupling (record in the firmware notes):** SV-5 must toggle the hold line by
software from core 1, never from an LEDC/RMT/PWM peripheral. A peripheral keeps running after
a crash, and that is exactly the free-running timer the window exists to reject.

## Block 8 — Emergency cut PR-7

| Item | Qty | Spec | Note |
|---|---|---|---|
| E-stop switch | 1 (off-board) | Mushroom head, **latching**, twist- or pull-to-release, NC contact, rated ≥ 250 VAC | REQ-SUB-PROT-016 (mechanical latch), -017 (distinguishable by touch) |
| Connector | 1 | 2-pin, Region P | |

The NC contact sits **in series with the contactor coil** after the coil driver, so it opens
the coil circuit directly. No path goes through board A or the watchdog logic
(REQ-SUB-PROT-014).

The e-stop wiring is at Region P potential, which is mains-referenced. Use mains-rated wire
and a mains-rated switch, and keep its leads away from the HMI.

## Block 9 — E-Fast sense, comparator and trip latch (PR-1 … PR-3)

| Item | Qty | Spec | Note |
|---|---|---|---|
| Burden resistor | 1 set | "27 kΩ class", **value OPEN (OI-3)**. Use ≥ 2 resistors in series, each rated for mains surge (MELF or ≥ 0.6 W THT, ≥ 350 V working) | REQ-SYS-097 / REQ-SUB-PROT-006: dimensioned for 230 V, not the 120 V datasheet case |
| Rectifier + RC filter | 1 set | Settles within 40 ms of the threshold being crossed | REQ-SUB-PROT-003 |
| Comparator + reference | 1 | Threshold OB-22, about 12–15 V across the burden; reference from PR-8 | Hysteresis so it does not chatter at the threshold |
| Trip latch | 1 | Set by the comparator. **Reset only by loss of load-side mains**, taken from the Block 10 load-side sense | REQ-SUB-PROT-002 |
| Gate removal | — | Latch output disables the Block 4 enable transistor within 1 ms | REQ-SUB-PROT-004 |
| Status opto, reinforced | 1 | P → S, to J9 E_FAST_IN (board A IO35) | REQ-SUB-PROT-005 |
| E-Fast connector | 1 | to the FTH II E-Fast terminal, mains-rated | |

- **The E-Fast signal only exists while the heating track is energised.** That is why
  firmware spreads heater duty (REQ-SYS-096, TH-5): the comparator is blind during off
  half-cycles, and OB-5 (100 ms premise) bounds that blindness. Board B's filter time
  constant has to fit inside the same budget.
- **PE leakage.** The burden circuit may add at most OB-25 to protective earth leakage, and
  the whole sense circuit must stay below 0.5 mA (REQ-SYS-098). Don't add Y-capacitors to PE
  anywhere on board B without counting them against this.
- **Latch reset by load-side loss.** The latch stays set until firmware sees the trip,
  enters Fault, stops kicking the watchdog, the contactor opens, and the load side goes dead.
  A cooling element can therefore never re-arm the heater on its own.
- **Read, never inhibit** (REQ-SUB-PROT-032). The only connection to board A is the status
  opto output. There must be no path from J9 into the comparator, latch or gate.
- **Get the Ferro Techniek designer pack (OI-3) before layout.** The burden value, the
  terminal to reference against and the threshold all come from it. This block cannot be
  finalised without it.

## Block 10 — Isolated status outputs to board A (HN-4, PR-9)

Board A provides a 10 kΩ pull-up to 3.3 V and an RC filter on each input (board A Block 9).
Board B only has to pull the line low through a reinforced opto transistor; it supplies no
voltage to board A's inputs.

| Signal (J9) | Board A pin | Board B source | Opto |
|---|---|---|---|
| ZERO_CR | IO36, 100 Ω / 100 pF | L/N **line side** (before the contactor) | AC-input opto (H11AA1 / LTV-814 class), wide-lead |
| E_FAST_IN | IO35, 1 kΩ / 10 nF | PR-3 latch output | DC opto |
| CNTCTR_AUX | IO44 (RXD0) | **Proposed:** L′/N′ load-side voltage sense | AC-input opto |

**Zero-cross on the line side** keeps the clock alive while the contactor is open. Board A
needs it for mains-loss detection at all times (REQ-SYS-091), not only while brewing.

- Input resistors: 230 VRMS across ≥ 2 series resistors, each ≥ 350 V working voltage
  (e.g. 2 × 47 kΩ 1 W ≈ 0.56 W total, ≈ 3.5 mA peak).
- **Check the CTR.** The opto must sink ≥ 0.33 mA (3.3 V / 10 kΩ) at the LED current that
  occurs at the pulse edges, at end-of-life CTR. Low-CTR parts at low LED current give a wide,
  temperature-dependent pulse.
- **Edge speed.** A phototransistor with a 10 kΩ load has rise and fall times of tens of µs.
  That is a constant delay, which is fine, but it varies with CTR and temperature, which eats
  into OB-19. If M2 measurements show too much jitter, swap in a Schmitt-output opto (H11L1
  class) fed from a bridge and powered from board B's SELV +5 V. Its open-collector output
  still works with board A's 3.3 V pull-up.

**Contactor state from load-side voltage, not an aux contact.** The model only says
"contactor auxiliary state". Sensing L′/N′ voltage is stronger: it reports what the loads
actually see, so it also catches a welded contact. The same signal resets the PR-3 latch. The
G7L proposed in Block 3 has no aux contact anyway.

**Fail-safe polarity (Proposed).** Opto **on** = healthy (E-Fast armed, load side live).
Then a dead protection supply or a broken wire reads as "tripped" rather than as "fine".
Confirm the expected polarity against the board A firmware before layout.

## Block 11 — Mains entry and protection

| Item | Qty | Spec | Note |
|---|---|---|---|
| IEC inlet with switch and fuse | 1 (chassis) | C14, T 10 A, double-pole switch | Not on board B; feeds the board through J-MAINS |
| MOV | 1 | S14K275, line side after the fuse | |
| PE | — | Chassis stud, star point. Board B needs PE only if the E-Fast circuit or a metal valve body needs it | REQ-SYS-015 Class I |
| Fuse holders on board | 3 | PSU T 1 A, PR-8 T 500 mA, load branch T 2 A | Clip-in 5×20 or TR5 |

A-8 (the mains supply) is RCD-protected by assumption. Board B must not rely on that.

## Block 12 — Mechanical and layout

- **Mounting holes:** 4 × M3, isolated (no copper) inside Region M and Region P, and a
  keepout ring sized to the reinforced distance wherever a screw or standoff sits near
  Region S. Board A shipped without holes (audit #7); don't repeat it here, since board B
  carries the heavier parts.
- **Slots:** a milled slot under each barrier-straddling opto and relay, plus a continuous
  silk line marking the barrier.
- **Silkscreen:** hazard symbol and "230 V" on Regions M and P; "SELV" on Region S; the
  function name at every connector.
- **Connector families:** mains connectors in one family (5.08 / 7.62 mm screw terminals or
  6.3 mm Fastons), SELV connectors in another (matching board A's). Nothing mains-rated
  should mate with anything SELV.
- **Copper:** 2-layer, 1.6 mm, 1 oz is enough if the heater current stays in wire (Block 3).
  Load-branch tracks ≥ 1.5 mm.
- **Temperature:** place board B away from the FTH II and keep the SSR heatsink on the
  chassis, not on the PCB. Rate parts for the measured air temperature (REQ-SYS-107).
- **No test points on Region P or M.** Probe only with a differential probe or an isolated
  scope, never with a ground-referenced scope clip.

---

## Connector summary (Proposed numbering)

| # | Purpose | Region | Pins |
|---|---|---|---|
| J1 | Control link to **board A J9**: 2 × 6 IDC box header, 2.54 mm, ribbon cable | S | 12 |
| J2 | +5 V out → board A J1 (JST XH; board A pin 1 = GND) | S | 2 |
| J3 | +12 V out → board A J2 (screw terminal; board A pin 1 = +12 V) | S | 2 |
| J4 | Mains in L / N from the inlet | M | 2 (+PE if needed) |
| J5 | Contactor load side L′ / N′ back to the board | M | 2 |
| J6 | Contactor coil | P | 2 |
| J7 | Emergency cut (NC) | P | 2 |
| J8 | SSR control + / − | P | 2 |
| J9 | E-Fast sense from the FTH II | P/M | 2 (per OI-3) |
| J10 | Pump out (SCR + diode, N′) | M | 2 |
| J11 | Three-way valve out | M | 2 |
| J12 | Spare actuator out | M | 2 |

## J1 — mating pinout with board A J9

Read from `Espressomaschine Board A.kicad_pcb` (pad → net, traced to the parts behind each
net) on 2026-10-12.

**Board A J9 is a 2 × 6 IDC box header, 2.54 mm, right angle** (BOOMELE 2.54-2*6P, LCSC
C9141). Fit the same header on board B and join them with a 12-way 1.27 mm ribbon and two IDC
sockets. A straight ribbon between two box headers keeps pin 1 on pin 1, and the keyed shroud
makes it impossible to plug in rotated. The ribbon carries SELV only. It never crosses the
barrier: it ends in Region S on board B.

| Pin | Board A net | Board A side (as built) | Direction | Board B side |
|---|---|---|---|---|
| 1 | → R37 1 kΩ → `E_FAST_IN` | IO35, R34 10 kΩ pull-up to 3V3, C16 10 nF | B → A | Block 9 status opto collector; emitter to GND |
| 2 | `GND` | | — | Region S GND (= Block 1 PSU 0 V) |
| 3 | → R38 100 Ω → `ZERO_CR` | IO36, R35 10 kΩ pull-up, C17 100 pF | B → A | Block 10 zero-cross opto collector |
| 4 | → R39 1 kΩ → `CNTCTR_AUX` | IO44 (RXD0), R36 10 kΩ pull-up, C18 100 nF | B → A | Block 10 load-side-sense opto collector |
| 5 | `R1_COM` | K1 COM (pads 5/6) — **valve** | floating | Block 6 valve loop: +12 V in |
| 6 | `R2_COM` | K2 COM (pads 5/6) — **spare** | floating | Block 6 spare loop: +12 V in |
| 7 | `R2_NO` | K2 NO (pad 1) — **spare** | floating | Block 6 spare-relay coil (+), coil (−) to GND |
| 8 | `R1_NO` | K1 NO (pad 1) — **valve** | floating | Block 6 valve-relay coil (+), coil (−) to GND |
| 9 | `CNTCTR_HLD` | Q5 AO3400A drain, sinks | A → B | Block 7 opto LED cathode; anode via R to +12 V |
| 10 | `GND` | | — | Region S GND |
| 11 | `SCR` | Q4 AO3400A drain, sinks | A → B | Block 5 opto-triac LED cathode; anode via R to +12 V |
| 12 | `SSR` | Q3 AO3400A drain, sinks | A → B | Block 4 opto LED cathode; anode via R to +12 V |

Notes from the trace:

- **The series resistors and pull-ups are on board A**, with the pull-up on the MCU side of
  the series resistor. When a board B opto transistor pulls pin 1 or 4 low, the pin sees
  ≈ 3.3 V × 1 k / 11 k + V_CE(sat) ≈ 0.5 V — a clean low. Board B needs no pull-ups and no
  series resistors of its own on these three lines.
- **The sink current per opto is only ≈ 0.3 mA** (3.3 V / 11 kΩ). Size the CTR check in
  Block 10 against that, not against a standard 10 mA test condition.
- **Pin 3 (zero-cross) has only 100 Ω in series**, which keeps the edge fast for OB-19.
  Don't add filtering on board B.
- **The drive pins (9, 11, 12) have no pull-up on board A.** Nothing on J9 supplies voltage,
  so board B must provide the LED anode supply (+12 V from Block 1). This confirms the
  assumption in Block 4. AO3400A is rated 30 V, so 12 V on the drain is fine.
- **K1 is the valve and K2 the spare.** The PCB only names them R1/R2; this follows
  `BOARD-A.md` Block 8, where the second relay is the "spare actuator". The NO/NC
  continuity check from audit #2 still applies before you trust pads 1/8.
- **Valve and spare pins sit next to each other and cross over** (COM on 5/6, NO on 8/7).
  Easy to swap in the board B schematic — label them by net, not by position.
- **Neither COM pin is tied to GND on board A.** That confirms the floating pairs; board B
  provides both ends of the loop.

## Power connectors to board A J1 / J2

Also from the board A PCB. **The two connectors have opposite pin orders**, so the cables
need care:

| Board A | Part | Pin 1 | Pin 2 | Board B side |
|---|---|---|---|---|
| J1, +5 V in | JST XH 2P right angle (S2B-XH-A-1), → F1 1.1 A PTC | **GND** | **+5 V** | JST XH cable from Board B J2 |
| J2, +12 V in | 5.0 mm screw terminal, KF301-5.0-2P, → F2 0.75 A PTC | **+12 V** | **GND** | Screw-terminal wire pair from Board B J3 |

Board A's J1 shares its JST XH housing with the sensor connectors J11/J12/J14 (audit #4). A
5 V cable plugged into a sensor socket won't damage anything but gives wrong readings. Colour
or label the board B power cable.

---

## Open decisions that change this board

| Question | Affects | Source |
|---|---|---|
| **E-Fast designer pack (OI-3):** burden value, reference terminal, threshold OB-22 | Block 9 entirely — blocks layout | OI-3, OB-22, PR-1 |
| Where does the SELV PSU live? (Proposed: board B) | Block 1, J2/J3, how opto LEDs are powered | not in model; OB-20 |
| Heater open-circuit detection (REQ-SYS-009, -079): no hardware path exists. Infer it from the thermal response, or add a heater-current sense opto on board B? | Possibly one more J9 signal → board A respin | model gap |
| Watchdog timeout OB-23 and window bounds | Block 7 component values | OB-23 |
| Contactor drop-out OB-24 | Block 3 Zener value, contactor choice | OB-24 |
| PE leakage budget OB-25 | Block 9, any EMI filtering | OB-25, REQ-SYS-098 |
| Mains-loss → log stop time OB-20 | Block 1 hold-up capacitance | OB-20 |
| ULKA Model E current and inrush (CON-1 has no rating) | Block 5 SCR, snubber, fuse | CON-1 |
| Three-way valve voltage, power, AC/DC | Block 6 relay and fuse | not in model |
| Status polarity expected by firmware | Block 10 | board A firmware |

## Model updates suggested

These are gaps in `espresso.gaphor` that this board exposes. They don't block layout:

- Add a block for the SELV PSU (location, hold-up time) and trace OB-20 to it.
- PCB-B says "carries the optocouplers for every signal crossing to Board A". Valve and spare
  cross through board B's reinforced relays instead. Reword it to "optocouplers or reinforced
  relays".
- Name the CNTCTR_AUX source: an aux contact or load-side sense (the proposal here).
- Add a heater-continuity mechanism to satisfy REQ-SYS-079, or state that it is inferred
  thermally.
- Add a Layer 3 "Board B realisation" diagram next to D-15, once this is built.

## Bring-up (M2) — safety before function

1. **Before mains ever touches it:** check every barrier with an insulation tester (500 V DC,
   expect > 100 MΩ) between Region S and Regions M/P, with board A **disconnected**.
2. **Region P first, from a bench supply in place of PR-8:** watchdog window, AND gate, coil
   driver, e-stop, and the comparator and latch with a signal generator in place of the
   burden. Most of the protection logic can be proven this way without mains at all.
3. **Then mains, through an isolation transformer, behind an RCD, with a dim-bulb or
   current-limited supply.** Use dummy loads (incandescent lamps) in place of heater and pump.
4. Measure: zero-cross pulse width and jitter (OB-19), SCR latching time, contactor drop-out
   (OB-24), and the time from hold-signal loss to contactor open.
5. Valve rests in the tray path with the contactor open and with the relay de-energised
   (REQ-SYS-074). Check both.
6. Only then connect board A, and finally the real FTH II and E-Fast.

## One honest note

Block 9 depends on a document you don't have yet (OI-3), and it is the block REQ-SYS-001/-002
lean on hardest. Everything else on this board can be laid out now. If the designer pack is
slow to arrive, route board B with the E-Fast section as a self-contained island with
generous footprints (burden pads that accept 2–3 series parts, a comparator footprint with
both threshold divider resistors free), so filling in the values later is a component swap
rather than a respin.
