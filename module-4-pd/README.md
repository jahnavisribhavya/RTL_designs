# 🕰️ Module 4 — Clock Tree Synthesis, Timing Checks & Physical Design Optimization

<p>
  <img src="https://img.shields.io/badge/Flow-OpenLane%20%2F%20OpenROAD-green" alt="OpenLane / OpenROAD">
  <img src="https://img.shields.io/badge/Synthesis-Yosys-red" alt="Yosys">
  <img src="https://img.shields.io/badge/STA-OpenSTA-orange" alt="OpenSTA">
  <img src="https://img.shields.io/badge/PDK-SKY130-blue" alt="SKY130">
  <img src="https://img.shields.io/badge/Constraints-SDC-yellow" alt="SDC">
  <img src="https://img.shields.io/badge/Env-Docker-lightblue" alt="Docker">
</p>

> Part of the SKY130 Physical Design module series.

## 📖 Module Overview

This document walks through the physical-implementation stage of the `picorv32a` design on the OpenLane/OpenROAD flow with the SkyWater SKY130 standard-cell library. It covers standard-cell placement, Clock Tree Synthesis (CTS), clock buffering and distribution, setup/hold timing analysis, clock skew, crosstalk, and how the supporting Liberty, LEF, SDC, and Tcl files drive each stage.

|  |  |
|---|---|
| 🛠️ **Tools used** | OpenLane, OpenROAD, Yosys, OpenSTA, SKY130 PDK, Liberty (.lib), LEF, SDC, Tcl, Docker |
| 🧩 **Example design(s)** | `picorv32a` |
| 📋 **Prerequisites** | OpenLane/OpenROAD environment running in Docker, SKY130 PDK installed, a synthesized `picorv32a` netlist, base SDC constraints defined |

## 📑 Contents

1. Technology Libraries and Physical Design Files
   1.1 Standard-Cell Timing Corners (Fast, Slow & Typical)
   1.2 Understanding the LEF File
2. Overview of the Physical Design Flow
3. Logic Synthesis and Cell Placement
   3.1 Executing Synthesis
   3.2 Placement of Standard Cells
   3.3 Detailed Placement View
4. Timing Constraints and Setup
   4.1 Base SDC Constraints File
   4.2 Pre-CTS and Tcl Configuration
5. Clock Tree Synthesis (CTS)
   5.1 CTS Goals and Clock-Tree Structure
   5.2 CTS Settings (cts.tcl)
6. Clock Buffering and Distribution
   6.1 Inserting Clock Buffers
   6.2 Shielding the Clock Net
7. Setup/Hold Timing Checks
   7.1 Setup-Time Requirements
   7.2 Hold-Time Requirements
   7.3 Setup Checks with an Ideal Clock
   7.4 Hold-Time Checks
8. Clock Skew and Signal Crosstalk
   8.1 Understanding Clock Skew
   8.2 Crosstalk Effects and Delta Delay
9. Timing Reports and Slack Evaluation
   9.1 Understanding Timing Slack
   9.2 Maximum-Delay Setup Checks
   9.3 Minimum-Delay Hold Checks
   9.4 Reading Delay Tables
10. Physical Design Settings and Routing Details
    10.1 Converting Labels into Ports
    10.2 Grid-to-Track Mapping and `tracks.info`
- Author

---

## 1️⃣ Technology Libraries and Physical Design Files

Technology files tell the physical-design tools how standard cells, routing layers, and timing behave. The SKY130 standard-cell library ships fast, slow, and typical characterization corners, plus a LEF description of each cell's physical footprint.

### 1.1 Standard-Cell Timing Corners (Fast, Slow & Typical)

**Fast corner** — cells characterized under conditions where they switch relatively quickly; used to study minimum-delay behavior and hold timing.

<img width="1140" height="652" alt="Screenshot 2026-10-15 175122" src="https://github.com/user-attachments/assets/90cad32a-9fdf-4517-8125-a997ac68954e" />


**Slow corner** — cells characterized under conditions where they switch relatively slowly; relevant to maximum-delay and setup timing.

<img width="1137" height="668" alt="Screenshot 2026-10-15 175133" src="https://github.com/user-attachments/assets/2bf0570d-cd87-4513-be45-746537df2d0a" />



**Typical corner** — nominal characterized conditions, used for baseline/typical timing evaluation.


<img width="931" height="597" alt="Screenshot 2026-10-15 175141" src="https://github.com/user-attachments/assets/07b6dad2-9671-445b-a84d-459f890fac1f" />


### 1.2 Understanding the LEF File

The LEF format captures the physical characteristics of a standard cell — dimensions, pin locations, and routing-related geometry needed for placement and routing.


<img width="792" height="626" alt="Screenshot 2026-10-15 175153" src="https://github.com/user-attachments/assets/9351a477-70a2-4312-b180-a095fd2d3209" />


---

## 2️⃣ Overview of the Physical Design Flow

The physical-design flow turns a synthesized netlist into an implementation by placing cells, building the clock network, routing connections, and checking timing:

```text
RTL Design
    |
    v
Synthesis
    |
    v
Floorplanning
    |
    v
Standard-Cell Placement
    |
    v
Placement Optimization
    |
    v
Clock Tree Synthesis
    |
    v
Clock Distribution
    |
    v
Timing Analysis
    |
    v
Routing and Optimization
    |
    v
Physical Verification
```

**Synthesis** maps RTL onto standard cells from the target library. **Floorplanning** sets the core area and design boundaries. **Placement** assigns physical locations to cells. **CTS** builds the clock distribution network with buffers and clock cells. **Routing** wires cells together on the metal layers. **Timing analysis** checks whether the implementation meets its constraints.

---

## 3️⃣ Logic Synthesis and Cell Placement

### 3.1 Executing Synthesis

Synthesis converts the RTL into a gate-level netlist mapped onto the chosen standard-cell library; this netlist is what feeds the rest of the physical flow.

<img width="1117" height="645" alt="Screenshot 2026-10-15 175215" src="https://github.com/user-attachments/assets/5ac95509-c7e7-48a2-b68d-c3a2221cf9ae" />


### 3.2 Placement of Standard Cells

Placement locates standard cells inside the core area, weighing cell density, wirelength, connectivity, and timing — a good placement keeps interconnect delay down and eases later timing optimization.


<img width="1135" height="671" alt="Screenshot 2026-10-15 175223" src="https://github.com/user-attachments/assets/7e212585-f27d-49aa-abba-f3a7e9e25048" />


### 3.3 Detailed Placement View

A zoomed-in view of the placement makes it easier to see how cells are physically distributed and spatially related within the core.

<img width="1142" height="646" alt="Screenshot 2026-10-15 175233" src="https://github.com/user-attachments/assets/e1233846-26c3-4b5c-9627-3518630c00f9" />


---

## 4️⃣ Timing Constraints and Setup

Timing constraints capture the design's operating requirements and feed Static Timing Analysis. SDC (Synopsys Design Constraints) is the format used here to define the clock, I/O delays, and timing uncertainty.

### 4.1 Base SDC Constraints File

`my_base.sdc` holds the design's core timing requirements, typically including:

- Clock period and clock definition
- Input and output delays
- Clock uncertainty
- Input transition times
- Output loads


<img width="1111" height="636" alt="Screenshot 2026-10-15 175242" src="https://github.com/user-attachments/assets/c861b0e7-0e45-4e3c-87b1-8e9c6defc4fc" />


### 4.2 Pre-CTS and Tcl Configuration

The pre-CTS configuration holds the settings used for implementation just before Clock Tree Synthesis runs. A separate Tcl configuration file carries additional design/flow settings for physical implementation — these must stay consistent with the chosen technology, design, and timing requirements.


<img width="1112" height="797" alt="Screenshot 2026-10-15 175258" src="https://github.com/user-attachments/assets/de49c583-fd8a-4225-ae40-1c7e4b9c441e" />


---

## 5️⃣ Clock Tree Synthesis (CTS)

### 5.1 CTS Goals and Clock-Tree Structure

**CTS** builds the distribution network that carries the clock from its source to every sequential element. As the number of clock sinks grows, distribution gets harder because of capacitive loading, long interconnects, insertion delay, arrival-time mismatches, clock skew, and transition requirements — CTS inserts buffers and organizes the network to manage all of this.

Its main goals are to:

1. Reach every required clock sink.
2. Keep skew between sequential elements under control.
3. Keep clock transition times acceptable.
4. Manage insertion delay and network loading.
5. Support setup and hold requirements.
6. Improve overall clock distribution quality.

A clock tree is generally made of a clock source, intermediate buffers, branch points, and sinks; its structure shapes latency, skew, power, and overall timing behavior.


<img width="1140" height="585" alt="Screenshot 2026-10-15 175307" src="https://github.com/user-attachments/assets/a09bb820-d03e-462d-8277-87fdae89ab3d" />


<img width="1157" height="587" alt="Screenshot 2026-10-15 175315" src="https://github.com/user-attachments/assets/e765990d-c791-43a4-a0f7-ba09eb24983f" />


### 5.2 CTS Settings (`cts.tcl`)

`cts.tcl` holds the settings that govern the CTS run and how the clock distribution network gets built.

<img width="1066" height="532" alt="Screenshot 2026-10-15 175325" src="https://github.com/user-attachments/assets/13bc72c4-5b68-493b-aff5-16f7c6bc459f" />


---

## 6️⃣ Clock Buffering and Distribution

### 6.1 Inserting Clock Buffers

Clock buffers drive the capacitive load of clock sinks and help distribute the clock across the design while controlling loading and transition times. Their count, sizing, and placement directly affect insertion delay, skew, and power. Inserting buffers splits one large clock load into smaller loads each buffer can drive, which keeps the clock signal intact and lets it reach physically separated flip-flops.

<img width="1421" height="895" alt="Screenshot 2026-10-15 175339" src="https://github.com/user-attachments/assets/652ef2c5-b7af-4273-93f0-c53d4513ad58" />


### 6.2 Shielding the Clock Net

Shielding routes a fixed-potential wire alongside a clock net to cut down unwanted coupling from neighboring signal wires — it improves signal integrity at the cost of extra routing resources.

<img width="1138" height="613" alt="Screenshot 2026-10-15 175349" src="https://github.com/user-attachments/assets/9cf15e2b-0476-41c5-afd4-0e470ad259d7" />



---

## 7️⃣ Setup/Hold Timing Checks

Setup and hold define the window during which data at a flip-flop's input must stay stable around the active clock edge.

### 7.1 Setup-Time Requirements

Setup time is the minimum interval data must be stable **before** the active clock edge. A setup violation happens when data shows up too late. For a simplified register-to-register path:

$$T_{cq} + T_{comb} + T_{setup} \leq T_{period}$$

where $T_{cq}$ is the launching flip-flop's clock-to-Q delay, $T_{comb}$ is the combinational/interconnect delay, $T_{setup}$ is the capturing flip-flop's setup time, and $T_{period}$ is the clock period. The real requirement also folds in clock arrival times and timing uncertainty.

### 7.2 Hold-Time Requirements

Hold time is the minimum interval data must stay stable **after** the active clock edge. A hold violation happens when data changes too soon. For a simplified path:

$$T_{cq,min} + T_{comb,min} \geq T_{hold}$$

The actual requirement depends on the launch/capture clock arrival times and the design's timing constraints — hold is especially sensitive to minimum data-path delay.

### 7.3 Setup Checks with an Ideal Clock

An ideal clock ignores clock-distribution delay and skew, which makes it useful for looking at data-path timing on its own before the implemented clock tree is factored in.

<img width="1095" height="657" alt="Screenshot 2026-10-15 175359" src="https://github.com/user-attachments/assets/ce63f5e4-7d71-409c-b05e-1c52e650a419" />

<img width="1057" height="367" alt="Screenshot 2026-10-15 175409" src="https://github.com/user-attachments/assets/716e4657-c657-4d8f-ae86-694047e6d76f" />


### 7.4 Hold-Time Checks

Hold analysis checks the minimum data-path delay against the hold requirement after the active edge; negative hold slack means the minimum-delay requirement isn't met.


Setup and hold checks together evaluate the timing relationship between launching and capturing elements — a path can pass setup and still fail hold, so both checks are necessary.

<img width="981" height="536" alt="Screenshot 2026-10-15 175417" src="https://github.com/user-attachments/assets/1bf39864-6002-4da9-ad61-637cb7b878df" />


<img width="1116" height="540" alt="Screenshot 2026-10-15 175428" src="https://github.com/user-attachments/assets/e74b95be-5393-4696-83f4-0ec33066afb8" />


---

## 8️⃣ Clock Skew and Signal Crosstalk

### 8.1 Understanding Clock Skew

Clock skew is the difference in clock arrival time between two sequential elements, caused by the clock passing through different buffer/interconnect combinations on its way to each one.

| Skew type | Meaning |
|---|---|
| Positive skew | Capture clock arrives later than the launch clock |
| Negative skew | Capture clock arrives earlier than the launch clock |
| Zero skew | Clock reaches both registers at the same time |

Skew affects setup and hold differently depending on its direction and size.

### 8.2 Crosstalk Effects and Delta Delay

Crosstalk is unwanted electrical coupling between neighboring interconnects — switching on one net can shift the voltage or delay of a nearby net through capacitive or inductive coupling, affecting transition times, propagation delay, timing margin, and signal integrity.

**Crosstalk-induced delta delay** is the change in propagation delay caused by a neighboring net's switching; depending on the relative switching direction and timing, coupling can speed up or slow down a signal.

<img width="1101" height="593" alt="Screenshot 2026-10-15 175439" src="https://github.com/user-attachments/assets/4a25e442-1309-40a7-b82b-2358e188cf22" />


---

## 9️⃣ Timing Reports and Slack Evaluation

Static Timing Analysis checks whether the design's timing requirements hold, by comparing signal arrival times against required arrival times — without needing input-vector simulation.

### 9.1 Understanding Timing Slack

For setup analysis:

$$\text{Setup Slack} = \text{Required Arrival Time} - \text{Data Arrival Time}$$

| Slack | Meaning |
|---|---|
| Positive slack | Timing requirement satisfied |
| Zero slack | Requirement met right at the boundary |
| Negative slack | Timing violation |

Setup and hold slack each use their own maximum-delay / minimum-delay checks.

### 9.2 Maximum-Delay Setup Checks

Maximum-delay analysis checks whether data beats the setup deadline by evaluating the longest relevant paths; negative slack here flags a setup violation.

<img width="977" height="506" alt="Screenshot 2026-10-15 175453" src="https://github.com/user-attachments/assets/d10d3941-aae3-481c-8657-de76557db1a7" />


### 9.3 Minimum-Delay Hold Checks

Minimum-delay analysis checks whether data arrives late enough to satisfy hold, by evaluating the shortest relevant paths; negative slack here flags a hold violation.


<img width="557" height="451" alt="Screenshot 2026-10-15 175508" src="https://github.com/user-attachments/assets/480e67e8-a652-4be4-8ea1-994ea0b98147" />


### 9.4 Reading Delay Tables

Timing reports and delay tables list cell delay, interconnect delay, arrival/required times, and slack — helpful for spotting critical paths and understanding how much cells vs. interconnect contribute to overall timing.

<img width="1077" height="653" alt="Screenshot 2026-10-15 175517" src="https://github.com/user-attachments/assets/3fc8152d-b122-4701-b558-0d3b17160132" />


---

## 🔟 Physical Design Settings and Routing Details

### 10.1 Converting Labels into Ports

Converting labels to ports is a preparation step tied to the design's connectivity and interface representation ahead of physical implementation.

<img width="1100" height="632" alt="Screenshot 2026-10-15 175532" src="https://github.com/user-attachments/assets/b17130f6-b0db-4884-93da-b3d4fe5cc9d5" />


### 10.2 Grid-to-Track Mapping and `tracks.info`

Routing-track information marks where wires can be placed on each routing layer; grid information is converted into track information to set this up.


<img width="1072" height="632" alt="Screenshot 2026-10-15 175544" src="https://github.com/user-attachments/assets/c59d69ce-3a63-4366-bb4f-64e4ed101893" />


`tracks.info` holds the routing-track data used through the rest of the physical implementation flow.

<img width="165" height="228" alt="Screenshot 2026-10-15 175602" src="https://github.com/user-attachments/assets/91d9f14f-88cf-4d25-97bf-3767220bb8b9" />



---

## ✅ Key Takeaways

- ✅ Placement directly affects timing — cell locations shape interconnect length and propagation delay.
- ✅ Clock buffers are what make clock distribution practical, spreading a single large clock load across manageable branches.
- ✅ CTS-introduced insertion delay and skew feed straight into setup and hold timing behavior.
- ✅ Setup and hold checks are complementary — a path can clear one and still fail the other, so both maximum- and minimum-delay analysis are needed.
- ✅ Crosstalk-induced delta delay can shift propagation delay and skew, and shielding is one way to control it.
- ✅ Liberty, LEF, and SDC files are the backbone that timing analysis and physical implementation both depend on.
- ✅ Slack values and path reports are the practical tools for finding and fixing timing violations during optimization.

## 👤 About the Author

**Jahnavi Sri Bhavya**
B.Tech, Electronics and Communication Engineering
Anurag University, Hyderabad

---
