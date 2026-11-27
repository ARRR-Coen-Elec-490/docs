# Software layer diagrams

Status: proposed design for team review, 2026-11-26. These diagrams specify intended behavior; they do not claim that firmware, ML models, or the mobile application are implemented.

| Document | Purpose |
| --- | --- |
| [MCU flowchart](mcu-flow.md) | Input handling, signal quality, metric calculation, independent safety checks, embedded inference, and output handling |
| [BLE sequence diagram](ble-sequence.md) | Session setup, Protobuf encoding, optional compression, fragmentation, decoding, storage, and recovery |
| [ML flowcharts](ml-flow.md) | Dataset and label design, model comparison, embedded deployment, and runtime decisions |

Johnny owns application UI/UX, wireframes, and the detailed app flow. These documents define only the app interfaces required by the three assigned layers.

## Presenting versus implementing

The three linked documents contain simplified meeting diagrams and brief explanations. Detailed formulas, data-field tables, protocol agreements, and review checks are preserved in [MCU implementation notes](implementation-notes/mcu-flow.md), [Bluetooth implementation notes](implementation-notes/ble-sequence.md), and [ML implementation notes](implementation-notes/ml-flow.md). A proposed interface contract means the rules two components must agree on; it is not a claim of implementation or a requirement already approved by the team.

Colours follow the architecture layers: orange for MCU work, green for ML, blue for Bluetooth, light blue for phone components, and red for safety/fault handling. Sequence diagrams group participants by layer. The labels and arrows also identify roles so colour is not the only way to interpret a diagram.

## Sources and precedence

The baseline is the documentation repository's `main` commit [`ec19291574ed4bffecdfd81b28a612bccba8a830`](https://github.com/ARRR-Coen-Elec-490/docs/tree/ec19291574ed4bffecdfd81b28a612bccba8a830), especially [white-box.md](../system-architecture/white-box.md) and the MCU, data-transfer, ML, and overall PNG diagrams linked there. The diagrams were visually inspected as well as reading their descriptions.

The supplied Discord notes dated November 24 and the following discussion refine that baseline: calculate thoracic-to-abdominal ratio, breathing pace, and HRR; use profile data as context; monitor raw HR, SpO2, and body temperature independently; compare simple supervised models first. The downloaded Phase II Word documents provide additional historical context, particularly *Design Tools & Technology*, *Preliminary Design Information*, and *Machine Learning Models*. They are not assumed to reflect the latest decisions.

An additional user-supplied flowchart shows sensor initialization, a battery gate, ADC conversion/timestamping, MCU classification/advice, wireless transmission, and phone decoding, ratio/HRV calculation, storage, and display. Its provenance/date are unknown. It was compared visually with these drafts; it has not been identified among the current architecture PNGs, and embedded images in the downloaded Office files have not been exhaustively matched. Treat it as an additional design reference rather than a confirmed current specification.

| Existing design or note | Treatment in these diagrams |
| --- | --- |
| Current white box places the model on the MCU | Preserve embedded inference; train offline in Python and validate the exported C/C++ artifact |
| Earlier Word document proposes a second RL model and Azure | Leave outside the initial supervised implementation; require a separate state/action/reward proposal and validation plan |
| Existing MCU picture passes raw inputs, metrics, and labels to output handling | Preserve all three paths and add timestamps, validity, model version, and bounded queues |
| Data transfer picture shows bidirectional user/session data | Expand into setup, context acknowledgement, streaming, error handling, and session completion |
| Existing prose calls Protobuf compression and predicts uncompressed app failure | Treat Protobuf as binary serialization; compression and throughput need measurement, not assumptions |
| Notes suggest HRR interpolation when insufficient time has passed | Report unavailable future horizons; interpolation is permitted only between observed measurements bracketing the requested elapsed time |
| Notes include possible breathing disabilities | Keep optional relevant conditions in app context pending a reviewed encoding, appropriate reference data, and a supported-use decision |
| Notes mention sedentary breathing; current white box says post-exercise recovery | Support seated measurement; distinguish resting sessions from post-exercise recovery, where HRR has a defined baseline |
| Current ML prose says breathing pace and posture | Pace is calculated directly; classify a reviewed breathing-pattern target. Anatomical posture is not assumed observable from these sensors |
| Additional flowchart calculates ratio/HRV on the phone after MCU classification | For the present feature-based MCU model, calculate required metrics before inference on the MCU; the phone may independently verify/recompute them from retained samples |
| Additional flowchart mentions HRV | HRV and HRR are different metrics. HRV is not added to the agreed initial feature set without a separate decision about beat-to-beat interval data and validation |
| Additional flowchart sends low-battery alert then ends | Preserve alert and safe-stop intent; stopping must not wait for wireless delivery, and critical runtime faults use the same priority path |
| Additional flowchart retries initialization indefinitely | Bound initialization retries and expose a persistent fault if recovery fails; do not hang indefinitely waiting for a sensor |
| Additional flowchart treats all inputs as analog voltages | Apply ADC conversion to analog belt channels; acquire digital sensors through their actual interfaces and retain per-channel timing |

## Shared boundaries

- MCU: acquire signals, calculate features, evaluate signal quality and deterministic safety rules, run a validated embedded model, and package results.
- BLE: move versioned messages between MCU and phone. Neither Protobuf nor BLE delivers application persistence guarantees by itself.
- Phone: supply session/profile context, persist validated messages locally, and present cues only when safety, freshness, and quality permit. Synchronization to a backend is separate from the BLE session.
- ML development: define ground truth, train and compare models, evaluate generalization, export the entire preprocessing/inference contract, and measure performance on the target MCU.
- Safety: deterministic checks operate independently of the ML prediction. A classifier result cannot clear an active fault or authorize unsupported training intensity.

Raw signal availability and sampling rate are distinct from BLE batch transmission rate. The historical 3–5 transfers per second is a provisional batching idea, not a sensor sampling specification. Preserve relative sample timestamps within each batch.

## Decisions needed before implementation

| Decision | Proposed owner | Why it matters |
| --- | --- | --- |
| Seated resting versus post-exercise session protocol | Whole team | Determines whether HRR is available and which references apply |
| Sampling rates, window duration/stride, filters, and belt calibration | MCU and ML | Defines feature meaning, latency, bandwidth, and ground truth alignment |
| Exact TA ratio and breathing event algorithm | MCU and ML | Both belt signals need calibrated, synchronized amplitudes and reliable peak/valley detection |
| HRR baseline and observed/interpolated rules | MCU and ML | Prevents fabricated future values and inconsistent recovery targets |
| Label vocabulary, annotation protocol, and dataset coverage | ML and domain reviewers | Supervised training requires a defensible target, not just feature columns |
| Profile fields, lookup-table provenance, and unsupported contexts | App, ML, and domain reviewers | Missing or unmatched context must not create invented expected values |
| Safety limits, persistence/hysteresis, invalid-sensor and stop policies | Hardware, MCU, app, and domain reviewers | No medical thresholds are invented in this proposal |
| UUIDs, `.proto` schema, codec, MTU strategy, maximum message size, queue bounds, and disconnect timeout | MCU and app | Both ends must implement the same wire protocol and failure behavior |
| MCU memory, inference latency, battery, and end-to-end timing budgets | MCU and ML | Determines whether the chosen model can actually be deployed |
| Whether any separate RL/cloud feature remains in scope | Whole team | Current main and older Word documents disagree |

The Phase II report's feedback target is within 10 seconds. Agree whether that is measured from a physical change or a completed feature window; account for acquisition windowing, processing, BLE, and app presentation together. No sampling rates, warning limits, labels, or byte limits below are approved specifications.

## Preview and editing

Mermaid is a text-based diagram language. The fenced `mermaid` blocks in these Markdown files are the editable source and render on GitHub. They can be reviewed without image-editing software.

Open a document in VS Code and choose **Markdown: Open Preview to the Side** (`Ctrl+K V`). VS Code 1.121 and later include Mermaid Markdown preview support. For an older installation, use [Markdown Preview Mermaid Support](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid), extension ID `bierner.markdown-mermaid`. Avoid installing competing preview extensions unless necessary. See the [VS Code release notes](https://code.visualstudio.com/updates/v1_121/) and [Mermaid syntax reference](https://mermaid.js.org/intro/syntax-reference).

These documents supplement the existing white-box overview rather than replacing its images. The app repository was separately rebased onto `main` at `25ef29d`; updating it does not update the downloaded Teams archive. This checkout of `docs` is a separate Git repository with its own branch and eventual review.

## Rendered previews and validation

The Markdown Mermaid blocks are authoritative. PNG snapshots are included for quick viewing and report drafts; regenerate them whenever a diagram changes.

| Diagram | Preview |
| --- | --- |
| MCU processing | [PNG](previews/mcu-flow-1.png) |
| BLE setup | [PNG](previews/ble-sequence-1.png) |
| BLE streaming | [PNG](previews/ble-sequence-2.png) |
| BLE recovery and completion | [PNG](previews/ble-sequence-3.png) |
| ML development | [PNG](previews/ml-flow-1.png) |
| ML inference | [PNG](previews/ml-flow-2.png) |

All six diagrams were rendered with `@mermaid-js/mermaid-cli` 11.12.0 and visually reviewed. Markdown was checked with `markdownlint-cli2` 0.18.1 against the repository configuration, and local link destinations were checked. Rendering validates syntax and layout, not physiological assumptions or firmware behavior.

To reproduce a preview, copy a diagram's fenced Mermaid contents into a temporary `.mmd` file, then run `mmdc -i diagram.mmd -o preview.png -b white -w 1800` using Mermaid CLI and its supported browser runtime. The validation tooling was installed outside the repositories; no app dependencies were added.
