# Documentation Repository

## Table of Contents <!-- omit in toc -->

- [Documentation Repository](#documentation-repository)
    - [Introduction](#introduction)
    - [Repository Structure](#repository-structure)
    - [System Architecture Documentation](#system-architecture-documentation)
    - [Hardware Documentation](#hardware-documentation)
    - [Software Documentation](#software-documentation)
    - [AI Usage Claim](#ai-usage-claim)

## Introduction

This repository serves as the Single Source of Truth (*SSOT*) for Team #23's fulfillment of the COEN/ELEC 490 2026-2027 Capstone Session.  Within this repository, one can find documentation related to every single part of this project, including hardware, software, architecture, and procurement documents.  The documentation found in this repository is updated and maintained regularly to ensure it follows all the appropriate IEEE, IEC, and ISO standards, according to each standards relevancy and availability.

## Repository Structure

```text
docs/
 ┣ ai-usage/                    # AI Prompting Templates and Logs (Empty)
 ┣ hardware-documentation/      # PCB & BOM Documents
 ┃ ┣ abtrs_torso/               # KiCAD PCB, Schematics, and Simulations
 ┃ ┣ abtrs_torso_information/   # PCB Support Documents
 ┃ ┣ bom/                       # Bill of Materials
 ┃ ┣ datasheets/                # Datasheets for all thirs-party components
 ┃ ┣ schematics/                # Auto-Generated Printables
 ┃ ┗ wiring-diagrams/           # System-Level Wiring Information
 ┣ software-documentation/      # Software Business Logic Documents
 ┣ system-architecture/         # Overarching System Architecture Documents
 ┗ user-manuals/                # Operation and Maintenance Documents (Empty)
```

## System Architecture Documentation

The ```system-architecture``` folder contains the black-box and white-box representations of the system, including the interactions betwen the hardware and software layers.  In addition, this folder contains diagrams related to the system operation including flowcharts, activity/sequence diagrams, and entity relationship diagrams.

## Hardware Documentation

The ```hardware-documentation``` folder contains all information related to the PCB and BOM.  Regarding the PCB, schematics, 3D Models, and simulations have been created to demonstrate the design and validation of the board needed for our project.  According to this PCB, a detailed and through BOM was ceated to hold all the components needed for the product being developed.  Furthermore, other 3D models, wiring diagrams, and support documents have been included to illustrate the complete hardware design of this project.

## Software Documentation

The ```software-documentation``` folder contains all the documents related to the various software sub-layers noted in white-box system architecture diagrams.  The contents of this folder expand each layer, providing detailed representations of any software business logic employed in this project.  A combination of flowcharts, activity/sequence diagrams, entity relationship diagrams, and class diagrams can be found for the different layers of the software architecture, to ensure the full illustration of the scope and responsibilities attributed to each software process found in the system.

## AI Usage Claim

In accordance with the policies put forth by both Concordia University and the COEN/ELEC 490 Capstone Committe, AI cannot be used to design or implement any aspects of the capstone project being developed by this team.  Instead, AI may only be used to enhance research or problem-solving processes across the team.  In other words, AI can be leveraged to enhance the toolsets employed by this team, but can in no way directly contribute to the final product of this project.  Failure to comply with these policies will result in grave penalties.  As such, to prevent such a scenario, the ```ai-usage``` folder has been created to contain any and all prompts used throughout the develoment of this project, and will serve as the team's method to demonstrate compliance with the policies discussed above.
