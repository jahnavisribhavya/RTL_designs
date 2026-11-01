# 🔧 VSDBabySoC-3 — Synthesis Review, Static Timing Analysis, and Physical Design Flow

<p>
  <img src="https://img.shields.io/badge/Tool-Yosys-green" alt="Yosys">
  <img src="https://img.shields.io/badge/Tool-OpenSTA-blue" alt="OpenSTA">
  <img src="https://img.shields.io/badge/Tool-OpenROAD-purple" alt="OpenROAD">
  <img src="https://img.shields.io/badge/PDK-SKY130-red" alt="SKY130">
  <img src="https://img.shields.io/badge/Language-Verilog-9cf" alt="Verilog">
</p>

> Part of the RTL Design and Synthesis Workshop series.

## 📖 Summary

This document covers two connected stages of the VSDBabySoC work. The first half recaps what synthesis does to an RTL design and why a gate-level netlist can look very different from the RTL it came from. The second half moves on to static timing analysis with OpenSTA across multiple PVT corners, in both a VM (pre-route) and a Codespace (post-route), followed by a walkthrough of the OpenROAD physical design flow for the same design.

| | |
|---|---|
| 🛠️ **Tools used** | Yosys, OpenSTA, OpenROAD |
| 🧩 **Example design** | VSDBabySoC on the SKY130 HD platform |
| 📋 **Prerequisites** | Familiarity with RTL synthesis, gate-level simulation, and the Linux terminal |

## 📑 Contents

- 1. Why Internal Waveforms May Differ
- 2. RTL Compared with Gate-Level
- 3. Part Played by the SKY130 Standard-Cell Library
- 4. Key Observations
  - 4.1 Coding Style in RTL Matters
  - 4.2 Synthesis Alters Structure
  - 4.3 Optimization Alters the Implementation
  - 4.4 The Technology Library Matters
  - 4.5 The Importance of GLS
  - 4.6 Internal Signals May Change
- 5. Full Synthesis Command Sequence
- 6. Static Timing Analysis with OpenSTA
  - 6.1 Background
  - 6.2 Key Timing Parameters
  - 6.3 OpenSTA Command Usage
- 7. STA of VSDBabySoC in the VM
  - 7.1 Scope
  - 7.2 VM run.tcl Script
  - 7.3 VM Timing Reports for Selected Corners
  - 7.4 WNS and TNS Graphs (VM)
  - 7.5 Findings
- 8. SKY130 Timing Libraries and PVT Corners
  - 8.1 Background
  - 8.2 SKY130 HD Library Set
  - 8.3 Process Corner Types
- 9. Multi-Corner STA in Codespace
  - 9.1 Scope
  - 9.2 Setup for the Analysis
  - 9.3 TCL Script Used in Codespace
  - 9.4 Commands for Timing Reports
  - 9.5 PVT Corners Covered
  - 9.6 Timing Reports for Four Selected Corners
  - 9.7 Overall Summary Report
  - 9.8 WNS and TNS Graphs (Codespace)
  - 9.9 Findings
- 10. OpenROAD Physical Design Flow
  - 10.1 Background
  - 10.2 Synthesis Step
  - 10.3 Floorplan Stage
  - 10.4 Placement Stage
  - 10.5 Routing Stage
  - 10.6 Findings
- 11. Key Learnings
- About the Author

---

## 1️⃣ Why Internal Waveforms May Differ

Synthesis may perform:

- Logic optimization
- Constant propagation
- Logic merging
- Redundant logic removal
- Technology mapping
- Hierarchy flattening
- Internal signal renaming

Because of this, the internal gate-level structure does not have to be identical to the RTL structure. A different internal waveform does not automatically indicate a functional error — the important comparison is the behaviour of the relevant observable signals and outputs.

---

## 2️⃣ RTL Compared with Gate-Level

| RTL | Gate-Level |
|---|---|
| Higher abstraction | Lower abstraction |
| Behaviour-oriented | Structure-oriented |
| Easier to read | More detailed |
| Technology independent | Technology mapped |
| Describes intended functionality | Describes cell connectivity |
| Used for functional verification | Used for post-synthesis verification |

---

## 3️⃣ Part Played by the SKY130 Standard-Cell Library

The standard-cell library determines which technology-specific cells can be used during mapping. The relationship is:

```text
RTL
 ↓
Generic Logic
 ↓
Technology Mapping
 ↓
SKY130 Standard Cells
```

Examples of standard-cell types include:

```text
INV
NAND
NOR
AND
OR
XOR
MUX
D Flip-Flop
```

The selected technology library therefore influences the structure of the final gate-level netlist.

---

## 4️⃣ Key Observations

### 4.1 Coding Style in RTL Matters

The MUX experiments demonstrated that RTL coding style influences how synthesis interprets the design.

### 4.2 Synthesis Alters Structure

The gate-level netlist can be structurally very different from the RTL.

### 4.3 Optimization Alters the Implementation

Redundant or unnecessary logic may be simplified or removed while preserving functionality.

### 4.4 The Technology Library Matters

The available technology library determines the cells used during technology mapping.

### 4.5 The Importance of GLS

RTL simulation verifies the original RTL, while GLS verifies the synthesized gate-level implementation.

### 4.6 Internal Signals May Change

Synthesis may rename, optimize, or remove internal signals, so functional comparison should focus on meaningful signals and outputs.

---

## 5️⃣ Full Synthesis Command Sequence

```bash
yosys

read_verilog src/module/vsdbabysoc.v
read_verilog -I src/include src/module/rvmyth.v
read_verilog -I src/include src/module/clk_gate.v

read_liberty -lib lib/avsdpll.lib
read_liberty -lib lib/avsddac.lib
read_liberty -lib lib/sky130_fd_sc_hd__tt_025C_1v80.lib

synth -top vsdbabysoc

dfflibmap -liberty lib/sky130_fd_sc_hd__tt_025C_1v80.lib

opt

abc -liberty lib/sky130_fd_sc_hd__tt_025C_1v80.lib

check

show vsdbabysoc

flatten
setundef -zero
clean -purge
rename -enumerate

write_verilog -noattr netlist/babysoc_netlist_new.v
```

---

## 6️⃣ Static Timing Analysis with OpenSTA

### 6.1 Background

Static Timing Analysis (STA) evaluates the timing behaviour of a digital circuit without requiring exhaustive functional simulation. It analyses signal propagation delays and checks whether timing paths satisfy the specified timing constraints.

OpenSTA is an open-source static timing analysis tool. In this work, it was used to examine timing paths in VSDBabySoC using SKY130 HD standard-cell timing libraries. The analysis involves loading the design netlist, timing constraints, and libraries, generating timing reports, and evaluating timing behaviour across different Process, Voltage, and Temperature (PVT) conditions.

### 6.2 Key Timing Parameters

| Parameter | Description |
|---|---|
| Startpoint | Beginning of a timing path |
| Endpoint | End of a timing path |
| Arrival time | Time at which a signal reaches the endpoint |
| Required time | Time by which the signal must arrive |
| Slack | Difference between required time and arrival time, interpreted according to the timing check |
| WNS | Worst Negative Slack; the worst reported timing slack |
| TNS | Total Negative Slack; accumulated negative slack across violating paths |

Positive slack generally indicates that a path meets its timing requirement. Negative slack indicates a timing violation.

### 6.3 OpenSTA Command Usage

Check whether OpenSTA is available in the VM:

```bash
which sta
sta -version
```

Inspect minimum- and maximum-delay timing paths:

```tcl
report_checks -path_delay min_max -fields {nets cap slew input_pins fanout}
```

The report provides timing-path details, including net information, capacitance, slew, input pins, and fanout.

---

## 7️⃣ STA of VSDBabySoC in the VM

### 7.1 Scope

Static Timing Analysis was performed on the synthesized VSDBabySoC design in the virtual machine using OpenSTA and SKY130 HD timing libraries. This analysis focuses on the **pre-route** design. The results help identify timing violations and understand how timing behaviour changes across PVT corners.

The VM screenshots are stored in the `images/` directory. The original filenames contain `(report checks)`.

### 7.2 VM run.tcl Script

The two parts of the VM `run.tcl` file:

**Part 1**

<img width="1702" height="870" alt="Screenshot 2026-11-01 120654" src="https://github.com/user-attachments/assets/ccf2ebf2-c347-4ec5-8a56-a009a885b72c" />



**Part 2**

<img width="1205" height="848" alt="Screenshot 2026-11-01 120705" src="https://github.com/user-attachments/assets/cfe0e5d3-3a43-4b31-93b5-4f94102faeb3" />



### 7.3 VM Timing Reports for Selected Corners

Each selected corner has two screenshot parts.

**Corner 1: SS, -40°C, 1.28 V**

This corner produced the following pre-route timing results:

| Metric | Value |
|---|---:|
| WNS | -62.56 ns |
| TNS | -13957.59 ns |

<img width="573" height="753" alt="Screenshot 2026-11-01 120718" src="https://github.com/user-attachments/assets/4f1d533e-5571-4691-acd9-3b31bc489f13" />


**Corner 2: SS, -40°C, 1.35 V**

<img width="760" height="892" alt="Screenshot 2026-11-01 120731" src="https://github.com/user-attachments/assets/4a2274fe-ce59-4142-9a2a-e3d8a8ca335d" />


<img width="646" height="870" alt="Screenshot 2026-11-01 120743" src="https://github.com/user-attachments/assets/152ede60-11e4-45e7-9223-f09967b5aa8d" />


**Corner 3: SS, -40°C, 1.44 V**

<img width="551" height="695" alt="Screenshot 2026-11-01 120757" src="https://github.com/user-attachments/assets/8d8253ee-bf13-4c97-a483-84c2419fc819" />


<img width="677" height="860" alt="Screenshot 2026-11-01 120811" src="https://github.com/user-attachments/assets/a98f112a-4913-436b-9984-2892dbbecf9f" />


**Corner 4: SS, 100°C, 1.40 V**
<img width="846" height="781" alt="Screenshot 2026-11-01 120824" src="https://github.com/user-attachments/assets/1564d4a1-85c0-470a-875b-aa52f8120021" />

<img width="802" height="847" alt="Screenshot 2026-11-01 120836" src="https://github.com/user-attachments/assets/b138c99a-98f4-4d26-b925-934d68ce8a1f" />


### 7.4 WNS and TNS Graphs (VM)

The WNS graph helps identify the worst reported slack across the evaluated corners. The TNS graph shows the accumulated negative slack of violating paths.

**VM WNS graph**

<img width="1843" height="718" alt="Screenshot 2026-11-01 120853" src="https://github.com/user-attachments/assets/04cfe672-82ab-47cb-ab76-09ad03975a8d" />


**VM TNS graph**

<img width="1860" height="753" alt="Screenshot 2026-11-01 120906" src="https://github.com/user-attachments/assets/5c18c29f-b826-4b81-98cd-2d4411067bb9" />



### 7.5 Findings

- The VM analysis examines the synthesized design before routing.
- The reported timing results vary across the selected timing libraries.
- The `ss_n40C_1v28` corner shows substantial negative WNS and TNS.
- WNS and TNS provide complementary information about timing violations.
- These results belong to the VM pre-route analysis and must be distinguished from the Codespace post-route results.

---

## 8️⃣ SKY130 Timing Libraries and PVT Corners

### 8.1 Background

Timing libraries provide timing and electrical models for standard cells, allowing tools to estimate cell delays under specified operating conditions. PVT stands for:

- **Process:** manufacturing-process variation.
- **Voltage:** supply-voltage condition.
- **Temperature:** operating-temperature condition.

Different PVT conditions can produce different cell delays, so analysing multiple corners helps evaluate timing behaviour across operating conditions.

### 8.2 SKY130 HD Library Set

The SKY130 HD timing-library directory was inspected in the VM. The library filenames identify their process corner, temperature, and supply-voltage conditions.

The libraries were used as inputs to the timing analysis. The GitHub repository documents the library inspection and analysis results; the complete timing-library files do not need to be uploaded.

### 8.3 Process Corner Types

| Corner | Description |
|---|---|
| FF | Fast-Fast |
| SS | Slow-Slow |
| TT | Typical-Typical |

The VM analysis evaluated 16 corners: 6 FF, 8 SS, and 2 TT.

---

## 9️⃣ Multi-Corner STA in Codespace

### 9.1 Scope

A separate multi-corner Static Timing Analysis was performed in Codespace using the VSDBabySoC physical-design environment. Unlike the VM analysis, the Codespace report evaluates the **routed** design using OpenROAD and the associated timing constraints, covering minimum- and maximum-delay timing behaviour across 16 PVT corners.

The Codespace screenshots are stored in `images_codespace/`. Their filenames use `rpt(1)` and `rpt(2)` for the two parts of each corner report.

### 9.2 Setup for the Analysis

The Codespace report documents the following files and directories:

| Item | Path |
|---|---|
| Routed design database | `results/sky130hd/vsdbabysoc/base/5_route.odb` |
| Timing constraints | `results/sky130hd/vsdbabysoc/base/5_route.sdc` |
| Standard-cell timing libraries | `designs/sky130hd/vsdbabysoc/libs/timing/` |
| PLL timing library | `designs/sky130hd/vsdbabysoc/lib/avsdpll.lib` |
| DAC timing library | `designs/sky130hd/vsdbabysoc/lib/avsddac.lib` |
| Report directory | `reports/sta_minmax_16/` |
| Summary CSV | `reports/sta_minmax_16/minmax_slack_summary.csv` |
| TCL script | `/tmp/sta_minmax.tcl` |

These paths describe the Codespace analysis environment.

### 9.3 TCL Script Used in Codespace

**Additional STA script**
<img width="1832" height="512" alt="Screenshot 2026-11-01 120923" src="https://github.com/user-attachments/assets/fd8d22b8-0549-44e1-bd47-b7bc0876e0f0" />



### 9.4 Commands for Timing Reports

The following commands were used to inspect timing paths and report slack metrics:

```tcl
report_checks -path_delay min_max -fields {nets cap slew input_pins fanout}

report_wns -min
report_wns -max

report_tns -min
report_tns -max
```

Minimum-delay and maximum-delay checks evaluate different timing requirements and must be interpreted separately.

### 9.5 PVT Corners Covered

| Corner group | Number of corners |
|---|---:|
| FF | 6 |
| SS | 8 |
| TT | 2 |
| **Total** | **16** |

### 9.6 Timing Reports for Four Selected Corners

The post-route report recorded maximum-delay violations in five SS corners. The following four reports show representative results for the corners with the most negative reported WNS values.

**Corner 1: SS, -40°C, 1.28 V**

| Metric | Value |
|---|---:|
| WNS | -20.83 ns |
| TNS | -5815.72 ns |

<img width="1280" height="727" alt="Screenshot 2026-11-01 120936" src="https://github.com/user-attachments/assets/62f59030-b1c7-40da-87fd-5458d9d917c1" />


<img width="1592" height="887" alt="Screenshot 2026-11-01 120952" src="https://github.com/user-attachments/assets/5e94222c-fd28-4382-983a-501ede4689a8" />


**Corner 2: SS, -40°C, 1.35 V**

| Metric | Value |
|---|---:|
| WNS | -10.66 ns |
| TNS | -2019.57 ns |

<img width="1612" height="870" alt="Screenshot 2026-11-01 121008" src="https://github.com/user-attachments/assets/73310740-d43b-4971-8a25-1f0ee8636f4d" />


<img width="1692" height="753" alt="Screenshot 2026-11-01 121024" src="https://github.com/user-attachments/assets/67911f75-b628-443f-b59b-b9b3257c6ad5" />



**Corner 3: SS, -40°C, 1.40 V**

| Metric | Value |
|---|---:|
| WNS | -6.36 ns |
| TNS | -762.02 ns |

<img width="1616" height="815" alt="Screenshot 2026-11-01 121043" src="https://github.com/user-attachments/assets/89324daa-471d-4b4d-b88a-a587a91c6678" />


<img width="1647" height="781" alt="Screenshot 2026-11-01 121057" src="https://github.com/user-attachments/assets/3d446c61-2550-4991-877a-9107f8feb1c7" />


**Corner 4: SS, -40°C, 1.44 V**

| Metric | Value |
|---|---:|
| WNS | -3.82 ns |
| TNS | -168.54 ns |

<img width="1581" height="817" alt="Screenshot 2026-11-01 121115" src="https://github.com/user-attachments/assets/9baba8bd-8ccd-4806-9d67-3edf2e4c9061" />


<img width="1520" height="831" alt="Screenshot 2026-11-01 121128" src="https://github.com/user-attachments/assets/25db091f-9b06-4782-8821-312d993c4d45" />



### 9.7 Overall Summary Report

The summary report provides an overview of the multi-corner timing analysis.

<img width="1232" height="422" alt="Screenshot 2026-11-01 122650" src="https://github.com/user-attachments/assets/96062d1c-0db2-4cff-8258-003212e7b976" />


### 9.8 WNS and TNS Graphs (Codespace)

**Codespace WNS graph**
<img width="1243" height="353" alt="Screenshot 2026-11-01 122700" src="https://github.com/user-attachments/assets/93be71fd-17b0-447b-a0a0-0b84805793e0" />


**Codespace TNS graph**
<img width="1257" height="498" alt="Screenshot 2026-11-01 122712" src="https://github.com/user-attachments/assets/2f038eb6-00c4-418b-b8b4-030005810fd8" />



### 9.9 Findings

- The Codespace analysis evaluates the routed VSDBabySoC design.
- Five SS corners showed maximum-delay timing violations.
- The `ss_n40C_1v28` corner produced the most negative reported WNS.
- WNS indicates the worst reported slack, while TNS represents accumulated negative slack.
- The VM and Codespace results represent different analysis stages and should not be treated as interchangeable.
- The full set of corner reports is retained in the repository as supporting evidence.

---

## 🔟 OpenROAD Physical Design Flow

### 10.1 Background

The OpenROAD physical-design flow was explored using VSDBabySoC and the SKY130 HD platform. The work includes synthesis, floorplanning, placement, and routing. The design configuration used was:

```bash
./designs/sky130hd/vsdbabysoc/config.mk
```

### 10.2 Synthesis Step

Synthesis generates a gate-level implementation from the RTL using the selected technology libraries.

```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk synth
```

### 10.3 Floorplan Stage

Floorplanning establishes the initial physical organisation of the design.

```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk floorplan
```

The `gui_floorplan` results were inspected to view the floorplan layout and its associated timing information.

<img width="1195" height="642" alt="Screenshot 2026-11-01 122730" src="https://github.com/user-attachments/assets/3aeeac7a-fb3f-4956-88cb-d987c92e1c6d" />

<img width="1218" height="587" alt="Screenshot 2026-11-01 122743" src="https://github.com/user-attachments/assets/183373bb-a479-4d86-b8be-044e32c17c18" />



### 10.4 Placement Stage

Placement assigns physical locations to standard cells within the placement region.

```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk place
```

The `gui_placement` results were inspected to view the placement layout and timing information.

<img width="1242" height="542" alt="Screenshot 2026-11-01 122757" src="https://github.com/user-attachments/assets/58554350-d6b8-4e33-85d8-f959e14987cc" />


<img width="1240" height="472" alt="Screenshot 2026-11-01 122807" src="https://github.com/user-attachments/assets/e6e6ce80-be2e-4cb4-8302-4ff314f24ed9" />


### 10.5 Routing Stage

Routing connects the placed standard cells through the available metal layers.

```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk route
```

The `gui_routing` results were inspected to view the routed layout and associated timing information.

<img width="1248" height="618" alt="Screenshot 2026-11-01 122824" src="https://github.com/user-attachments/assets/60432ba6-a4f9-4f03-8e42-6a010200dd1b" />

<img width="1238" height="661" alt="Screenshot 2026-11-01 122836" src="https://github.com/user-attachments/assets/d119e2ed-4341-47e6-80ac-f55d613171df" />


### 10.6 Findings

- `gui_floorplan` provides a view of the floorplanned design.
- `gui_placement` provides a view of the placed standard cells.
- `gui_routing` provides a view of the routed design.
- The GUI views help connect the physical layout with timing information reported during the flow.
- The routed design can be used for post-route timing analysis.

---

## 1️⃣1️⃣ Key Learnings

The flow studied across this work:

```text
RTL Coding
     ↓
RTL Simulation
     ↓
Functional Verification
     ↓
Synthesis
     ↓
Logic Optimization
     ↓
Sequential Cell Mapping
     ↓
Technology Mapping
     ↓
Gate-Level Netlist
     ↓
Gate-Level Simulation
     ↓
Functional Comparison
     ↓
Static Timing Analysis
     ↓
Physical Design (Floorplan → Placement → Routing)
```

- ✅ Connected RTL coding style to the hardware synthesis infers, using the MUX experiments, and extended that understanding to a larger hierarchical design with VSDBabySoC.
- ✅ Understood how optimization, technology mapping, and the choice of technology library change a gate-level netlist relative to its RTL.
- ✅ Used static timing analysis with OpenSTA, including TCL scripts and timing commands, to inspect timing paths.
- ✅ Applied the timing metrics — arrival time, required time, slack, WNS, and TNS — to real reports.
- ✅ Evaluated timing across 16 PVT corners (6 FF, 8 SS, 2 TT) using the SKY130 HD timing libraries.
- ✅ Distinguished VM pre-route results from Codespace post-route results, rather than treating them as interchangeable.
- ✅ Explored the OpenROAD flow — synthesis, floorplanning, placement, and routing — through its GUI views.
- ✅ Identified the operating conditions under which maximum-delay requirements were not met.

The VM report showed substantial negative slack in the `ss_n40C_1v28` pre-route analysis. The Codespace post-route report also identified five SS corners with maximum-delay violations, with `ss_n40C_1v28.lib` producing the most negative reported WNS. These results show why timing has to be evaluated across multiple PVT conditions and interpreted according to the implementation stage.

More broadly, RTL design is not only about writing code that produces the expected simulation output. It also means understanding how synthesis interprets the RTL, how optimization changes the implementation, how technology libraries influence the mapped hardware, and how timing constraints, standard-cell libraries, PVT conditions, and physical implementation together determine the timing behaviour of a digital circuit.

---

## 👤 About the Author

**Jahnavi Sri Bhavya**
Department of Electronics and Communication Engineering (ECE)
Anurag University
