# Software layer diagrams

This branch defines a selected v1 design for review. It specifies intended behavior; firmware, training, and app integration are separate implementation work.

| Document | Contents |
| --- | --- |
| [MCU flowchart](mcu-flow.md) | Sensor acquisition, three-check fault handling, metrics, embedded ML, and processed output |
| [Bluetooth sequences](ble-sequence.md) | Connection, v1 Protobuf transfer, invalid data, and session completion |
| [ML flowcharts](ml-flow.md) | Participant-separated training, classifier comparison, embedded release, and runtime handling |
| [Design specification v1](design-spec-v1.md) | Selected rates, windows, fault deadlines, fallback policy, packet contract, model inputs, and release criteria |

Orange identifies MCU work, green ML, blue Bluetooth, light blue phone components, and red fault handling. Johnny owns app UI/UX and prototypes; these documents specify the interfaces and behavior that those designs must support.

## Selected scope

- Calculate features and run the released model on the MCU.
- Keep raw belt samples on the MCU. Send processed metrics, predictions, selected HR/SpO2/temperature measurements, timestamps, and status once per second.
- Use three consecutive invalid health checks to enter sensor-fault handling, with a 30-second recovery deadline and an explicit stop path.
- Offer the fixed box-breathing timer only after user acceptance, labelled unmonitored. Critical faults stop all cues; box breathing is not a safety guarantee.
- Use Protobuf with no additional compression, bounded fragmentation, and explicit gaps; v1 does not implement telemetry replay.
- Train a breathing-pattern classifier from pace and normalized TA ratio. HRR and demographics are report context. No numerical regression, demographic lookup table, RL, or Azure dependency is included in v1.

There is no outstanding options list: the [v1 specification](design-spec-v1.md) selects the defaults and defines what blocks release. Those choices are proposals in this PR, not claims of team approval or measured clinical safety.

## Sources and reconciliation

Rebased onto docs main `3521850` and reviewed its system-operation flowcharts and updated sequence image. See the [source baseline and deliberate changes](design-spec-v1.md#source-baseline-and-deliberate-changes) for differences from the older raw-data and automatic-fallback descriptions. The original downloaded Teams archive is a historical snapshot, not a live checkout.

## Preview

Open a Markdown document and use **Markdown: Open Preview to the Side** (`Ctrl+K V`). The Mermaid blocks are the editable source. PNG snapshots for reports are below; regenerate them when source changes.

| Diagram | Preview |
| --- | --- |
| MCU processing | [PNG](previews/mcu-flow-1.png) |
| BLE setup | [PNG](previews/ble-sequence-1.png) |
| BLE streaming | [PNG](previews/ble-sequence-2.png) |
| BLE recovery and completion | [PNG](previews/ble-sequence-3.png) |
| ML development | [PNG](previews/ml-flow-1.png) |
| ML inference | [PNG](previews/ml-flow-2.png) |

Render extracted Mermaid with Mermaid CLI 11.12.0 (`mmdc -i diagram.mmd -o preview.png -b white -w 1800`). Markdown checks use the repository configuration. Syntax/layout checks do not replace the specification's implementation acceptance tests.
