# Software design specification v1

This PR selects a complete v1 engineering design for the MCU, Bluetooth, and ML diagrams. The values below are implementation targets, not measured performance or proof of physiological safety. There are no unselected protocol or model options in v1. A failed release check blocks deployment instead of silently weakening the design.

## Source baseline and deliberate changes

Rebased onto docs main `352185099aa9f6ac422604072e8aa86de70245dd`. Reviewed [system-operation.md](../system-architecture/system-operation.md), its input/output/context diagrams, the three overarching flowcharts, and the updated embedded-device/mobile-app sequence image. These establish MCU feature calculation and inference, a three-invalid-check counter, app-local storage, and explicit start/stop messages.

The latest user clarification takes precedence over the overview's raw-window transmission: **raw belt samples stay on the MCU; normal packets contain calculated metrics, predictions, selected sensor measurements, and status**. Raw-sample streaming and debug capture are excluded from v1, not implicit optional packet fields. Sensor acquisition, analysis-window length, and Bluetooth update rate are separate quantities.

The overview describes automatic box breathing as a proven recovery fallback. This specification does not make that safety claim. After sensor loss, stop adaptive coaching and offer a user-selected, explicitly unmonitored 4–4–4–4 guide. A critical device fault or a user reporting discomfort ends all guidance. The existing PNGs remain historical references; the Mermaid diagrams and this specification define this PR's behavior.

## Session and acquisition

| Item | Selected v1 behavior |
| --- | --- |
| Session | Seated post-exercise recovery, maximum 300 seconds; user can stop at any time. A resting-only session is recorded as such and never reports HRR without an exercise-end baseline |
| Startup | Three initialization attempts, each with a one-second deadline; missing required sensors or a hardware fault prevents START |
| Required channels | Chest, abdomen, HR, SpO2, body-temperature sensor; battery/device fault status is also required |
| Belt acquisition | Paired 50 Hz samples with monotonic millisecond timestamps; 1,000 paired samples in a 20-second ring buffer |
| Belt calibration | Ten seconds of comfortable unpaced breathing before recovery timing begins; subtract per-channel median, divide by channel 90th–10th percentile amplitude. Zero span or sensor-reported saturation fails calibration. Calibration is fixed for that session |
| Digital readings | Request HR/SpO2 at 1 Hz and temperature/battery at 1 Hz; retain acquisition time and module validity rather than pretending each poll is a new measurement |
| Freshness | HR/SpO2 no older than two seconds; body/device temperature and battery status no older than five seconds |
| Filtering | Causal five-sample moving average on each normalized belt channel; same arithmetic, calibration, and feature ordering in Python and C/C++ |
| Analysis | Trailing 20-second window, evaluated every second; discard incomplete or invalid cycles and require at least one complete accepted cycle |
| Window quality | At least 95% valid paired samples and no gap longer than 200 ms. Never bridge a gap to fabricate a breathing cycle |
| UI / transmission | One processed telemetry packet per second; safety/status changes are sent immediately. No waveform samples leave the MCU |
| Timing budget | Completed feature window to phone cue within one second; no claim that a 20-second window detects a physical change within one second. The report's ten-second physical-change target is a separate acceptance test and must not be claimed from this budget alone |

These rates are requested rates, not claims that every selected sensor supports them. A sensor/firmware combination that cannot satisfy them fails the integration gate. Body temperature and device temperature are different fields. Do not substitute ambient temperature for body temperature.

## Metrics and recovery timing

- **Breathing pace:** find peak/trough pairs in the filtered abdomen signal, using calibrated chest only when abdomen is invalid and the session is already in nonadaptive fault handling. Normal inference requires both belts. Use peak-to-peak complete periods of 2–20 seconds and a peak-to-trough amplitude of at least 0.1 normalized units. Merge peaks less than one second apart by retaining the higher peak. Pace is `60 / median(accepted_periods_seconds)` over the analysis window. These are signal-processing bounds, not medical normal ranges.
- **TA ratio:** use the median peak-to-trough normalized chest amplitude divided by the median synchronized abdomen amplitude over accepted cycles. Denominator below 0.05 normalized units makes the feature invalid. This is a session-normalized movement ratio, not a volume measurement or an anatomical-posture measurement. Store calibration version with the session.
- **HRR baseline:** user marks exercise end; use median valid HR in the preceding five seconds, with at least three readings. Missing baseline leaves all HRR unavailable and does not fabricate an exercise-end value.
- **HRR1/2/3:** baseline minus HR at 60/120/180 seconds. Use an observed reading at the horizon, otherwise interpolate only between valid observations bracketing it, each within two seconds. Publish unavailable until bracketing data exists. Missing horizons remain unavailable; never extrapolate future HRR or substitute zero.
- **Profile context:** app retains age, biological sex, height, weight, and optional respiratory-condition information. Send only session mode and reference version to MCU v1. Demographics remain report context, not classifier inputs. No demographic expected-value lookup table or medical reference range is used in v1 because the uploaded sources supply no validated table. This explicitly removes the earlier proposed lookup dependency.

## Fault state machine

| State / event | Required behavior |
| --- | --- |
| READY | All startup checks pass; await acknowledged START |
| WARMUP | Acquire a full valid analysis window. Missing early HRR is normal, not a sensor fault |
| ADAPTIVE | Valid features and released model available; app may present the reviewed pattern label |
| First invalid health check | Immediately invalidate affected metrics/prediction and suppress adaptive cues; keep valid unrelated telemetry |
| Three consecutive invalid one-second health checks | Enter SENSOR_FAULT, notify once on entry, and record a gap. Continue checking sensors without blocking other work |
| SENSOR_FAULT | Start a 30-second recovery deadline. Phone can offer an explicit unmonitored guide; it cannot claim restored monitoring |
| Sensor recovery | Three consecutive valid health checks followed by a fresh full 20-second valid window; show “monitoring restored.” Resume adaptive mode only after user confirmation |
| Recovery deadline expires | End the monitored session, retain valid records, and label the result interrupted. End any fallback guide |
| BLE lost | Phone suppresses adaptive cues immediately; device stops the session after five seconds disconnected. No implicit restart on reconnect |
| Storage failure | App suppresses guidance, sends STOP, and marks the session incomplete |
| Critical device fault / STOP / 300-second expiry | Stop immediately, cancel all cues, send final status if possible, and release buffers into standby. Never wait for a packet ACK to stop |

Health checks use sensor-response errors, missing/stale samples, saturation/status flags, and the window quality checks above. A flat physiological signal alone is not proof of broken hardware. Battery/thermal hard-stop conditions use the selected hardware's documented fault/status outputs; no fabricated HR, SpO2, or body-temperature medical alarm thresholds are introduced. In v1 these physiological readings are shown with validity, **not claimed as a validated clinical alarm system**. If hardware fault status is unavailable or unsupported, startup fails rather than guessing a safe operating limit.

### Unmonitored box-breathing guide

The phone implements a fixed timer: inhale four seconds, hold four, exhale four, hold four. It uses no model and no missing sensor values. It is offered only on SENSOR_FAULT after clear user acceptance, for at most the remaining 30-second fault-recovery period, and never after a critical device fault. Always show “Sensor monitoring unavailable,” keep Stop visible, and stop immediately if the user reports discomfort. It is not a safety guarantee or a treatment for breathlessness. NHS guidance says not to force breathing/holds and to stop for dizziness or discomfort: [NHS breathing guidance](https://asph.nhs.uk/nervous-system-regulation). This design selects opt-in fallback rather than automatic breathing holds.

## MCU output and phone responsibilities

One telemetry record contains session ID, sequence, monotonic sample/window timestamps, pace, TA ratio, available HRR1/2/3 and provenance, latest HR/SpO2/body temperature, battery/device status, model version, class/abstention status, and per-field validity. It contains no belt samples, names, email, free-text conditions, or credentials.

The phone validates identity, ordering, timing, and field validity; stores records locally in SQLite using `(device_boot_id, session_id, sequence)` as the duplicate key; and displays current measurements and a categorical pattern. v1 does not automatically prescribe a new physiological breathing target from a class label. Backend synchronization is a separate retryable post-session process and never blocks local capture. On storage failure, stop the monitored session.

## Bluetooth protocol v1

| Item | Selected behavior |
| --- | --- |
| Roles | Wearable peripheral/GATT server; phone central/GATT client; require encrypted bonded connection before accepting context or START |
| Service | `49000001-23ab-4cde-9012-490000000001` |
| Control | `49000002-23ab-4cde-9012-490000000001`, write with response; CONFIG, START, STOP, RESUME, STATUS commands |
| Telemetry | `49000003-23ab-4cde-9012-490000000001`, notifications at 1 Hz |
| Status | `49000004-23ab-4cde-9012-490000000001`, notifications for command results, faults, heartbeat and SESSION_END |
| Serialization | Shared Protobuf schema version 1; explicit field presence/validity; SI units except HR in beats/min, SpO2 in percent, pace in breaths/min |
| Compression | None in v1. Protobuf serialization is the only encoding; no compression/decompression stage is claimed |
| Message bound | Maximum 512 encoded bytes for any message; reject oversize before transmission/allocation |
| Fragment header | Eight bytes: version u8, kind u8, sequence u16 little-endian, fragment index u8, fragment count u8, total encoded length u16 little-endian. Kinds: control=1, telemetry=2, status=3 |
| Fragment payload | `min(actual negotiated ATT_MTU - 3, platform notification limit) - 8` bytes. MTU 23 therefore permits 12 payload bytes per fragment; no hard-coded larger MTU assumption |
| Reassembly | Maximum two in-progress 512-byte assemblies per direction; two-second deadline; 43-fragment cap. Reject inconsistent sizes, out-of-range indexes, wrong session or version |
| Session identity | Device advertises boot ID in STATUS; CONFIG establishes session ID. Identity is also in the decoded envelope. Drop partial assemblies on reconnect/session change |
| Sequence | Independent u16 counters per message kind, reset only on a new acknowledged session; 300-second maximum prevents normal 1 Hz telemetry rollover. End with protocol error rather than silently wrap |
| Commands | u32 command ID inside Protobuf envelope; cache last eight results. Repeated ID returns prior result without repeating START or resetting HRR. Two-second reply timeout, at most three sends total |
| Queue | Four telemetry messages maximum; keep latest when full and expose dropped sequence ranges. Two reserved status messages, with faults/STOP ahead of telemetry |
| Delivery | Notifications are best effort. Record sequence gaps and duplicates; **no telemetry ACK, replay, or raw-data backlog in v1**. Command result acknowledgement is separate from a BLE write response |
| Heartbeat | Status every second; phone suppresses adaptive cues after two seconds without valid telemetry/status. MCU five-second disconnect stop applies independently |
| Reconnect | Read STATUS. A stopped session stays stopped; new monitoring requires a new session. Reconnection before the deadline may continue the same active session, but missing history remains missing |
| Completion | SESSION_END includes final telemetry sequence, reason, and dropped-range summary. App marks complete only when observed coverage agrees; a lost final status yields incomplete, not invented completion |

CONFIG includes session ID/mode, acquisition and feature schema versions, required model version, and duration. MCU accepts only the v1 values in this document. Status contains acceptance/rejection reason, state, boot/session identity, configuration/model versions, and validity/fault flags. These field meanings are fixed here; a generated `.proto` implementation is separate work and must preserve this contract.

## ML specification v1

| Item | Selected design |
| --- | --- |
| Task | Supervised classification of thoracic-dominant, abdominal-dominant, or mixed breathing; not anatomical posture or a medical diagnosis |
| Ground truth | Seated guided trials for each instructed pattern, annotated from synchronized reference observation by two reviewers. Keep only consensus intervals, exclude the first/last five seconds of a transition and any uncertain interval. Instructions alone do not prove the participant followed the pattern |
| Features | Ordered vector `[breathing_pace_bpm, session_normalized_TA_ratio]`. HRR and demographics are calculated/report context, not v1 classifier inputs; no early-session imputation problem |
| Normalization | StandardScaler fitted only on training folds; export its mean/scale with the model. Reject invalid features, do not fill with zero |
| Development split | Deterministic seed 490; hold out 20% of participants, at least two, before windowing. Require at least ten participants total and every class represented in development and held-out sets; otherwise block model release |
| Validation | Five participant-grouped folds inside development data; keep all sessions/windows of each participant together. Fit learned preprocessing only on training folds |
| Candidates | Majority baseline; linear SVM with C in {0.1, 1, 10}; logistic regression with C in {0.1, 1, 10}; Random Forest with 10 or 25 trees and maximum depth 3 or 5, random seed 490 |
| Selection | Best mean validation macro-F1 among models meeting resource gates; within 0.01 choose linear SVM, then logistic regression, then smaller forest. Baseline is a comparison, not an embedded coaching model |
| Numeric regression | Not part of v1: no independent numeric prediction target exists. Linear regression/SVR are excluded rather than compared incorrectly to classifiers |
| Release accuracy gate | Held-out macro-F1 at least 0.80, recall at least 0.70 for every class, and at least 0.05 macro-F1 above majority baseline. These are engineering acceptance targets, not clinical validation |
| Embedded gate | Frozen preprocessing + model exported to C/C++; no prediction mismatch on at least 1,000 held-out/edge-case vectors, maximum inference 100 ms, incremental peak RAM at most 64 KiB, model/preprocessing flash at most 256 KiB on target board |
| Export | Use m2cgen only for a supported winning estimator; implement scaler explicitly in C/C++. Unsupported export or parity failure blocks deployment, not an unreviewed runtime replacement |
| Inference output | Class enum plus feature/window/model versions and time; no confidence value because v1 does not calibrate probabilities. Invalid/OOD inputs return ABSTAIN |
| Out-of-domain check | Reject features outside per-feature development-training min/max bounds; final bounds are packaged with the frozen model. Invalid bounds/model artifacts cause startup model-unavailable state |
| Missing model / failed release | Collect valid selected telemetry without adaptive pattern coaching; do not deploy an unvalidated classifier or invent labels |
| Updates | No on-device training, automatic per-session updates, Azure dependency, or RL in v1. New artifact requires the same validation and version change |

Do not tune against the held-out test set. A failed test or a changed candidate after test inspection requires new untouched evaluation participants. Grouped splitting and fold-local scaling follow [scikit-learn guidance](https://scikit-learn.org/stable/common_pitfalls.html#data-leakage).

## Acceptance evidence

Before firmware/app release, demonstrate startup faults, read deadlines, all three invalid checks, 30-second fault expiry, opt-in guide and Stop, three-valid-check recovery plus full-window/user confirmation, five-second disconnection stop, HRR availability, Protobuf fixtures on both platforms, MTU-23 fragmentation, malformed fragments, two-second reassembly timeout, full queues, missing/duplicate notifications, storage failure, and incomplete final coverage. Measure actual MCU memory/latency and full sensor-to-phone timing. A documentation PR does not satisfy these implementation tests.
