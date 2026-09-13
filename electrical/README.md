# electrical/

Reference material for the servo driver board, plus the FSR sensing circuit
this project adds.

```
electrical/
└── ws_sch.pdf      Waveshare Bus Servo Adapter (A) schematic, 1 page
```

## What the board is

The SO-100 uses an **off-the-shelf** Waveshare *Serial Bus Servo Driver Board*,
sold on its wiki as **Bus Servo Adapter (A)**. One per arm, so **two** for a
leader + follower pair.

It does two jobs: USB-to-serial bridging, and converting that to the
**half-duplex TTL bus** the STS3215 servos speak. It is not a motor controller —
the servos contain their own drivers and closed-loop control. The board just
carries packets and power.

## This board is not open hardware

`ws_sch.pdf` is the vendor's published schematic, retrieved from
[the Waveshare wiki](https://www.waveshare.com/wiki/Bus_Servo_Adapter_(A)).
It is **reference only**:

| Available | Not available |
|---|---|
| Schematic (PDF) | Gerbers |
| 3D model (STEP, in a zip) | KiCad / Altium / EAGLE source |
| SDKs and demo code | Netlist, BOM, footprints |

Seeed's equivalent XIAO bus servo board is the same story — schematic PDF only,
no source, no OSHW marking. **There is no publicly available editable PCB design
for either board.** If you need one, you redraw it from this PDF.

## What the schematic shows

Extracted from the PDF: 17 capacitors, 14 resistors, 5 diodes, 5 ICs.

- **U1, 16 pins**, with nets named `CH_RXD` / `CH_TXD` — a CH343-family USB-UART bridge
- **USB Type-C** with the usual CC1/CC2 pulldowns
- **DC barrel jack** plus a screw terminal for the servo power rail
- **A switching buck regulator** — `SW` / `BST` / `FB` pins and ferrite bead `FB1`, producing the `3V3` rail
- **MMBT small-signal transistors** in a resistor network — the half-duplex direction-control circuit, and the only non-obvious part of the design

A capture of that schematic is an evening's work if you ever want your own board.

## Wiring the pair

Each arm is an independent chain. Nothing crosses between them except the host.

```
  wall ──> 5V PSU ──> barrel jack ─┐
                                   ├─ Waveshare board ──> servo 1 ─> 2 ─> ... ─> 6
  host ──> USB-C ──────────────────┘        (daisy chain, 3-pin cables)

  x2 — one complete chain for the leader, one for the follower
  plus: FSR microcontroller ──> USB ──> host        (third USB device)
```

Three USB serial devices total. They enumerate as `/dev/ttyACM*` in arbitrary
order, which is why `software/required_libraries.md` sets up udev symlinks —
use `/dev/so100_follower`, `/dev/so100_leader`, `/dev/fsr_mcu`, never `ttyACM0`.

## Things that cost money to get wrong

**Set both jumpers to the `B` channel (USB).** Wrong position and the board
does not enumerate at all. This is the single most common bring-up failure, and
LeRobot's own troubleshooting calls it out.

**Match the power supply to the servos.** The SO-100 BOM specifies **7.4 V
servos with a 5 V supply**. 12 V servos exist and take a 12 V supply — they are
an SO-101 option, not an SO-100 one. Feeding 12 V into 7.4 V servos destroys
them, six at a time. Check the label on the servo, not the listing you bought from.

**The board cannot read sensors.** No spare ADC, no exposed GPIO — it is a
serial bridge and a power pass-through. Every sensor this project adds needs its
own microcontroller and its own USB cable. There is no way around this short of
designing a replacement board.

**Power before USB.** Plug the barrel jack in before configuring servos; the
board enumerates without it but the servos will not respond, which reads as a
cabling fault and sends you debugging the wrong thing.

## The FSR circuit

A voltage divider read by a microcontroller ADC — the FSR has no digital
interface.

```
  3V3 ──────[ FSR 402 ]──────┬────── ADC pin  (MCU)
                             │
                          [ 10k ]
                             │
  GND ───────────────────────┘
```

More force → lower FSR resistance → higher ADC reading.

`R_FIXED` sets the useful range: sensitivity peaks where `R_FIXED ≈ R_FSR`.
10 kΩ suits roughly 1–10 N. Saturating near full scale? Drop to 3.3 kΩ.
Readings hugging zero? Try 47 kΩ.

Any 3.3 V MCU with an ADC works — Seeed XIAO, Pi Pico, or an Arduino (use
`VCC = 5.0` on a 5 V Uno). Firmware, calibration and the host-side reader are in
`software/required_libraries.md`.

**An FSR is not a load cell.** Expect ±15–25% absolute accuracy even after
careful calibration. Good for relative comparison — "setting 40 squeezes twice
as hard as setting 20" — not for quoting Newtons. A compression load cell with
an HX711 is the upgrade if true force values matter.

## Nothing here has been built

No board has been designed, populated, or powered on for this project. The only
artifact is the vendor PDF. The FSR circuit above is specified but not
assembled.
