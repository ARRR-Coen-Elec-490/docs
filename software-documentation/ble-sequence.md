# Bluetooth and data-transfer sequences

Proposed design for team review. Orange identifies the MCU, blue identifies Bluetooth, and light blue identifies phone components. Each column is a component, not necessarily a separate device.

## Establish a session

```mermaid
%%{init: {"theme": "base", "themeVariables": {"actorBkg": "#E2EBFA", "actorBorder": "#456A9E", "actorTextColor": "#172C4D", "signalColor": "#244FBD", "signalTextColor": "#172C4D", "noteBkgColor": "#FFF0D5", "noteBorderColor": "#A66012"}}}%%
sequenceDiagram
    accTitle: Establish a Bluetooth session
    accDescr: Connect, check compatibility, send context, and start only after configuration is accepted.
    autonumber
    box rgb(226,235,250) Phone
        participant App as Mobile app
    end
    box rgb(221,231,255) Data transfer
        participant BLE as Bluetooth link
    end
    box rgb(255,229,194) Wearable
        participant MCU as MCU
    end
    App->>BLE: Find device and connect securely
    BLE->>MCU: Establish connection
    App->>MCU: Check supported protocol and settings
    MCU-->>App: Return device capabilities
    App->>MCU: Send v1 session config, duration, and model version
    alt Settings accepted
        MCU-->>App: Confirm configuration
        App->>MCU: Align session timing and request start
        MCU-->>App: Confirm session started
    else Settings rejected
        MCU-->>App: Explain problem, do not start
    end
```

## Encode, transfer, decode, and store

```mermaid
%%{init: {"theme": "base", "themeVariables": {"actorBkg": "#E2EBFA", "actorBorder": "#456A9E", "actorTextColor": "#172C4D", "signalColor": "#244FBD", "signalTextColor": "#172C4D", "noteBkgColor": "#FFF0D5", "noteBorderColor": "#A66012"}}}%%
sequenceDiagram
    accTitle: Encode, transfer, decode, and store a sensor batch
    accDescr: Serialize processed telemetry, split and transport, reassemble and decode, validate and save locally without telemetry acknowledgements.
    autonumber
    box rgb(255,229,194) Wearable
        participant MCU as MCU
    end
    box rgb(221,231,255) Data transfer
        participant BLE as Bluetooth link
    end
    box rgb(226,235,250) Phone
        participant App as App decoder
        participant Store as Local storage
        participant UI as Display and cues
    end
    loop During the active session
        MCU->>MCU: Protobuf encode processed metrics, prediction, and selected telemetry
        MCU->>BLE: Send numbered pieces within Bluetooth size limits
        BLE-->>App: Deliver pieces
        App->>App: Bounded reassembly and Protobuf decode, no compression
        alt Missing or invalid data
            App->>UI: Mark unavailable, suppress affected cues
            Note over MCU,App: Record missing sequences, v1 has no replay or raw samples
        else Valid data
            App->>Store: Save without duplicating previous records
            alt Save succeeds
                Store-->>App: Confirm saved
                App->>UI: Show data, coach only if fresh and safe
            else Save fails
                App->>UI: Report storage fault, send STOP and mark incomplete
            end
        end
    end
```

Protobuf serializes the selected fields into bytes and the phone decodes them. v1 uses no additional compression. Process fault/status messages immediately; recording has no telemetry ACK dependency.

## Disconnection and session completion

```mermaid
%%{init: {"theme": "base", "themeVariables": {"actorBkg": "#E2EBFA", "actorBorder": "#456A9E", "actorTextColor": "#172C4D", "signalColor": "#244FBD", "signalTextColor": "#172C4D", "noteBkgColor": "#FFF0D5", "noteBorderColor": "#A66012"}}}%%
sequenceDiagram
    accTitle: Disconnection and session completion
    accDescr: Suppress stale coaching, reconnect only before the five-second stop deadline, and preserve gaps without replay.
    autonumber
    box rgb(226,235,250) Phone
        participant App as Mobile app
    end
    box rgb(255,229,194) Wearable
        participant MCU as MCU
    end
    alt Bluetooth disconnects
        rect rgb(248,212,212)
            App->>App: Show disconnected, stop stale coaching
            MCU->>MCU: Stop monitored session after 5 seconds disconnected
        end
        opt Reconnection succeeds within 5 seconds
            App->>MCU: Read session state, resume only if still active
            MCU-->>App: Return state and missing-range summary, no replay
            App->>App: Record gaps, require fresh data before adaptive cues
        end
    else Normal session end
        App->>MCU: Request stop
        MCU-->>App: Send final data and completion status
        App->>App: Save and show complete or incomplete summary
    end
    Note over App,MCU: Never restart a stopped session automatically.<br/>Internet is not required for local recording.
```

## Selected transfer contract

Use Protobuf schema v1 with no compression. Raw belt samples never enter telemetry packets. Transfer processed metrics, predictions, selected measurements, and validity/status at 1 Hz. Control commands have explicit result replies; telemetry uses best-effort notifications with gap detection, no replay, and no telemetry-storage ACK.

The [v1 specification](design-spec-v1.md#bluetooth-protocol-v1) fixes the UUIDs, eight-byte fragment header, 512-byte message bound, two-second reassembly timeout, queue limits, and five-second disconnection stop. On lost/invalid data suppress affected cues; on local storage failure stop the session.
