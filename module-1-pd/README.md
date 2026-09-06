# 🔧 Module 1 — Introduction to Open-Source EDA, OpenLANE, and SKY130 PDK

<p>
  <img src="https://img.shields.io/badge/Tool-OpenLANE-blue" alt="OpenLANE">
  <img src="https://img.shields.io/badge/Tool-OpenROAD-purple" alt="OpenROAD">
  <img src="https://img.shields.io/badge/Tool-Magic-orange" alt="Magic">
  <img src="https://img.shields.io/badge/PDK-SKY130-red" alt="SKY130">
  <img src="https://img.shields.io/badge/Flow-RTL--to--GDS-green" alt="RTL to GDS">
</p>

> Part of the Chip Design Program — Physical Design series.

## 📖 Module Overview

This module covers the background needed before touching any physical design tool: how software ultimately becomes hardware, what an open-source ASIC design flow actually consists of, what a PDK is and why it exists, and a first look at OpenLANE as the automated flow that ties RTL, EDA tools, and the SKY130 PDK together into a finished GDSII layout.

| | |
|---|---|
| 🛠️ **Tools covered** | OpenLANE, OpenROAD, Yosys, OpenSTA, Magic, Netgen, Fault |
| 🧩 **PDK** | SKY130 (Google + SkyWater, open-source 130nm) |
| 📋 **Prerequisites** | Basic familiarity with digital logic and Linux terminal |

## 📑 Contents

- 1. From Software Concepts to Hardware
- 2. Fundamentals of Open-Source ASIC Design
- 3. Understanding the PDK
- 4. SKY130 — The Open-Source Process Design Kit
- 5. Overview of the EDA Toolchain
- 6. RTL-to-GDSII Design Flow
- 7. Getting Started with OpenLANE
  - 7.1 Inside the OpenLANE Flow
  - 7.2 Design-for-Test (DFT) Integration
  - 7.3 OpenROAD — Automated Physical Implementation
  - 7.4 Managing Antenna Rule Violations
  - 7.5 Exploring the Design Space
- 8. Lab: Inspecting the OpenLANE PDK Directory
- 9. Lab: Configuring the OpenLANE Environment
- 10. Lab: Executing the OpenLANE Flow with picorv32a
- 11. Key Learnings
- About the Author

---

## 1️⃣ From Software Concepts to Hardware

Before getting into ASIC-specific tools, it helps to place chip design inside the bigger picture of how any program eventually runs on hardware. Application software and system software both eventually reduce to instructions a compiler and assembler turn into binary — and that binary only means something because a specific piece of hardware was built to understand it.

<img width="1251" height="737" alt="Screenshot 2026-09-07 001225" src="https://github.com/user-attachments/assets/4ee7d5a7-819d-4627-b2e3-08302d31d1ba" />


The same idea applies directly to RISC-V: a C program is cross-compiled and assembled into RISC-V machine code, and that machine code only runs correctly because a specific RTL implementation (like `picorv32`) was built, synthesized, and laid out in silicon to execute exactly that instruction set.

<img width="1920" height="1080" alt="Screenshot 2026-09-06 233223" src="https://github.com/user-attachments/assets/18e759a6-9275-4863-bcc3-693647f70cf5" />


Zooming into a single instruction makes the chain explicit: an instruction like `add x6, x10, x6` is defined by the Instruction Set Architecture (the "architecture" of the computer), assembled into binary, and that same binary can be traced forward into a synthesized gate-level netlist and finally a physical layout that implements exactly that operation.

<img width="1920" height="1080" alt="Screenshot 2026-09-06 233330" src="https://github.com/user-attachments/assets/c59c50cb-9282-4c3f-8d08-b9eb038e5f94" />


---

## 2️⃣ Fundamentals of Open-Source ASIC Design

An ASIC comes together from three ingredients: **RTL designs** (the logic itself, often sourced from places like librecores.org, opencores.org, or GitHub), **EDA tools** (Qflow, OpenROAD, OpenLANE) that turn that RTL into a manufacturable layout, and **PDK data** describing the target fabrication process.

<img width="1920" height="1080" alt="Screenshot 2026-09-06 233417" src="https://github.com/user-attachments/assets/6a1645f3-d355-4cd7-958e-599683d4e68c" />


---

## 3️⃣ Understanding the PDK

In the early era of chip design, IC design was tightly coupled to whatever manufacturing process a given company had access to — whoever controlled the physics controlled the creative agenda. Lynn Conway and Carver Mead changed this by pioneering a **structured design methodology** based on λ-based design rules, which separated *design* from *technology* for the first time. That separation is what eventually made **Pure Play Fabs** (companies that only manufacture) and **Fabless design companies** (companies that only design) possible as distinct business models.

<img width="1920" height="1080" alt="Screenshot 2026-09-06 233929" src="https://github.com/user-attachments/assets/1cd22d8e-1d25-4c6f-ae42-5e0bc05f83d4" />


A **Process Design Kit (PDK)** is the practical result of that separation — a collection of files that models a specific fabrication process for the EDA tools used to design an IC. It typically includes:

- Process design rules (DRC, LVS, PEX)
- Device models
- Digital standard-cell libraries
- I/O libraries


---

## 4️⃣ SKY130 — The Open-Source Process Design Kit

SKY130 is the PDK used throughout this program — a 130nm process, released as a fully open-source, production-grade PDK through a collaboration between **Google** and **SkyWater Technology**.

<img width="1920" height="1080" alt="Screenshot 2026-09-06 234007" src="https://github.com/user-attachments/assets/7ab2eee4-a5fa-4625-b4e9-f62286c63cb1" />


Its openness is what makes the rest of this program possible without any proprietary licensing — the same PDK data referenced by OpenLANE here is publicly available at `github.com/google/skywater-pdk`.

---

## 5️⃣ Overview of the EDA Toolchain

Turning RTL into a working chip involves far more individual steps than "synthesis" and "place and route" alone suggest. Some of the stages an EDA toolchain has to cover:

<img width="1920" height="1080" alt="Screenshot 2026-09-06 234113" src="https://github.com/user-attachments/assets/a7f4fc3c-4b96-428c-96b7-aab9d3faf30e" />


HDL simulation, HDL design entry, RTL synthesis, logic synthesis, floor planning, power planning, placement (global and detailed), clock tree synthesis, routing (global and detailed), RC extraction, static timing analysis, DRC, LVS, DFM, DFT, IR drop analysis, and static code/logic equivalence checking all have to happen — usually each backed by its own specialized tool.

---

## 6️⃣ RTL-to-GDSII Design Flow

At a high level, all of that reduces to one pipeline: RTL and PDK data go in, and a **GDSII** file — the manufacturable layout — comes out, passing through synthesis, floorplanning/power planning, placement, clock tree synthesis, routing, and sign-off along the way.

<img width="1920" height="1080" alt="Screenshot 2026-09-06 234145" src="https://github.com/user-attachments/assets/17831a65-2d68-4347-b4da-ea2ab3205bb6" />


**Synthesis (Synth)**

*Definition:* translating the behavioral RTL description into a gate-level netlist built entirely from cells available in the target standard-cell library.

*Why it's needed:* RTL describes *what* the circuit should do, not the actual hardware that does it — synthesis is the step that commits to real, physically-implementable logic gates, which is the only thing later stages (placement, routing) know how to work with.
 
**Floorplanning and Power Planning (FP+PP)**
 
*Definition:* deciding the chip's physical die dimensions, where major blocks and macros sit within that area, and building out the power distribution network (power rings, straps, and rails) that will supply every cell.

*Why it's needed:* every later stage depends on a fixed floorplan — placement can't run without knowing the available area, and cells can't function at all without a power network already in place to connect to. Getting this step wrong (too little area, poor power distribution) creates problems that are expensive or impossible to fix later.
 
**Placement (Place)**
 
*Definition:* assigning every standard cell in the netlist a specific physical (x, y) location within the floorplan, typically done in two passes — global placement (approximate positions optimizing overall wirelength) followed by detailed placement (legalizing those positions onto the actual placement grid/rows).

*Why it's needed:* routing later needs real cell coordinates to connect, and how well cells are placed relative to each other directly determines how short (or long, and therefore slow) the wires connecting them will need to be.
 
**Clock Tree Synthesis (CTS)**
 
*Definition:* building a dedicated network of buffers/inverters that distributes the clock signal from its source to every sequential element (flip-flop) in the design.

*Why it's needed:* a clock signal reaching different flip-flops at different times (clock skew) can break timing and cause functional failures — CTS exists specifically to deliver the clock to everywhere it's needed with controlled, minimal skew, which a naive direct wire from the clock pin could never guarantee at scale.
 
**Routing (Route)**
 
*Definition:* drawing the actual physical metal wires that implement every electrical connection in the netlist, again typically in two passes — global routing (coarse-grained paths through routing regions) followed by detailed routing (exact metal-layer, track-level wire geometry).

*Why it's needed:* placement only fixes *where* cells are — routing is what actually connects them electrically. This step also has to respect the PDK's manufacturing design rules (spacing, width, via rules) so the resulting layout can actually be fabricated correctly.
 
**Sign-Off**
 
*Definition:* a final battery of verification checks on the completed layout — including Design Rule Checking (DRC), Layout-vs-Schematic (LVS), Static Timing Analysis (STA), and parasitic extraction (PEX) — confirming the layout is both manufacturable and functionally/timing-correct before it's released.

*Why it's needed:* this is the last checkpoint before a design becomes an unchangeable, expensive-to-fix physical mask set — sign-off exists to catch anything earlier stages might have missed (a stray DRC violation, a timing path that slipped through) while it's still fixable in software rather than silicon.
 
---

## 7️⃣ Getting Started with OpenLANE

OpenLANE is an automated, open-source RTL-to-GDSII flow built specifically around the tools and PDK introduced above — it's the tool this program actually uses to walk that pipeline end-to-end.

### 7.1 Inside the OpenLANE Flow

Underneath the simplified picture, OpenLANE chains together a specific set of tools for each stage: **Yosys + abc** for RTL synthesis, **OpenSTA** for static timing analysis, **Fault** for DFT, an **OpenROAD** application block handling floorplanning/placement/CTS/optimization/global routing, **TritonRoute** for detailed routing, and **Magic + Netgen** for physical verification and GDSII streaming — with a logic equivalence check (LEC) and a design-exploration loop feeding back into synthesis if results aren't good enough.

<img width="1920" height="1080" alt="Screenshot 2026-09-06 234240" src="https://github.com/user-attachments/assets/03bd7160-253c-4838-978b-1a1acb4394f6" />


### 7.2 Design-for-Test (DFT) Integration

Before physical implementation even begins, OpenLANE (via **Fault**) inserts test infrastructure into the design — scan insertion, automatic test pattern generation (ATPG), test pattern compaction, fault coverage, and fault simulation — so that manufactured chips can later be tested for defects.

<img width="1920" height="1080" alt="Screenshot 2026-09-06 234305" src="https://github.com/user-attachments/assets/ad8b0665-bb97-4a53-bcd4-0dcb867b7195" />



### 7.3 OpenROAD — Automated Physical Implementation

OpenROAD handles what's often just called automated PnR (Place and Route): floor/power planning, end decoupling capacitor and tap cell insertion, global and detailed placement, post-placement optimization, clock tree synthesis, and global and detailed routing.

<img width="1920" height="1080" alt="Screenshot 2026-09-06 234327" src="https://github.com/user-attachments/assets/ebec3cb0-eaa2-40df-b1c6-714f1876197b" />



### 7.4 Managing Antenna Rule Violations

A subtle fabrication issue: a long metal wire segment can act as an antenna during manufacturing. Reactive ion etching causes charge to accumulate on the wire, and that accumulated charge can damage the transistor gate it eventually connects to before the rest of the circuit exists to safely discharge it.

<img width="1815" height="892" alt="Screenshot 2026-09-06 122405" src="https://github.com/user-attachments/assets/c3d8d29d-d06c-42d0-adf9-57d5eefaf9e0" />

One fix is **bridging** — routing part of the net up to a higher metal layer and back down, which breaks the charge-accumulating path — though this requires router awareness that wasn't fully available at the time.

OpenLANE instead takes a **preventive approach**: a fake antenna diode is added next to every cell input right after placement. The Antenna Checker (Magic) then runs on the routed layout, and only where it actually reports a violation does the fake diode get swapped for a real one — avoiding the cost of adding real diodes everywhere "just in case."

### 7.5 Exploring the Design Space

Because so many flow parameters can be tuned, OpenLANE includes a **Design Space Exploration** utility to search for the best set of flow configurations for a given design, rather than requiring that tuning to be done by hand.


It ships with a large number of reference examples to start from — 43 designs with known-good configurations at the time of this session, with more added over time.

---

## 8️⃣ Lab: Inspecting the OpenLANE PDK Directory

With the concepts in place, the actual OpenLANE working directory was explored to see how the SKY130 PDK data is laid out on disk:

```bash
cd ~/Desktop/work/tools/openlane_working_dir
ls
cd pdks
ls
cd sky130A
ls
cd libs.ref
ls -ltr
```

<img width="1657" height="991" alt="Screenshot 2026-09-06 154016" src="https://github.com/user-attachments/assets/f8aa8506-55c3-4f05-9385-4e6fcd50d734" />

`libs.ref` contains every SKY130 standard-cell flavor available — `sky130_fd_sc_hd` (high density, the one used throughout this workshop), along with `_hs`, `_ms`, `_ls`, `_hdll`, `_hvl`, `_lp` variants, plus `sky130_fd_io`, `sky130_sram_macros`, and `sky130_fd_pr`. Moving into `libs.tech` instead reveals the actual tool installations bundled with the PDK setup — `magic`, `netgen`, `klayout`, `ngspice`, `qflow`, `openlane`, `irsim`, `xschem`, and `xcircuit`.

---

## 9️⃣ Lab: Configuring the OpenLANE Environment

```bash
cd ..
docker run -it -v $(pwd):/openLANE_flow -v $PDK_ROOT:$PDK_ROOT -e PDK_ROOT=$PDK_ROOT -u $(id -u $USER):$(id -g $USER) efabless/openlane:v0.21
```

<img width="1917" height="1027" alt="Screenshot 2026-09-06 155706" src="https://github.com/user-attachments/assets/caefcc4c-37ee-48b2-b04b-a7008c4f5c48" />

An existing shell alias (`docker` aliased to a `docker run ...` command) was also interfering with plain Docker commands like `docker images` — `unalias docker` was needed before the container could even be inspected properly. Once inside the container, `pwd` and `ls -ltr` confirmed the OpenLANE flow's actual top-level structure: `flow.tcl`, `run_designs.py`, the `designs/`, `scripts/`, `docker_build/`, and `configuration/` directories, and the top-level `README.md`.

---

## 🔟 Lab: Executing the OpenLANE Flow with picorv32a

With OpenLANE actually running, the next step was walking a real design — `picorv32a` — through the flow interactively, rather than just reading about each stage.

```bash
cd ~/Desktop/work/tools/openlane_working_dir/openlane
./flow.tcl -interactive
package require openlane 0.9
prep -design picorv32a
```
<img width="1917" height="1008" alt="Screenshot 2026-09-06 165639" src="https://github.com/user-attachments/assets/ca5ebd59-0d0d-47b1-89a2-b9d3997d0bb6" />


`prep` sets up a timestamped run directory (`designs/picorv32a/runs/06-09_11-26` in this run) and merges the SKY130 LEF files, filler/tap/decap cell definitions, and clock buffer variants (`sky130_fd_sc_hd__clkbuf_1/2/4/8/16`) needed for the rest of the flow. It also loads the design's own config alongside per-library configs (`sky130A_sky130_fd_sc_hd_config.tcl` and its `_ms`/`_ls`/`_hs`/`_hdll` siblings) and the shared `floorplan.tcl`/`synthesis.tcl`/`cts.tcl`/`routing.tcl`/`placement.tcl` configuration files.

Exploring the resulting run directory:

```bash
cd designs/picorv32a
ls -ltr
cd src
ls -ltr
cd ../runs
ls -ltr
cd 06-09_11-26
ls -ltr
```
<img width="1917" height="1028" alt="Screenshot 2026-09-06 165946" src="https://github.com/user-attachments/assets/6e626abb-97d3-419d-ac6b-02d1319abbbd" />
<img width="1917" height="1018" alt="Screenshot 2026-09-06 170058" src="https://github.com/user-attachments/assets/e4f2a587-bbbf-4696-a569-c02e925cbede" />


This reveals the standard OpenLANE run layout: `PDK_SOURCES`, `tmp/`, `results/`, `reports/`, `OPENLANE_VERSION`, `cmds.log`, `logs/`, and `config.tcl` — every stage's output and logs are organized under this one timestamped folder.

Running synthesis:

```bash
run_synthesis
```
<img width="1917" height="996" alt="Screenshot 2026-09-06 170258" src="https://github.com/user-attachments/assets/974df28f-3a6a-4436-ae56-178d089c189b" />


Synthesis invokes Yosys, which prints its GPL license banner and default corner libraries (`sky130_fd_sc_hd__ff_n40C_1v95.lib`, `sky130_fd_sc_hd__ss_100C_1v60.lib`) before generating SDC-style timing constraints from the environment — clock port and period, I/O delay values as a percentage of the clock period, a virtual clock for unclocked inputs, input/output delays, clock uncertainty, clock transition, and propagated clock settings. The run completed in about 13 seconds of user time with a 96.52 MB peak memory footprint, producing `results/synthesis/picorv32a.synthesis.v` — the synthesized gate-level netlist, whose module port list (`mem_instr`, `mem_ready`, `mem_addr`, `mem_wdata`, ... `pcpi_ready`, `irq`, `trace_valid`, `trace_data`) matches picorv32's actual memory and interrupt interface.

One warning surfaced during this run — a net reported as having **no driver** — a reminder that even a successful synthesis run is worth reading through for warnings, not just checking that it didn't error out.

Static timing analysis on the synthesized design surfaced a real critical path report, tracing a signal from a flip-flop (`sky130_fd_sc_hd__dfxtp_2`) forward through a chain of combinational cells — several `or2`/`or3`/`or4` gates, an `o22ai`, an `a221o`, and an `o2111a` — before reaching its endpoint, with the ideal clock network delay and each gate's pin-to-pin delay itemized along the way.

<img width="1917" height="1028" alt="Screenshot 2026-09-06 170740" src="https://github.com/user-attachments/assets/94d64486-fe54-4063-9ffb-89fe60ca709d" />

**Takeaway from this lab specifically:** running a real design through the flow interactively — rather than as one opaque batch command — makes each stage's actual output legible: the merged PDK data `prep` assembles, the constraints synthesis derives from the config, and the specific gates STA reports on the critical path.

---

## 1️⃣1️⃣ Key Learnings

- ✅ Traced the full chain from application software down to physical hardware, using RISC-V as a concrete example.
- ✅ Understood the three ingredients of an open-source ASIC flow: RTL designs, EDA tools, and PDK data.
- ✅ Learned what a PDK actually is, and the historical shift (Conway/Mead) that made separating design from fabrication possible in the first place.
- ✅ Identified SKY130 as an open, production-grade PDK from Google and SkyWater.
- ✅ Surveyed the breadth of the EDA tools landscape and how it collapses into a simplified RTL-to-GDSII flow.
- ✅ Walked through OpenLANE's detailed internal flow, including DFT, OpenROAD-based PnR, antenna violation handling, and design space exploration.
- ✅ Explored the actual SKY130 PDK directory structure and the tools bundled alongside it.
- ✅ Hit and diagnosed real Docker setup issues (disk space, permissions, a conflicting alias) when first bringing up OpenLANE.
- ✅ Ran `picorv32a` through `prep` and `run_synthesis` interactively, and read a real post-synthesis STA critical path report gate by gate.

---

## 👤 About the Author

**JAHNAVI SRI BHAVYA**
Department of Electronics and Communication Engineering (ECE)
Anurag University
