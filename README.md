# XEM7310 FPGA Breakout Board

Custom breakout / interface board for the
[Opal Kelly XEM7310](https://opalkelly.com/products/xem7310/) FPGA module, designed by
**Kunmo Kim** in Altium Designer at the **Berkeley Wireless Research Center (BWRC)**.

It mates to the XEM7310 through two mezzanine connectors and fans the FPGA's scan-chain,
analog, and digital signals out to the main/DUT board —
[pcb-for-128GSPS-adc-based-serdes](https://github.com/kunmok/pcb-for-128GSPS-adc-based-serdes) —
over a Samtec FFSD cable, while generating the bank I/O supplies and providing the level
shifting, clock probes, debug, and I²C support needed for bring-up and characterization.

![XEM7310 FPGA Breakout Board (v0.5)](Photo/pcb_v0p5.png)

> 📷 Photo shows the older **v0.5**; the latest revision fixes several bugs and uses a slightly
> cheaper BOM.
> 📄 Full schematic drawing: [`FPGA_breakout_board.pdf`](FPGA_breakout_board.pdf).

## At a glance

| | |
|---|---|
| FPGA module | Opal Kelly XEM7310 |
| PCB | 4-layer; fabricated & assembled by [JLCPCB](https://jlcpcb.com/) |
| `MC2` bank | LVCMOS12 — 1.2 V (TPS7A5701, adjustable) |
| `MC1` bank | LVCMOS33 — 3.3 V (TLV1117LV33) |
| Power in | 5 V — barrel jack / banana / VBUS, TVS-protected |
| Power-up order | +5 V → +1.2 V (VCCO_MC2) → +3.3 V (VCCO_MC1) |
| Main-board link | Samtec FFSD cable, ≈100 Ω, 100 Ω series-terminated |
| Debug / probe | JTAG (ILA), 2× SMA (50 Ω), I²C (Qwiic) |

## Power

Input is 5 V via the barrel jack (`J_JACK`), banana jacks (`VDCIN_CONN` / `GND_CONN`), or
VBUS, protected by a TVS diode (`D3`). Two on-board LDOs generate the FPGA bank I/O supplies:

| Bank | I/O standard | Supply | LDO |
|------|--------------|--------|-----|
| `MC2` | LVCMOS12 | 1.2 V — adjustable via `P1V2_POT` (Bourns 3266W-1-503, 50 kΩ) | TPS7A5701 (`U5`) |
| `MC1` | LVCMOS33 | 3.3 V | TLV1117LV33 (`U1`) |

`MC1` runs at 3.3 V because the on-chip **LT3074**'s enable input requires a 3.3 V signal.

> ⚠️ **Power-up sequence (external 1.2 V supply)** — required for correct operation, not to
> prevent damage. Follow the schematic order **+5 V → +1.2 V (VCCO_MC2) → +3.3 V (VCCO_MC1)**.
> Before driving 1.2 V into `VCCO_MC2`, isolate the XEM7310's own supply path (**do not touch
> FB8** per the schematic) to avoid back-driving the module. See Opal Kelly's
> [Powering the XEM7310](https://docs.opalkelly.com/xem7310/powering-the-xem7310/) for the
> module's power requirements.

The XEM7310 `PG_5V5` power-good flag crosses the voltage boundary via **two-step level
shifting** (two SN74LV1T34 buffers, `U_LS1` / `U_LS2`) — the reason the sequence above matters.

## Clock-test quad (`CLKQUAD`)

Optional scan-chain interface (`J_CLKQUAD`, Samtec STMM-107) for clock testing — used **only**
to measure **CLK jitter** and **ILRO lock status**, otherwise kept off. Because high scan speed
isn't needed, it runs through a dedicated **SN74AXC8T245** translator (`U2`) instead of LVCMOS
I/O, keeping the FPGA banks free.

## High-speed ADC data interface (SMA)

Two SMA connectors (Bel Cinch 142-0761-881, 26.5 GHz) on **50 Ω microstrip** carry:

- **`MEM_CLK`** — 100 MHz clock, FPGA → SerDes board.
- **`MEM_DATA_I`** — 100 Mbps retimed ADC output data, SerDes board → FPGA. Tested up to
  **200 Mbps**; likely capable of more, but untested beyond what this setup needed.

Together these two signals drive the SerDes board's **on-chip memory**: `MEM_CLK` clocks the
read-out while `MEM_DATA_I` streams the stored samples back to the FPGA, letting the FPGA read
out the retimed ADC data samples held in memory.

## FFSD cable

Scan / analog / digital signals reach the
[main board](https://github.com/kunmok/pcb-for-128GSPS-adc-based-serdes) over a Samtec FFSD
ribbon cable (`J_ESHF`, ESHF-125-01-L-D-SM), whose characteristic impedance was measured at
**≈100 Ω** in the lab. Matching **100 Ω series resistors** (`R4`–`R31`, 0402) sit close to the
connector on the driver side.

## Control & configuration

Most configuration signals originate from the **XEM7310 FPGA**, with a **Microchip
MCP2221A** USB-to-I²C/UART bridge providing an alternate path for some of them.
- I2C Control: Special thanks to @nonNoise for their [PyMCP2221A repository](https://github.com/nonNoise/PyMCP2221A), which was instrumental in implementing I2C control via the MCP2221A for this project.

**Scan chains** — all scan-chain signals are driven by the FPGA. The main board carries two
independent chains: a **5-pin** chain for analog-circuit control and a **7-pin** chain for
digital-circuit control. Both chains run at **12.5 Mbps**. The 7-pin chain runs on a
**two-phase non-overlapping clock**, making it immune to hold-time violations.

**LDO enable** — normally driven by the FPGA, but can alternatively be driven from the
**MCP2221A GPIO** pins.

**I²C** — normally handled by the **MCP2221A** at **100 kbps**; the FPGA can drive it as a
fallback.

## Connectors & key parts

| Function | Part | Designator(s) |
|----------|------|---------------|
| XEM7310 mezzanine | Samtec BTE-040-01-F-D-A (0.8 mm) | `MC1`, `MC2` |
| FFSD cable | Samtec ESHF-125-01-L-D-SM | `J_ESHF` |
| CLKQUAD header | Samtec STMM-107-02-L-D | `J_CLKQUAD` |
| SMA clk/data (26.5 GHz) | Bel Cinch 142-0761-881 | `J_MEM_CLK`, `J_MEM_DATA` |
| JTAG (ILA, 2×7, 2 mm) | Molex 87831-1420 | `J_JTAG` |
| I²C — Qwiic / headers | Würth 61300311121 / Samtec TSW | `P1` / `J_SCL*`, `J_SDA*` |
| USB-to-I²C/UART bridge | Microchip MCP2221A | `U3` |
| 1.2 V LDO (5 A, adj.) | TI TPS7A5701RTER | `U5` |
| 3.3 V LDO (1 A) | TI TLV1117LV33DCYR | `U1` |
| Level translator (CLKQUAD) | TI SN74AXC8T245RHLR | `U2` |
| Level shifter (PG_5V5) | TI SN74LV1T34DBVR | `U_LS1`, `U_LS2` |
| 100 Ω series term (0402) | Panasonic ERJ-2RKF1000X | `R4`–`R31` |
| TVS / jack / banana | SMAJ6.0CA / CUI PJ-102AH / Keystone 575-4 | `D3` / `J_JACK` / `*_CONN` |

Full BOM:
[`FPGA_breakout_board_BOM.xlsx`](Project%20Outputs%20for%20FPGA_breakout_board/FPGA_breakout_board_BOM.xlsx).

## Files

4-layer stack-up (Top / two inner planes / Bottom); DRC passes with **0 violations**.

```
BoardTop.SchDoc                Top-level schematic
FPGA_breakout_board_v1.PcbDoc  PCB layout
FPGA_breakout_board.PrjPcb     Altium project
FPGA_breakout_board.pdf        Schematic + fabrication drawings
FPGA_breakout_board.OutJob     Output-generation job
Component/                     Symbols, footprints, 3D/STEP models, vendor libs
Project Outputs for .../       Gerbers, NC drill, BOM, pick & place, DRC report
```

Open `FPGA_breakout_board.PrjPcb` in Altium Designer. Run the `.OutJob` to regenerate
manufacturing outputs; ready-to-send Gerbers are in `Project Outputs .../Gerber.zip`.

## Author

Designed by **Kunmo Kim** — [kunmo.kim0225@gmail.com](mailto:kunmo.kim0225@gmail.com) ·
[kunmok.github.io](https://kunmok.github.io)
