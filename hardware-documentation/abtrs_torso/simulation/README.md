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
- Power-on (V2 and V3 start at 0 V, 60 s): with the cord near 2.2k the signal is usable after ~1 s and centred after ~20 s. With the cord at 10x, the output sits at the 3.3 V rail for ~22 s and is centred after ~50 s. Firmware should ignore the first 30 s.
  To re-run it, set V2 and V3 to Pulse: y1=0, y2=3.3 (V3: 1.65), td=10m, tr=1m (V3: 250m), tf=1m, tw=1000, per=2000.
- ADC load: a 1 MΩ resistor on ADC_OUT models the worst-case nRF52840 ADC input. Gain from node S drops from 20.8 dB to about 20.5 dB (about 3%), with no change to the corners.

The operating point output is in afe_op_results.txt. The plots are afe_bode.png, afe_tran_1pct.png, afe_tran_5pct.png, afe_tran_cord10x.png, afe_tran_cord0p1x.png, afe_tran_poweron.png, afe_tran_poweron_cord10x.png and afe_bode_adcload.png.
