# Breathing front-end simulation (afe_sim)

This is a KiCad 10 / ngspice test project for the chest breathing channel. The ABD channel uses the same circuit, so the results apply to both.

In the schematic, R1 (2.2k) with V1 (a sine source in series) stands in for the stretch cord. V2 is the 3.3 V supply and V3 is VREF (1.65 V).

## Before you run it

MCP6001.lib is not in the repo because Microchip's licence doesn't allow us to redistribute it. To get the simulation working:

1. Download the model from microchip.com/product/MCP6001 (Design Resources > Spice Models).
2. Save it in this folder as MCP6001.lib.
3. In the op amp's simulation model, map the pins (symbol -> model): 1->5, 2->4, 3->1, 4->2, 5->3.
4. Set the simulator compatibility mode to "PSpice and LTspice".

## Results (Nov 24)

- Operating point: S, AMP_IN and VREF sit at 1.65 V, ADC_OUT at 1.643 V. Each channel draws 0.86 mA.
- AC: gain of 11 (20.8 dB) from node S, with a passband of about 0.016 Hz to 3-5 Hz.
- Transient, 1% stretch at 15 breaths/min: +/-85 mV at the ADC. At 5% stretch the swing is +/-430 mV with no clipping.
- Cord resistance at 10x or 0.1x nominal: the output stays centred, but the signal is about 3x smaller (+/-29 mV).

The operating point output is in afe_op_results.txt. The plots are afe_bode.png, afe_tran_1pct.png, afe_tran_5pct.png, afe_tran_cord10x.png and afe_tran_cord0p1x.png.