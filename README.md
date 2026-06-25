# Vacuum-tube-curve-tracer
An app to drive programmable lab power supplies to log tube curves

Coded for the KUAIQU 300v 1A lab powersupply from Aliexpress.
# Tube Curve Tracer

A Windows app (C# / WinForms, .NET Framework 4.8) that drives a programmable
0–300 V DC bench supply over its serial protocol to sweep a vacuum tube's anode
voltage, log the resulting anode current, and plot a live **family of
plate-characteristic curves** (anode current Ia vs anode voltage Va, one curve
per grid bias Vg). Results export to **CSV** and **PNG**.

---

## ⚠️ SAFETY — READ THIS FIRST

**This project controls voltages up to 300 V DC. That is enough to KILL you.**

- 300 V DC across the chest can stop your heart. DC also tends to make you
  clamp onto the conductor rather than let go.
- Never touch the tube pins, anode cap, wiring, or supply terminals while the
  output is enabled. The app's **EMERGENCY STOP** button cuts both supplies'
  outputs, but do not rely on software as your only safety layer — physically
  power down before touching anything.
- Tube anodes and plate resistors get hot. Large filter/coupling caps can stay
  charged after power-off; bleed them before handling.
- Use insulated leads, keep one hand in your pocket, and work on a
  non-conductive surface. If you are not comfortable around lethal voltages,
  don't build this.

You accept all risk for what you do with this software.

---

## What it produces

The classic output on a tube datasheet: a set of curves, each at a fixed grid
bias, with anode voltage on the X axis and anode current on the Y axis. As the
grid is driven more negative the curves sit lower and flatten — the tube passes
less current for the same anode voltage. The app sweeps the anode at each grid
bias you specify and draws one Series per curve.

---

## Hardware / wiring

| Supply / input        | Connects to                                              |
|-----------------------|----------------------------------------------------------|
| Anode supply (req.)   | Tube **plate (anode)** through a sense path; current is read back from the supply. |
| Grid supply (optional)| Tube **control grid**. The grid needs a *negative* bias. |
| Filament              | Tube heater, from a **separate** AC/DC filament supply.  |

**Grid polarity:** these supplies source positive voltage. To get a negative
grid bias, wire the grid supply's output **reversed** (its "+" to circuit
ground / cathode, its "−" to the grid) so the grid sits negative with respect to
the cathode. In the app you enter the grid bias as a **magnitude**; the supply
is commanded with that magnitude and the curves are labelled `Vg = -X.X V`.
Double-check the actual polarity at the tube with a meter before enabling output.

**Filament voltage** is entered in the app purely as a logged reference value —
the app does **not** drive the heater. Power the heater from its own supply at
the tube's rated voltage (e.g. 6.3 V or 12.6 V) before running a sweep.

**Pentodes / tetrodes** (6V6, EL84, …) also need a **screen grid (g2)** supply,
typically held at a fixed voltage. This app drives anode + control-grid only, so
power the screen from a separate fixed supply and note its value. The presets
flag this where relevant.

**Current resolution:** the protocol reports current to 1 mA. That's fine for
power tubes but coarse for small-signal types (a 12AX7 may draw well under a
milliamp at low anode voltage), so expect quantised/steppy low-current readings
on those.

---

## Serial protocol (decoded from the manufacturer doc)

- **Line settings:** 8 data bits, no parity, 1 stop bit (8N1). Baud one of
  1200 / 2400 / 4800 / 9600 / 19200; **default 9600**.
- **Frame (ASCII, 13 chars):** `<` + `FF` (2-char function) + `DDDDDD`
  (6-digit value) + `AAA` (3-digit device address) + `>`.
- **Value scaling:** field value = real × 1000 (3 decimals). 250 V → `250000`,
  0.183 A → `000183`.
- **Commands (PC→supply):** `01` set voltage, `02` query voltage, `03` set
  current, `04` query current, `07` output ON, `08` output OFF, `091` connect,
  `092` disconnect.
- **Responses (supply→PC):** `11OK…` set-V ack, `13OK…` set-I ack, `12……`
  voltage reading, `14……` current reading. The leading flag char on a reading
  is the regulation mode: **`1` = constant-voltage (CV)**, **`C` = constant-
  current (CC)**. There is no checksum — frames are delimited only by `<` `>`.


## Using it

1. **Connect supplies.** Pick the COM port + baud + device address for the anode
   supply and click Connect. If you have a second supply for the grid, tick
   "Use grid supply", set its port/address, and connect it too. Otherwise leave
   it unticked and enter a single manual grid voltage.
2. **Pick the tube.** Choose a preset (12AX7, 12AU7, 12AT7, 6V6, EL84, 300B) or
   Custom. The preset fills in sensible filament / anode-range / grid-range /
   current-limit defaults — **review them**, they're starting points, not gospel.
3. **Set the anode sweep:** start V, stop V, number of steps, dwell time per
   point (the settle delay before sampling), and *samples per point*. Each point
   takes that many current readings after the dwell and uses the **median**,
   which rejects the brief current spikes the supply's output capacitor produces
   when the voltage steps up. If you still see spikes, raise the dwell and/or the
   sample count. Estimated run time is shown.
4. **Set the grid bias:** in supply mode, a start/stop/step magnitude generates
   one curve per bias value; in manual mode, a single bias value gives one curve.
5. **Output files:** tick auto-save to drop a timestamped CSV + PNG into the
   chosen folder when the sweep finishes. You can also export either at any time.
6. **Run.** Start begins the sweep; the chart fills in live. Stop ends gracefully
   after the current point; **EMERGENCY STOP** cuts all outputs immediately.

### CSV columns

`GridVoltage_V, AnodeV_set_V, AnodeV_meas_V, AnodeCurrent_mA, State, Timestamp`

(plus a metadata header with tube name, filament voltage, and sweep settings).
`State` is `CV` or `CC` — a `CC` row means the supply hit its current limit at
that point and the reading is clamped, not a true characteristic point.

---

## How it maps to the original request

- **Anode sweep + live current logging + big graph** → the anode supply is swept
  low→high in N steps with a configurable dwell, current is read at each point,
  and everything plots live on the chart panel.
- **Optional second supply for grid bias, else manual grid voltage** → the
  "Use grid supply" toggle switches between sweeping a real grid supply (a curve
  per bias) and a single manually-entered bias.
- **Filament voltage input** → entered and logged as a reference (heater is
  powered separately).
- **Save as CSV and image** → auto-save on completion plus manual Export CSV /
  Export PNG buttons.


