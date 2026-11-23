# Partial Reconfiguration of BPSK, QPSK and 16-QAM Modulators on Artix-7

Three digital modulators (BPSK, QPSK and 16-QAM) share two reconfigurable regions of one Artix-7 FPGA. Two modulators run at the same time, and either one can be replaced by the third while the other keeps running, by loading a small partial bitstream.

Built with Vivado 2022.2 (Dynamic Function eXchange, DFX) for the Nexys A7-100T board (XC7A100T-CSG324-1).

## Why partial reconfiguration

Building all three modulators side by side uses chip area even when only two are needed at a time. With partial reconfiguration, two reserved regions on the chip each hold whichever modulator is needed, and only that region is rewritten when the choice changes.

## Features

- Three modulators with identical ports: BPSK, QPSK and 16-QAM (Gray-coded)
- Two reconfigurable partitions (`u_rp1`, `u_rp2`) in one static region
- Three configurations, so every module can sit in every partition
- Six partial bitstreams, plus one full bitstream per configuration
- Live demo on the board: 4 switches set the symbol, 16 LEDs show the outputs
- GUI-only Vivado flow, no Tcl scripting needed

## How it works

| Term | Meaning |
|---|---|
| Static region | Never reconfigured: clock, reset, switches, LED output |
| Reconfigurable partition (RP) | A reserved region that can be rewritten while the rest runs |
| Reconfigurable module (RM) | A design that can be loaded into an RP (all have the same ports) |
| Pblock | The rectangle on the chip that reserves an RP's area |
| Partial bitstream | Rewrites only one partition |

Every module has the same interface: a clock, a reset, a 4-bit symbol input and two 8-bit signed outputs (I and Q). Both partitions receive the same symbol, and a register shows the I or Q output of each on the LEDs.

### Configurations

| Configuration | `u_rp1` | `u_rp2` |
|---|---|---|
| 1 | BPSK | QPSK |
| 2 | QPSK | 16-QAM |
| 3 | 16-QAM | BPSK |

### Modulators

| Modulator | Input bits used | Output (I, Q) |
|---|---|---|
| BPSK | bit 0 | I = ±100, Q = 0 |
| QPSK | bit 0 gives I, bit 1 gives Q | I, Q = ±70 |
| 16-QAM | bits 3–2 give I, bits 1–0 give Q | Gray levels: 00 = −90, 01 = −30, 11 = +30, 10 = +90 |

## Repository layout

Suggested layout (adjust to match your folders):

```
rtl/            bpsk_mod.v, qpsk_mod.v, qam16_mod.v, top.v
constraints/    pins.xdc (Nexys A7-100T), floorplan.xdc (pblocks)
sim/            testbench
docs/           report, slides, photos
```

## Requirements

- Vivado 2022.2 (the DFX flow is included in the standard editions; check your license)
- Nexys A7-100T board (XC7A100T-CSG324-1) and a USB cable

For another Artix-7 board, change the part and the pins in `pins.xdc`, and redraw the pblocks in the Device view.

## Build (Vivado GUI)

1. Create an RTL project for `xc7a100tcsg324-1`. Add the files in `rtl/` and `constraints/`.
2. In `top.v`, instantiate the same module twice as `u_rp1` and `u_rp2`.
3. Enable DFX (Tools menu), create the partition definition `rp_mod` from `u_rp1`, and make `u_rp2` a second reconfigurable partition. Add the BPSK, QPSK and 16-QAM modules as reconfigurable modules.
4. In the DFX Wizard, create the three configurations from the table above (config 1 is the parent run, the others are child runs).
5. Run synthesis and check in the Device view that each pblock is one clock-region row high and the two do not overlap. Each pblock needs `RESET_AFTER_RECONFIG` and `SNAPPING_MODE = ROUTING`.
6. Run implementation for all configurations and generate bitstreams. Each run folder gets a full `top.bit` and two `*_partial.bit` files.

## Running on the board

1. In Hardware Manager, program the full `top.bit` from the config 1 run, then press `btnC` (reset).
2. Set the symbol with `SW3`–`SW0`. Set `SW15` down to show the I output, or up to show Q.
3. `LED7`–`LED0` show `u_rp1` and `LED15`–`LED8` show `u_rp2`, as 8-bit two's-complement values.
4. To swap a module, right-click the device, choose Program Device, select a partial bitstream, and press `btnC` afterwards.

### Expected LED values (`SW15` down, `SW3`–`SW0` = 0000)

| State | LED 15–8 / LED 7–0 | Meaning |
|---|---|---|
| Config 1 (BPSK in `u_rp1`, QPSK in `u_rp2`) | `0100 0110` / `0110 0100` | +70 / +100 |
| After loading 16-QAM into `u_rp2` | `1010 0110` / `0110 0100` | −90 / +100 |
| Config 2 (QPSK in `u_rp1`, 16-QAM in `u_rp2`) | `1010 0110` / `0100 0110` | −90 / +70 |

Each partial bitstream is about 312 KB.

## Results

Configuration 1 and the swap to configuration 2 were tested on the board with the values above. Swapping one module changed only that partition's output, while the other kept running.

[Add photos of the board and LEDs here, e.g. `docs/board_config1.jpg`.]

## Limitations

- Partial bitstreams are loaded from the PC over JTAG.
- No decoupling logic: a partition's output can glitch while it is being reloaded.
- Reconfiguration time has not been measured.
- Partial bitstreams are valid only with the full bitstream from the same build.

## Future work

Self-reconfiguration, where the FPGA loads its own partial bitstreams through ICAP, using a MicroBlaze processor, an AXI HWICAP block and bitstreams stored in block RAM, with switches choosing the module. This is in progress and not yet working.

## References

- AMD, UG909: Vivado Design Suite User Guide, Dynamic Function eXchange
- Digilent, Nexys A7 Reference Manual

## Author

[Sree Nidi M.S], [Shiv Nadar University], [2026]

