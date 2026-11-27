# MCU implementation notes

Status: proposed design. See [shared assumptions and open decisions](../README.md). This expands the input handling, preprocessing, embedded ML, and output handling in [white-box section 3.2](../../system-architecture/white-box.md#32-embedded-mcu-layer).

## Session processing

The branches after input-quality assessment describe logically independent work, not necessarily multiple processor threads. Safety checks must continue even while a feature window is warming up or inference is unavailable. The record builder joins current statuses at a batch deadline; it does not wait indefinitely for an inference result or BLE delivery. An older prediction must be marked with its original window time and must not be presented as a fresh result.

The exact scheduler, ring-buffer sizes, retry retention, and disconnect grace period are open. A critical fault takes the safe-stop path immediately; flushing pending output is best effort and cannot delay stopping. The phone independently times out stale packets and suppresses cues if the BLE link is lost.

The additionally supplied legacy MCU/phone flowchart reinforces sensor self-test, battery checks, timestamping, invalid-packet rejection, and local storage. Implement sensor initialization with bounded retries followed by an exposed fault. Acquisition uses ADC conversion for analog belt inputs and the appropriate bus/interface for digital sensors; not every measurement passes through the MCU ADC. Timestamp at acquisition rather than after potentially delayed downstream processing.

That flowchart places ratio/HRV calculation on the phone, after MCU classification. For the current model, required features must exist before inference, so metric calculation belongs before the MCU classifier. Phone-side recomputation can support verification but must use the same definitions and timestamps. HRV is not HRR: it is a separate proposed metric requiring appropriate beat-to-beat interval measurements, not merely a stream of averaged heart-rate values. It remains outside the initial three-metric scope unless explicitly approved.

## Metric definitions and validity

| Metric | Proposed calculation | Required validity and context |
| --- | --- | --- |
| Breathing pace, breaths/min | Detect complete cycles using peaks and valleys in filtered belt signals; for accepted cycle periods, use `60 / mean(period_seconds)` | Enough complete cycles; reject movement artifacts and implausible detections; define which belt or combined signal supplies events |
| Thoracic-to-abdominal ratio, TA | Window aggregate of calibrated chest excursion amplitude divided by calibrated abdomen excursion amplitude | Match the two signals in time and units; define aggregation; near-zero abdomen amplitude yields invalid, not infinity |
| HRR at horizon `h` | `HR_at_exercise_end - HR_at_recovery_elapsed_h`, for `h = 60, 120, 180` seconds | Valid recovery-start marker and baseline; adequate quality near each horizon; not available for a resting-only session without an exercise-end reference |
| Profile-relative reference | Compare a measured metric with an approved lookup entry for the supported profile and session protocol | Record reference-table version; missing profile or unmatched table entry gives unavailable context |

TA ratio is not the same as `chest / (chest + abdomen)` or a clinical volume ratio. Choose and validate the convention before training; the same feature implementation must be used in Python and firmware. Raw piezo amplitudes do not automatically represent absolute tidal volume or anatomical posture.

HRR is a timed recovery measurement, not simply the difference between the beginning and end of every sliding window. Before 60 seconds, HRR1 is unavailable; before 120/180 seconds, HRR2/3 are unavailable. Interpolation can estimate HR at a horizon only after valid observations exist on both sides of that horizon, within an agreed maximum gap. Never extrapolate a future HRR value and label it observed. Preserve `observed`, `interpolated`, or `unavailable` for each horizon. A separate elapsed-recovery measurement may be defined, but it must not be named HRR1/2/3.

Missing features must not become numeric zero. Early-session operation can use a separately trained and validated feature subset, such as a no-HRR model; otherwise it abstains. Record which model variant was used.

## Inputs and output record

| Group | Fields or meaning |
| --- | --- |
| Biological inputs | Synchronized chest and abdomen readings; HR in beats/min; SpO2 in percent; body-temperature reading with sensor site and units defined |
| Device inputs | Battery state, device temperature, sensor health, stop command, and link state; device temperature is distinct from body temperature |
| Session configuration | Opaque session ID; session mode; versioned sample/window configuration; supported profile context; model and reference-table compatibility |
| Identity and timing | Device/boot ID, session ID, sequence number, monotonic sample/window times, configuration and feature-schema versions |
| Measurements | Raw or calibrated sample batch with per-channel timestamps, units/scales, validity, and missing-sample information |
| Features | Window boundaries, pace, TA ratio, HRR horizons and their provenance, feature-validity mask |
| Decision | Reviewed pattern label or abstention; model version; prediction time; optional calibrated confidence only if supported |
| Safety/status | Active warnings/faults, quality reasons, dropped ranges, and completion status |

The record is a semantic contract, not a finalized `.proto` schema. Names, numeric encodings, limits, and field numbers need joint approval with the app team. Only the profile fields actually needed by the chosen model/reference lookup should be sent to the device; names and emails are not inference inputs.

## Review checks

- Test warmup, missing channels, flat belts, zero abdomen amplitude, out-of-order timestamps, and low-quality HR near an HRR horizon.
- Verify an invalid sample neither clears a safety warning nor produces a valid classifier output.
- Verify safety and acquisition continue under output backpressure, disconnects, and slow inference.
- Measure acquisition-to-cue timing and validate reference feature vectors against the Python implementation.
