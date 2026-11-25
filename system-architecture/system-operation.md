# System Operation

## 1. Operational Flowchart

```mermaid
---
title: System Operation Flowchart
---
flowchart TD
    start([Start]) --> sys_init[System Initialization]
    sys_init[System Initialization] --> sys_check1{Sensors<br>Functional?}
    sys_check1{Sensors<br>Functional?} -->|YES| sys_check2{Battery<br>Functional?}
    sys_check1{Sensors<br>Functional?} -->|NO| retry
    retry --> sys_init

```

## 2. Operational Sequence Diagram
