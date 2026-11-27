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
    App->>MCU: Send encoded session settings and user context
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
    accDescr: Serialize, optionally compress, split and transport, reassemble and decode, validate, and save before acknowledging storage.
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
        MCU->>MCU: Protobuf encode readings, metrics, prediction, and status
        opt Compression agreed and shown to help
            MCU->>MCU: Compress encoded bytes
        end
        MCU->>BLE: Send numbered pieces within Bluetooth size limits
        BLE-->>App: Deliver pieces
        App->>App: Reassemble, decompress if used, and Protobuf decode
        alt Missing or invalid data
            App->>UI: Mark unavailable, suppress affected cues
            Note over MCU,App: Record gaps, request replay only if supported
        else Valid data
            App->>Store: Save without duplicating previous records
            alt Save succeeds
                Store-->>App: Confirm saved
                App-->>MCU: Acknowledge saved progress
                App->>UI: Show data, coach only if fresh and safe
            else Save fails
                App->>UI: Report storage fault, apply pause/stop policy
            end
        end
    end
```

Protobuf converts structured data into bytes; it is not a compression codec. The phone reverses the same steps in the opposite order. Safety alerts are handled promptly when received without waiting for normal storage acknowledgements.

## Disconnection and session completion

```mermaid
%%{init: {"theme": "base", "themeVariables": {"actorBkg": "#E2EBFA", "actorBorder": "#456A9E", "actorTextColor": "#172C4D", "signalColor": "#244FBD", "signalTextColor": "#172C4D", "noteBkgColor": "#FFF0D5", "noteBorderColor": "#A66012"}}}%%
sequenceDiagram
    accTitle: Disconnection and session completion
    accDescr: Suppress stale coaching, recover retained data within agreed limits, and show missing data in the final summary.
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
            MCU->>MCU: Keep limited data, stop at disconnect deadline
        end
        opt Reconnection succeeds within agreed policy
            App->>MCU: Check session state, request retained missing data
            MCU-->>App: Replay available records, report gaps
            App->>App: Save replay, never use old data for live cues
        end
    else Normal session end
        App->>MCU: Request stop
        MCU-->>App: Send final data and completion status
        App->>App: Save and show complete or incomplete summary
    end
    Note over App,MCU: Never restart a stopped session automatically.<br/>Internet is not required for local recording.
```

## What must both ends agree on?

These are proposed interface agreements, sometimes called a **contract**. They ensure the MCU and phone interpret the same bytes the same way. They do not mean these features are implemented.

| Topic | Proposed agreement |
| --- | --- |
| Packet contents | Same Protobuf schema, field meanings, and units |
| Timing and identity | Identify the session, message order, and measurement time |
| Message size | Split large messages and limit memory used for reassembly |
| Missing data | Detect gaps/duplicates; use limited replay only if implemented |
| Safety and stopping | Suppress stale cues and agree when disconnected sessions stop |

Exact identifiers, byte layouts, acknowledgement/replay behavior, size limits, and timeouts remain open decisions. The [Bluetooth implementation notes](implementation-notes/ble-sequence.md) retain the detailed proposed contract and test cases.
