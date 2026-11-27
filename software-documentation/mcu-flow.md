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
    ready -->|Yes| setup["Calibrate belts for 10 seconds;<br/>record exercise-end HR baseline"]
    setup --> read["Acquire paired belts at 50 Hz;<br/>timestamp and check sensors at 1 Hz"]
    read --> safety["Independent safety checks:<br/>HR, SpO2, temperature, and battery"]
    safety --> critical{"Critical fault?"}
    critical -->|Yes| stop["Stop safely;<br/>send alert if possible"]
    critical -->|No| status["Update safety status;<br/>warnings can suppress coaching"]
    read --> health{"Required sensor readings valid?"}
    health -->|Yes| reset["Reset invalid counter"]
    reset --> window{"Valid 20-second feature window?"}
    health -->|No| invalid["Suppress adaptive cues;<br/>send invalid status; increment counter"]
    invalid --> count{"Three invalid checks?"}
    count -->|No| read
    count -->|Yes| failed["SENSOR_FAULT: notify phone;<br/>start 30-second recovery deadline"]
    failed --> fallback["Phone offers opt-in unmonitored<br/>4-4-4-4 guide; never for critical faults"]
    fallback --> poll["Poll sensors at 1 Hz;<br/>keep safety and STOP active"]
    poll --> recover{"Three valid checks + full window<br/>and user confirms resume?"}
    recover -->|Yes| read
    recover -->|No| expired{"30-second deadline reached?"}
    expired -->|No| poll
    expired -->|Yes| stop
    window -->|No| wait["Report warming up or unavailable;<br/>keep older results timestamped"]
    window -->|Yes| metrics["Calculate breathing pace,<br/>TA ratio, and available HRR"]
    metrics --> model["Run released MCU classifier<br/>on pace and normalized TA ratio;<br/>otherwise report unavailable"]
    model --> pack["Combine metrics, prediction,<br/>selected HR / SpO2 / temperature,<br/>and status; no raw belt samples"]
    wait --> pack
    status --> pack
    pack --> tx["Encode and queue Bluetooth data<br/>without blocking sensor collection"]
    tx --> done{"STOP, 300-second end,<br/>or 5-second disconnect?"}
    done -->|No| read
    done -->|Yes| stop
    classDef mcu fill:#FFE5C2,stroke:#A66012,color:#30200F
    classDef ml fill:#D9EAD3,stroke:#477A3C,color:#203D1C
    classDef ble fill:#DDE7FF,stroke:#244FBD,color:#152C64
    classDef danger fill:#F8D4D4,stroke:#B43D3D,color:#641D1D
    class start,init,ready,setup,read,window,wait,metrics,pack,done,reset,poll mcu
    classDef phone fill:#E2EBFA,stroke:#456A9E,color:#172C4D
    class fallback phone
    class model ml
    class tx ble
    class fault,safety,critical,status,stop,health,invalid,count,failed,recover,expired danger
```

Safety checks run independently of inference. Raw belt samples stay on the MCU; only processed metrics, predictions, selected telemetry, and status are sent. WARMUP or missing early HRR is not a broken sensor. The one-second health check fails on missing/stale required readings or sensor/status errors.

Three consecutive invalid checks enter SENSOR_FAULT. Adaptive cues stop on the first invalid check. The phone may offer the opt-in unmonitored box guide during the 30-second recovery period; a critical fault stops everything. Recovery requires three valid checks, a fresh full feature window, and user confirmation. The deadline always provides an exit for a broken sensor.

Exact calculations, limits, and protocol are selected in [design specification v1](design-spec-v1.md).
