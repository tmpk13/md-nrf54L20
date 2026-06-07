# microbit_nrf54l20 - build notes

micro:bit-style board around an nRF54L20 (QFN-52), built with the KiCad MCP.
Schematic only: symbols + footprints linked, connectivity by net labels (no wires),
components placed in non-overlapping functional blocks.

## KiCad library situation on this machine
- `kicad-cli` is 9.0.8; the MCP's `list_symbol_libraries` / `list_libraries` read a
  near-empty KiCad 10.0 lib-table, so they look almost empty.
- The full standard libraries ARE installed at `/usr/share/kicad/symbols` and
  `/usr/share/kicad/footprints`, and the KiCad 9.0 global tables
  (`~/.config/kicad/9.0/{sym,fp}-lib-table`) reference them. So `kicad-cli`
  (render / ERC / netlist) resolves `Device:*`, `Package_DFN_QFN:*`, etc. fine.
- Footprint strings stored in the schematic are just text; they only need to
  resolve at PCB time, so standard-library names work without MCP registration.
- Custom `nRF54L20` symbol lives in `microbit_nrf54l20.kicad_sym`, registered
  project-scope (writes `./sym-lib-table`). QFN-52 footprint:
  `Package_DFN_QFN:QFN-52-1EP_7x8mm_P0.5mm_EP5.41x6.45mm` (exposed pad = pad 53).

## get_schematic_pin_locations Y-axis bug (important)
- The MCP `get_schematic_pin_locations` returns `schematic_y = origin_y + lib_y`,
  but KiCad instantiates a library symbol with the Y negated
  (`schematic_y = origin_y - lib_y`). So the reported Y is mirrored about the
  component origin.
- Consequence: net labels placed at the reported coordinates attach to the pin's
  vertical-mirror partner. Pins that differ only in X (e.g. the LED matrix
  anode/cathode) are unaffected; anything with pins spread in Y (the MCU, the
  edge header, the sensor, the LDO, USB) gets scrambled.
- Fix applied: mirror every net label's Y about its nearest component origin
  (`new_y = 2*origin_y - y`). This leaves the already-correct labels untouched.
- X is reported correctly (verified against the matrix), only Y is wrong.

## Verifying connectivity with label-only nets
- Labels placed exactly on a pin endpoint connect in KiCad without any wire,
  which is what "use nets, do not wire" asks for.
- The MCP's `list_schematic_nets` and `generate_netlist` only trace wire
  segments, so they report 0 / "no connections" even when labels are correct.
  Use the real tool for ground truth:
  `kicad-cli sch export netlist --output out.net file.kicad_sch`
  then check that each net's member pins (and their `pinfunction`) match intent.

## Design choices (from the review pass)
- Battery (2xAAA) diode-ORs onto the +3V3 rail through D27, NOT into the LDO
  input: an AP2112K-3.3 can't regulate 3.3V from ~2.7V (3V cell minus Schottky),
  so the SoC runs directly off the battery rail (nRF runs down to 1.7V) while
  USB feeds the same rail via the LDO; D27 blocks backfeed.
- The symbol pin assignments are the REAL datasheet pinout (nRF54LM20A, QFN52 /
  QGAA, Table 82 in nRF54LM20A_nRF54LM20B_Datasheet_v1.0.pdf, p.1234+): pin number,
  name and type match the datasheet. GPIO are P0.xx/P1.xx/P2.xx; dedicated pins are
  VDD(6,42,51), VSS(35,45,53=EP), VBUS/DECUSB/D+/D-/TXRTUNE(17-21), SWDCLK/SWDIO/
  nRESET(31-33), ANT(34), DECRF(36), XC1/XC2(37/38), DCC(43)/DECD(44)/DECA(46).
  Notable: the 32.768 kHz crystal is on P1.20/P1.21 (XL1/XL2, pins 48/49); DECRF is
  tied to DECA per the datasheet; the DCDC inductor L1 sits between DCC and VDD;
  P2.00 (pin 52) is left as a spare NC.
- Application function is carried by the NET labels, never the pin name - e.g. real
  pin P1.05 sits on net COL2, P0.08 on net SDA, P1.29 on BTN_A.
- The 5x5 LED matrix was removed per request; the ROW*/COL* nets remain as plain
  U1<->edge-connector GPIO nets.

## Misc
- Sending many schematic-mutating MCP calls in one assistant turn is safe: the
  server serialises them (verified by re-counting components afterwards).
- Page size is set to `(paper "User" 850 600)` so the spread-out blocks fit in
  the rendered/exported sheet; the MCP preserves the header on later writes.
