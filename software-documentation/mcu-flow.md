# MCU layer flowchart

Proposed design for team review. Orange = MCU, green = ML, blue = Bluetooth, red = safety/fault handling.

## Session processing

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "16px", "primaryColor": "#FFE5C2", "primaryBorderColor": "#A66012", "primaryTextColor": "#30200F", "lineColor": "#525252"}, "flowchart": {"wrappingWidth": 300, "rankSpacing": 30}}}%%
flowchart TD
    accTitle: MCU input handling, metric calculation, and output
    accDescr: Collect timestamped readings, check safety independently, calculate valid features for the embedded model, and transmit results without blocking acquisition.
    start([Start]) --> init["Initialize sensors, Bluetooth,<br/>and embedded model"]
    init --> ready{"Device and session<br/>configuration ready?"}
    ready -->|No| fault["Report problem;<br/>do not start session"]
    ready -->|Yes| setup["Calibrate belts; record recovery<br/>baseline if applicable"]
    setup --> read["Read and timestamp sensors;<br/>check signal quality"]
    read --> safety["Independent safety checks:<br/>HR, SpO2, temperature, and battery"]
    safety --> critical{"Critical fault?"}
    critical -->|Yes| stop["Stop safely;<br/>send alert if possible"]
    critical -->|No| status["Update safety status;<br/>warnings can suppress coaching"]
    read --> window{"Enough valid data<br/>for a new feature window?"}
    window -->|No| wait["Report warming up or unavailable;<br/>keep older results timestamped"]
    window -->|Yes| metrics["Calculate breathing pace,<br/>TA ratio, and available HRR"]
    metrics --> model["Apply supported user context;<br/>run ML with valid required inputs,<br/>otherwise report unavailable"]
    model --> pack["Combine readings, metrics,<br/>prediction, and safety status"]
    wait --> pack
    status --> pack
    pack --> tx["Encode and queue Bluetooth data<br/>without blocking sensor collection"]
    tx --> done{"Session ended or<br/>disconnect timeout reached?"}
    done -->|No| read
    done -->|Yes| stop
    classDef mcu fill:#FFE5C2,stroke:#A66012,color:#30200F
    classDef ml fill:#D9EAD3,stroke:#477A3C,color:#203D1C
    classDef ble fill:#DDE7FF,stroke:#244FBD,color:#152C64
    classDef danger fill:#F8D4D4,stroke:#B43D3D,color:#641D1D
    class start,init,ready,setup,read,window,wait,metrics,pack,done mcu
    class model ml
    class tx ble
    class fault,safety,critical,status,stop danger
```

The two branches after sensor collection describe work that must both happen. Safety checks continue while features are warming up or ML is unavailable. Critical stops do not wait for Bluetooth delivery. The phone independently suppresses stale cues after a disconnect. Initialization retries and transmission queues must have limits.

## Metric definitions and validity

| Metric | Meaning |
| --- | --- |
| Breathing pace | Breaths per minute from accepted peaks and valleys in the breathing signals |
| TA ratio | Calibrated chest amplitude divided by abdomen amplitude; agree the exact window calculation |
| HRR1 / HRR2 / HRR3 | Exercise-end heart rate minus heart rate after 1 / 2 / 3 minutes of recovery |

HRR requires a valid exercise-end baseline and remains unavailable until its recovery time has passed. It differs from HRV in the older diagram. Invalid or missing values must not become zero or an invented prediction.

## Inputs, outputs, and open decisions

- **Inputs:** chest/abdomen signals, HR, SpO2, body/device temperature, battery, and supported session/profile context.
- **Outputs:** timestamped readings, metrics, an ML result or unavailable status, and safety information.
- **To agree:** sampling rates, window length, calibration, safety limits, and disconnect behavior.

Detailed formulas, data fields, and checks are in the [MCU implementation notes](implementation-notes/mcu-flow.md). They support implementation; they are not extra steps to present in the meeting.
