# Bluetooth implementation notes

Status: proposed wire behavior, not an implemented protocol. This expands [white-box section 3.3](../../system-architecture/white-box.md#33-data-transfer-layer). UUIDs, schema fields, lengths, codecs, retention, acknowledgements, and timeouts remain team decisions; see [shared decisions](../README.md).

## Establish a session

Commands need bounded IDs and idempotent handling: retrying a lost START response must not start a second session or reset the recovery baseline. A GATT write response confirms the write operation, not successful decoding, model compatibility, or durable storage; application-level status messages confirm those outcomes. Status/control messages also require framing if larger than the effective payload.

## Stream and consume a batch

ACK/replay is a proposed application feature, not something BLE notifications supply. Maintain a highest **contiguous** persisted sequence plus known gap ranges; acknowledging a higher received packet alone must not erase earlier loss. Process safety events immediately when received while keeping persisted progress honest. Replay data may fill history but must not trigger historical coaching cues.

## Disconnect, replay, and completion

## Encoding and transport contract

The order is **construct message → Protobuf serialize → optional compress → frame/fragment → BLE → reassemble → optional decompress → Protobuf decode → validate → persist**. [Protocol Buffers](https://protobuf.dev/overview/) defines structured serialization; it is not a general-purpose compression codec. Start without additional compression, then benchmark actual bytes, latency, MCU memory, battery, and phone throughput. Small batches can grow when compressed.

| Topic | Proposed agreement between components |
| --- | --- |
| Roles | Phone is BLE central/GATT client; wearable is peripheral/GATT server |
| Channels | Control/configuration writes, status responses, and telemetry notifications; exact characteristic layout is TBD |
| Schema | One reviewed `.proto` source; generated or compatible bounded MCU codec and TypeScript codec; explicit enum values, optional-field presence, units, scales, and compatibility rules |
| Identity | Device/boot/session IDs plus monotonically increasing message sequence; define rollover behavior |
| Time | Monotonic sample/window times, sample interval or per-sample offsets, and a session clock mapping with uncertainty |
| Payload bound | Size fragments for actual connection capacity; a standard single ATT notification carries at most `ATT_MTU - 3` bytes, before subtracting application framing |
| Frame metadata | Protocol version, message kind, identity, sequence, fragment index/count, codec, encoded/decoded lengths; exact repeated/header layout TBD |
| Defensive decoding | Cap fragment count, message size, expanded size, active assemblies, and assembly lifetime; reject unsupported codecs before allocation |
| Queue behavior | Bounded sender/replay queues; prioritize safety and control; explicit dropped ranges and timeout policy |
| Reliability | BLE link retransmission is not proof of app receipt or storage; application sequence tracking, deduplication, and bounded replay address that gap |
| Completion | Final sequence and loss summary; incomplete sessions remain visibly incomplete |

The notification limit is described in the [Bluetooth SIG Device Time Service specification](https://files.bluetooth.com/download/dts_v1-0-pdf/); effective limits still need validation on the chosen phone/library/firmware combination. MTU negotiation differs across platforms. BLE link integrity checks exist; a custom application checksum is not automatically necessary and is not a replacement for authentication or encryption. Decide on any end-to-end checksum from the framing/reassembly threat and error model.

BLE throughput does not require a blanket assumption that uncompressed samples will crash the app. Measure sample rate × channels × encoded bytes, metadata overhead, batching delay, queue growth, and storage/UI load. Local storage and display refresh can use different rates without discarding original sample timestamps.

## Review checks

- Use shared binary fixtures to verify both codecs preserve optional/missing values, enums, units, and all supported schema versions.
- Test one/many fragments, repeated fragments, missing/reordered batches, unknown codec, decode failure, and expanded-size limits.
- Test disconnect during context setup, streaming, storage commit, and final ACK; inject device reboot and full queues.
- Confirm replay is idempotent, gaps are not silently erased, stale results cannot trigger cues, and local capture works without internet.
