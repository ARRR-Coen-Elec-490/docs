# Black Box Diagram

## 1. Description

This file will contain a detailed black box diagram for the ABTRS System Architecture.  This diagram will build off of what has already been presented in PHASE I of this capstone project.  Through the processes of incremental design and R&D, the black box reresentation of this system has evolved past its initial rudimentary illustration.

## 2. Diagram

```mermaid
---
title: Black-Box Diagram
---
flowchart LR
    %% Inputs
    subgraph left_col[ ]
        direction TB
        subgraph bio_in[Biological Signal Inputs]
            direction TB
            bpm[Heart Rate]
            spo2[Blood Oxygen]
            chest[Chest Contribution]
            ab[Abdominal Contribution]
            body_temp[Body Temperature]

            %% Hidden structural connections
            bpm~~~spo2
            spo2~~~chest
            chest~~~ab
            ab~~~body_temp
        end

        subgraph sys_in[System Inputs]
            direction TB
            power[Power]
            temp[System<br/>Temperature]

            %% Hidden structural connections
            power~~~temp
        end
    end

    subgraph center_col[ ]
        direction TB
        %% System
        subgraph system[System]
            direction TB
            mcu[Embedded<br/>Microcontroller<br/>Layer]
            app[Mobile<br/>Application<br/>Layer]
            ml[Machine<br/>Learning<br/>Layer]
            cloud[Cloud<br/>Services<br/>Layer]

            %% Hidden structural connections
            mcu~~~app
            app~~~cloud
            cloud~~~ml
        end

        %% Environment
        subgraph env[Environment]
            direction TB
            sweat[Sweat]
            body_heat[Body Heat]

            %% Hidden structural connections
            sweat~~~body_heat
        end
    end


    %% Outputs
    subgraph out[Ouputs]
        direction TB
        audio[Audio<br/>Cues]
        visual[Visual<br/>Cues]
        chart[Live<br/>Charting]

        %% Hidden structural connections
        audio~~~visual
        visual~~~chart
    end


    %% Connections
    bio_in ===> system
    sys_in ===> system
    system ===> out

    %% Styling
    %% Hidden structural details
    style left_col fill:none, stroke:none
    style center_col fill:none, stroke:none
```

## 3. Explanation

### 3.1 Inputs

### 3.2 Outputs

### 3.3 Internal Process

### 3.4 Environment

### 3.5 Power
