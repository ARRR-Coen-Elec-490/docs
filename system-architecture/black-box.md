# Black Box Diagram

## 1. Description

This file will contain a detailed black box diagram for the ABTRS System Architecture.  This diagram will build off of what has already been presented in PHASE I of this capstone project.  Through the processes of incremental design and R&D, the black box reresentation of this system has evolved past its initial rudimentary illustration.

## 2. Diagram

<div align="center">
    <img src="black-box.png" alt="Black Box Diagram">
</div>

## 3. Explanation

### 3.1 Inputs

There are two main classes of inputs that will be entering the system at any given time during its operation.  The first class of inputs are the "Biological Signal Inputs", represented with the colour <span style="color: blue;">blue</span>.  Such inputs are gathered from the physical state of the user during post-exercise recovery, and will be passed to the PCB via several sensors.  Before arriving at the microcontroller, the signals must be passed through specific signal processing circuits to ensure that noise is attenuated.  Once clean, the signals will be passed through the system layers as required by its varying processes.

The second class of inputs are the "System Inputs", and are represented with the colour <span style="color: red;">red</span>.  The first input is the temperature of the system, which will be monitored to ensure that the system does not overheat (*e.g., thermal runaway*), as well as to ensure the well-being of the user wearing the device.  The second is the power being supplied to the system, which will be discussed in [Section 3.5](#35-power).

### 3.2 Outputs

The outputs of the system are generally easier to classify, as they are all produced via the mobile companion application.  All the outputs are represented by the colour <span style="color: orange;">orange</span>.  The outputs are made up of audio and visual cues to help the user during their post-exercise recovery session.  Both options were chosen, so as to ensure accessibility for the product.  Finally, the live charting will also allow the user to stay informed on other aspects of their breathing, such as their heart rate and blood-oxygen level, among others.

### 3.3 Internal Process

The internal process of the system  can be divided into four main layers, namely the "Embedded MCU", "Mobile Application", "Cloud Services", and "Machine Learning" layers.  All four layers are represented by the colour <span style="color: purple;">purple</span>.  More details related to the interaction between these layers, their roles, and their sub-architetures, will be communicated through other software documents, including the White-Box System Architecture Diagram.

### 3.4 Environment

The environment in the case of this specific project will pertain to the end user, who will be wearing the system on their torso area.  In this regard, the body heat and sweat coming from the user could have an effect on certain sensor readings.  On the other hand, the user must be protected from being harmed by the biomedical device (*e.g., getting burned, hyperventilating, etc.*).  While this device is non-invasive, precautions must still be taken to ensure the safety of the user, while maintaining a reliable system.

### 3.5 Power

Finally, the power being supplied to the system is a crucial aspect of the project.  In this way, a concrete battery sensing and management system must be defined to ensure that the systerm canw ork reliably within the desired amount of time.  The details of this system will be communicated through separate hardware and software documentation, available within this repository.
