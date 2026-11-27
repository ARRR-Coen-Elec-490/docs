# System Operation

## 1. Operational Flowchart

### 1.1 Overarching Process

The flowcharts shown below belong to the overarching process of the ABTRS system.  Due to the sheer size of the overarching process, the primary operational flowchart has been split into three flowcharts, all shown and explained below.  The system operational flowchart is meant to illustrate the full process of the entire system for a given post-exercise recovery session.  In addition, the flowchart considers certain errors that can be encountered, and how the system should handle them.  While the major errors have been covered here, separate flowcharts and activity sequence diagrams contain more details regarding data communication, extenuating circumstances, and other software structures that cannot be captured accurately in a flowchart.

<br>

<div align="center">
    <img src="./assets/sys-op-fc-1.png" alt="System Operation Flowchart #1">
</div>

<br>

The first of the three flowcharts shows the starting and ending points of the system's operation, including manually turning the system on, and how the system must terminate after the user has completed their session.  On the right side of the diagram, the flowchart covers two major errors encountered during system initialization.  In the case of sensor connectivity issues, or low battery, the user will be notified so as to take appropriate action, before the system begins termination.  If neither error is encountered, the system will remain on standby, until the user commences a session.  Once a session is started, the mobile application must create the appropriate context for the current session, gathering important user data and inferring important comparison metrics.  Once this sub-process is complete, the session context is transmitted to the embedded system hostedby the user's wearable device.

<br>

<div align="center">
    <img src="./assets/sys-op-fc-2.png" alt="System Operation Flowchart #2">
</div>

<br>

The second of the three flowcharts shows the bulk of the system's processing logic, shared between all the major software architecture layers.  The process shown by this flowchart is characterized by the primary system loop, which will be active throughout a given post-exercise recovery session.  The input handling process will package the incoming time-stamped raw sensor values with the session context created by the mobile application.  Once this sub-process is complete, the embedded layer must ensure the created data packet is valid (*i.e., not missing any sensor data*).  If the produced packet it valid, the raw sensor data must be pre-processed to calculate and extract key metrics for breathing and recovery analysis.  These metrics, called *features*, along with certain contextual values, must be passed to the machine learning model, which itself will conduct the breathing and recovery analysis, before producing its decisions, called *labels*.  These labels will then be packaged along with the raw sensor data and the windowed metrics in the output handling sub-process, before finally being transmitted to the mobile application.  Once this entire process is complete, the mobile application must determine if the session is complete.  If it is not, the primary loop repeats, whereas if it is, the mobile application must execute the post-session processing sub-process.

Interestingly, the paragraph above has only explored the case where the sensor data is valid.  In the unlikely event that the sensor data is invalid (*e.g., corrupted, missing, inaccurate, etc.*), the system will keep polling for valid sensor data.  If it cannot find any, the system will commence the failsafe process, wherein the remainder of the session will follow a standard box breathing pattern.  This will allow the user to finish their session with a proven recovery method, while the system attempts to find the next valid data packet.  Once this warning signal is received by the mobile application layer, steps will be taken to notify the user of the incident, and the subsequently account for the failsafe portion in the post-session report.

<br>

<div align="center">
    <img src="./assets/sys-op-fc-3.png" alt="System Operation Flowchart #3">
</div>

<br>

The final flowchart shows the last portion of the primary system loop, which is entirely contained within the mobile application layer.  Here, the user interface is adjusted based on the data being received from the embedded layer (*OR, in the case of failsafe, adjusted to demonstrate the standard box-breathing cues*).  Adjustments to the audio-visual cues are made based on the labels received indirectly from the machine learning model.  Any data recived from the current packet is also stored locally with other historical data, which will be pertinent for post-session dashboards.

### 1.2 "Create Session Context" Process

<br>

<div align="center">
    <img src="./assets/create-session-context.png" alt="Create Session Context Process">
</div>

<br>

The "Create Session Context" sub-process describes the creation of a given post-exercise recovery session's context.  This includes the age, biological sex, height, and weight of the user, as well as expected recovery metrics based on these values.  Once all this information is retrieved from the local storage, a proper session context packet can be created, encoded, and compressed, based on the data transfer communication protocol.  Once these steps are completed, the resulting packet is transmitted to the embedded layer, a process which is described in the overarching operational flowchart above.

### 1.3 "System Cleanup" Process

<br>

<div align="center">
    <img src="./assets/system-cleanup.png" alt="System-Cleanup Process">
</div>

<br>

The "System-Cleanup" sub-process flowchart describes the very end of the system operation for a given post-exercise recovery session.  Once the mobile application creates and transmits the "End Session" signal, the user will be returned to the homescreen, where they have the option of either starting another session, viewing a post-session (*OR inter-session*) dashboard, or changing settings, among other actions.  On the embedded side, once the signal is transmitted, the microcontroller must take steps to terminate its different processes, trash unused data, and ultimately enter a dormant cycle.  Finally, the user has the option to turn the system off, depending on whether or not he user must charge the system.

### 1.4 "Input Handling" Process

<br>

<div align="center">
    <img src="./assets/input-handling.png" alt="Input Handling Process">
</div>

<br>

The "Input Handling" sub-process flowchart describes the sensor input data handling, which includes reading, time-stamping, and queueing the data from the sensors.  The act of queueing the sensor data will facilitate the windowed calculations, which are key to the machine learning model's analysis.  Along with the sensor data, the session context must be considered, in order to include adequate comparison metrics for the incoming data stream.  Finally, the input handling sequence must validate the incoming data to ensure that a thourough analysis can be conducted.  Finally, as the operational flowchart is focusing on the system's rimary function, being the anlysis of the user's breathing and recovery, the system temperature analysis is not included in this sequence.  Since this is considered a secondary process in the system, the battery sensing and management process will be communicated through other documents.

### 1.5 "Output Handling" Process

<br>

<div align="center">
    <img src="./assets/output-handling.png" alt="Output Handling Process">
</div>

<br>

The "Output Handling" sub-process flowchart describes the creation of a *Session Data Packet*, which must be transmitted to the mobile application layer for further processing.  This data packet is composed of the window of real-time data used for the current loop, the metrics calculated from this window, and the machine learning labels inferred from these metrics.  Each of these three inputs has a different purpose when transmitted to the mobile application layer.  Once the packet is created, it must be encoded and compressed according to the data transfer layer protocol, so as to reduce the load imposed on the mobile application layer by the real-time data stream.

### 1.6 "Post-Session Processing" Process

<br>

<div align="center">
    <img src="./assets/post-session-processing.png" alt="Post-Session Processing Process">
</div>

<br>

The "Post-Session Processing" sub-process flowchart describes the creation of the final UI prompt, once the session has been completed.  The final UI prompt is composed of a final display graphics, as well as a short audio indicator.  Once this UI prompt is displayed and projected, the final step of the sub-process will be to create signal to end the session, which will be passed to the [System-Cleanup](#13-system-cleanup-process) sub-process.

## 2. Operational Sequence Diagram

### 2.1 Embedded Device - Mobile App Interaction Sequence

The sequence diagram shows the two-way communication between the embedded device and the mobile application across the entire lifecycle of a monitoring session. A more detailed view of how the embedded device and mobile app communicate to check statuses (ex: battery), ensure proper configurations, and start and stopping sessions.

<br>

<div align="center">
    <img src="./assets/sys-op-sd.png" alt="System Communication Sequence Diagram">
</div>

<br>
