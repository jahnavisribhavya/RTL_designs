# 🟨 Module-2 — Physical Design: Floorplanning, Cell Placement, Power Distribution, and Clock Tree Synthesis

<p>
  <img src="https://img.shields.io/badge/Flow-OpenLane-purple" alt="OpenLane">
  <img src="https://img.shields.io/badge/Tool-Magic-lightgrey" alt="Magic">
  <img src="https://img.shields.io/badge/Tool-OpenROAD-blueviolet" alt="OpenROAD">
  <img src="https://img.shields.io/badge/PDK-SKY130-red" alt="SKY130">
  <img src="https://img.shields.io/badge/Design-picorv32a-9cf" alt="picorv32a">
</p>

## 📖 Module Overview

This module moves from synthesized gate-level netlists into the **physical design** stages of the ASIC flow: taking a netlist and turning it into a manufacturable layout. It covers the theory behind chip floorplanning (core/die, utilization, preplaced cells, power planning, decoupling capacitors, pin placement), the placement and optimization steps that follow, the basics of library characterization (NLDM/CCS timing) that make timing-driven placement possible, an introduction to clock tree synthesis, and — one level below all of it — how the standard cells themselves are designed, from Euler's paths and stick diagrams down to SPICE-level characterization. A hands-on run of the OpenLane flow on the `picorv32a` design ties the theory to real floorplan, placement, and DRC output.

| | |
|---|---|
| 🛠️ **Tools used** | OpenLane, OpenROAD, Magic, Yosys |
| 🧩 **Example design** | `picorv32a` (RISC-V core) |
| 📋 **Prerequisites** | Module-1 (synthesis flow), basic familiarity with standard-cell libraries |

## 📑 Contents

- 1. Fundamentals of Chip Floorplanning
  - 1.1 Understanding Core and Die
  - 1.2 Core Utilization and Aspect Ratio
  - 1.3 Fixed and Preplaced Cells
  - 1.4 Decoupling Capacitors and Power Distribution
  - 1.5 Input and Output Pin Placement
  - 1.6 Placement Blockages
- 2. Standard Cell Placement
  - 2.1 Initial Global Placement
  - 2.2 Detailed and Optimized Placement
- 3. Library Characterization and Clock Network Synthesis
  - 3.1 NLDM, CCS, and Cell Characterization
  - 3.2 Clock Tree Synthesis (CTS)
- 4. Standard Cell Design — Building a Cell from the Ground Up
  - 4.1 Design Inputs and Requirements
  - 4.2 Circuit Implementation and Characterization
  - 4.3 Layout Using Euler Paths and Stick Diagrams
  - 4.4 Layout Verification: DRC, LVS, and Parasitics
- 5. Lab: Performing Floorplanning and Placement for picorv32a
- 6. Key Learnings

---

## 1️⃣ Fundamentals of Chip Floorplanning

### 1.1 Understanding Core and Die

<img width="728" height="627" alt="Screenshot 2026-09-06 234638" src="https://github.com/user-attachments/assets/8e537373-361d-4f3b-a9ed-5f09a6581d99" />


Every chip is fabricated as one of many identical rectangles stepped across a **silicon wafer**. Each of those rectangles is the **die** — the full physical footprint of the chip, including scribe lines and I/O pads. Inside the die sits the **core** — the region where all logical cells (flip-flops, gates, etc.) are actually placed and routed. The gap between core and die boundary is reserved for I/O pads, ESD structures, and (as covered below) the power distribution network.

### 1.2 Core Utilization and Aspect Ratio

<img width="1920" height="1080" alt="Screenshot 2026-09-06 234709" src="https://github.com/user-attachments/assets/772a20a6-f30c-4114-a830-974afc12735c" />



Before anything is placed, floorplanning must fix the width and height of the core and die. Two derived quantities describe how "full" and how "square" that area is:

- **Utilization Factor** = (Area occupied by logical cells) / (Total area of the core). A utilization factor of 100% means the logic completely fills the core with no room left for routing — in practice this is unrealistic, and real designs target well under 100% so the router has space to work with.
- **Aspect Ratio** = Height / Width. An aspect ratio of 1 gives a square core; values away from 1 give a rectangular one. Both numbers are chosen based on the design's cell count and the physical constraints of the target package.

### 1.3 Fixed and Preplaced Cells

<img width="1920" height="1080" alt="Screenshot 2026-09-06 234756" src="https://github.com/user-attachments/assets/c66b4791-97f8-4c4b-88b5-aa343a1e0d41" />

<img width="1920" height="1080" alt="Screenshot 2026-09-06 234836" src="https://github.com/user-attachments/assets/a70eea61-8922-4985-8104-7047623f135e" />



Not everything in a design is a simple standard cell placed automatically. Larger, pre-designed functional blocks — memories, clock-gating cells, comparators, muxes, or any other hand-crafted/hard IP — are placed at **fixed, known locations** before automatic placement and routing begins. These are called **preplaced cells**, and the process of deciding where these IPs sit within the core is called **floorplanning**. Once their locations are fixed, the automated placement tool places the remaining standard cells around them, treating each preplaced block as a fixed obstacle it must route around.

### 1.4 Decoupling Capacitors and Power Distribution

**Why decoupling capacitors are needed:** Every gate draws current from the shared Vdd/Vss power grid only when it switches. As you move away from the power supply pad, the metal wire supplying Vdd looks less like an ideal voltage source and more like a resistor-inductor (Rdd, Ldd) network in series with the actual supply. When a gate switches, it demands a short burst of current (**switching/peak current, I_peak**) — and drawing that current through a non-zero Rdd/Ldd causes the local voltage to sag momentarily. This is a **voltage droop / ground bounce** problem: shared rails, many switching cells, and non-ideal wire impedance combine to create transient noise on Vdd and Vss.

<img width="1350" height="310" alt="Screenshot 2026-09-06 235440" src="https://github.com/user-attachments/assets/b317ec5d-18c1-4760-b983-dfdae001e9bc" />


<img width="1920" height="1080" alt="Screenshot 2026-09-06 235154" src="https://github.com/user-attachments/assets/6d2ec2d3-7f01-418b-94e1-73a4a45c72c5" />



If enough gates switch simultaneously, the resulting dip or bump in the supply rail can cross into the **undefined region** between the valid logic-'1' threshold (Vih) and logic-'0' threshold (Vil) — a value neither reliably interpreted as a clean high nor a clean low. This is exactly the kind of noise that can flip a bit or cause a false transition.

**Solution — decoupling capacitors (decaps):** A capacitor is placed in parallel with Vdd/Vss right next to the switching cells it protects. When a cell switches, the decap — already charged to Vdd — supplies the burst of current locally instead of forcing it to travel all the way back through the resistive/inductive supply network. The RC/RL network then has time to replenish the decap's charge before the next switching event, smoothing out the local supply.

<img width="1920" height="1080" alt="Screenshot 2026-09-06 235417" src="https://github.com/user-attachments/assets/a1bd025b-332d-4950-b42d-c993c32452f9" />

<img width="1920" height="1080" alt="Screenshot 2026-09-06 235322" src="https://github.com/user-attachments/assets/7fa4906d-4080-4a74-a280-c2b45be1deb6" />



Physically, decaps (labeled DECAP1, DECAP2, DECAP3 in the floorplan) are placed as their own cells around the preplaced blocks, right alongside the other logic.

**Power planning:** Beyond individual decaps, the whole core needs a robust **power distribution network (PDN)** — a mesh of horizontal and vertical Vdd/Vss straps overlaid across the die so every cell has a short, low-resistance path to the supply, no matter where it sits.

<img width="1920" height="1080" alt="Screenshot 2026-09-06 235506" src="https://github.com/user-attachments/assets/83680fd4-3277-4cf4-b835-b672d076f5ec" />



This mesh is what the decaps tie into, and it is also why power planning happens early in floorplanning: routing tracks for both signal and power need to be reserved before cell placement gets dense.

### 1.5 Input and Output Pin Placement

<img width="1920" height="1080" alt="Screenshot 2026-09-06 235617" src="https://github.com/user-attachments/assets/3875e955-c3e3-4024-8d24-67e62d060bcf" />


With preplaced blocks and the power mesh fixed, the input/output pins of the design (`Din1..4`, `Dout1..4`, `Clk1`, `Clk2`, `ClkOut`, etc.) are assigned physical locations around the die/core boundary. Pin placement affects how easily signals can later be routed to and from the core logic, so pins are typically placed close to whichever internal block they connect to most directly.

### 1.6 Placement Blockages

<img width="1920" height="1080" alt="Screenshot 2026-09-06 235635" src="https://github.com/user-attachments/assets/3c4c5402-e13b-4952-996b-5f114c30e19d" />


Finally, a **placement blockage** is marked over the regions already occupied by preplaced cells and the power mesh, telling the automatic placer "don't put any standard cells here." Once pins, preplaced blocks, decaps, and the power mesh are all fixed and blockages are marked, the floorplan is considered ready for the placement and routing step.

---

## 2️⃣ Standard Cell Placement

### 2.1 Initial Global Placement

Once the floorplan (core/die size, preplaced cells, power mesh, pins) is fixed, the **placer** takes every standard cell in the synthesized netlist and assigns it an approximate location within the available rows of the core. This first pass — **global placement** — optimizes primarily for wirelength and cell density, without yet guaranteeing that every cell sits on a legal, non-overlapping row position.

### 2.2 Detailed and Optimized Placement

<img width="1920" height="1080" alt="Screenshot 2026-09-06 235816" src="https://github.com/user-attachments/assets/b36a3d28-b5c2-4f7b-94ab-4473dd46ea4c" />



**Detailed placement** then legalizes that rough result: cells are snapped onto actual placement rows and site grid, overlaps are removed, and the tool re-optimizes locally for timing and wirelength based on estimated interconnect delay. This is also where the timing engine starts to matter — the placer needs to know how much delay and capacitance each net will add once wires are drawn, which is exactly what library characterization (next section) provides.

---

## 3️⃣ Library Characterization and Clock Network Synthesis

### 3.1 NLDM, CCS, and Cell Characterization

<img width="1920" height="1080" alt="Screenshot 2026-09-06 235939" src="https://github.com/user-attachments/assets/3df0c840-ee67-4586-99ef-d2a4c8d66f85" />



Every timing-driven step in the flow — placement, CTS, routing, static timing analysis — depends on knowing, for every cell in the library, how its delay and output slew change with input slew and output load. Two common modeling styles capture this:

- **NLDM (Non-Linear Delay Model)** — represents a cell's delay and output transition time as two-dimensional lookup tables indexed by input transition time and output load capacitance. It's compact and fast to use, and is the traditional basis for `.lib` (Liberty) timing arcs.
- **CCS (Composite Current Source)** — models a cell's output as an equivalent current source rather than a simple RC delay, capturing waveform shape more accurately (especially for noise and crosstalk analysis) at the cost of larger characterization data.

Both are produced by **characterizing** each cell across a sweep of input slews and output loads in SPICE, then distilling the results into the lookup tables a synthesis/STA tool consumes — this is the same kind of `.lib` data referenced when Yosys selected `sky130_fd_sc_hd__mux2_1` in earlier modules; here the emphasis is on where that data comes from and why it's structured as lookup tables rather than closed-form equations.

### 3.2 Clock Tree Synthesis (CTS)


After placement, every flip-flop's clock pin needs to receive the clock signal at (ideally) the same time — minimizing **clock skew** — while also controlling **insertion delay**. Rather than routing one long wire from the clock source to every flip-flop (which would have wildly different delays to near vs. far flip-flops), **Clock Tree Synthesis** builds a balanced tree of buffers/inverters (shown as the triangular buffer symbols feeding each FF1/FF2 block) so that every leaf of the tree — every flip-flop clock pin — sees a comparable delay from the root. Balancing this tree is itself a placement-and-routing-aware problem: CTS runs after initial placement is known, precisely because it needs real physical distances to size and place its buffers correctly.

---

## 4️⃣ Standard Cell Design — Building a Cell from the Ground Up

Everything above treats standard cells (an AND gate, a DFF, a buffer) as fixed, pre-characterized building blocks. This section looks one level deeper: how is a single standard cell itself designed?

<img width="1920" height="1080" alt="Screenshot 2026-09-07 000141" src="https://github.com/user-attachments/assets/8d9e321f-858f-41b8-8231-dc9cc42b717b" />


### 4.1 Design Inputs and Requirements

A cell designer starts from three kinds of inputs, all supplied by the foundry as part of the **Process Design Kit (PDK)**:

- **Process design rules (DRC/LVS rules)** — the geometric constraints a layout must obey to be manufacturable (minimum widths, spacings, extensions) and the rules for verifying the layout matches the intended schematic.
- **SPICE models** — transistor-level models (like the BSIM parameters shown below) used to simulate the cell's electrical behavior before committing to layout.
- **A defined cell specification** — the logic function, input/output pin arrangement, and target drive strength the new cell needs to implement.

<img width="1920" height="1080" alt="Screenshot 2026-09-07 000233" src="https://github.com/user-attachments/assets/15a5587e-b8cd-4469-82a6-b388ed1be2f8" />


### 4.2 Circuit Implementation and Characterization

The designer first builds and simulates the transistor-level circuit (e.g. sizing PMOS/NMOS ratios for a target switching threshold Vm), then, once the circuit works, characterizes it — measuring delay, power, and noise-margin behavior across process/voltage/temperature corners — the same NLDM/CCS characterization discussed above, but here it's the source data being generated rather than consumed.

### 4.3 Layout Using Euler Paths and Stick Diagrams
<img width="1920" height="1080" alt="Screenshot 2026-09-07 000320" src="https://github.com/user-attachments/assets/eb2520f2-aeb0-4a50-9b7f-dca446fe1cf1" />


<img width="1920" height="1080" alt="Screenshot 2026-09-07 000411" src="https://github.com/user-attachments/assets/970b39cf-2c8e-4d7a-ab5b-2941ed050f7f" />



Translating a transistor-level schematic into a compact layout starts with graph theory: the PMOS network and NMOS network of a complex gate (e.g. `(B+D).(A+C)+E.F`) are each represented as a graph, and an **Euler's path** — a path that traverses every edge of the graph exactly once — is found that is *common* to both the PMOS and NMOS graphs. Following that shared Euler's path when placing transistors lets the polysilicon gate strips run in one continuous, non-broken line across the cell, which directly minimizes the diffusion breaks needed and keeps the resulting layout compact. The **stick diagram** is the intermediate abstraction between this graph and the final layout: it shows relative placement and connectivity of poly, diffusion, and metal without yet committing to exact dimensions.

### 4.4 Layout Verification: DRC, LVS, and Parasitics

Layout dimensions are specified in units of **lambda (λ)**, defined as half the minimum feature size (λ = L/2, where L is the process's minimum gate length). Expressing rules this way (e.g. "poly width: 2λ", "poly-to-active spacing: 1λ") lets the same design rules scale automatically if the process's minimum feature size changes. Once a layout is drawn, it must pass:

- **DRC (Design Rule Check)** — verifying every geometric constraint (widths, spacings, extensions) from the PDK's rule deck is satisfied.
- **LVS (Layout vs. Schematic)** — verifying the layout's extracted netlist matches the original transistor-level schematic.

Only after clearing both is the cell considered ready to be added to the standard-cell library that the rest of the flow (synthesis, placement, CTS, routing) will draw from.

---

## 5️⃣ Lab: Performing Floorplanning and Placement for picorv32a

With the theory in place, the same steps were run through OpenLane on the `picorv32a` RISC-V core design.

```bash
cd ~/Desktop/work/tools/openlane_working_dir/openlane
# inside the OpenLane flow shell
```

<img width="1917" height="1018" alt="Screenshot 2026-09-06 172157" src="https://github.com/user-attachments/assets/d14727d4-7928-4a05-b5f2-24a3068cd90f" />

The flow's `floorplan` step reports the die/core geometry it computed (visible as repeated `STEP 460 0 ;` DEF placement-grid entries) before writing out `picorv32a.floorplan.def`:

```bash
cd designs/picorv32a/runs/06-09_11-26/results/floorplan
ls -ltr
less picorv32a.floorplan.def
```

<img width="1917" height="1020" alt="Screenshot 2026-09-06 172538" src="https://github.com/user-attachments/assets/9da89218-3ab4-4e24-8d90-a0b6519a57be" />


Opening the resulting floorplan in **Magic** (with DRC checking enabled) shows the die outline with rows ready for placement:

<img width="1275" height="733" alt="layout" src="https://github.com/user-attachments/assets/f4bc2a99-72e7-47eb-9333-0e832a8a69c5" />
<img width="962" height="668" alt="layout1" src="https://github.com/user-attachments/assets/bdb97531-887d-4602-8051-866bc0b48e14" />


Running the placement stage next produces a denser, gate-level view of the same design once cells have been dropped into rows and legalized:
<img width="1038" height="641" alt="placementmagic" src="https://github.com/user-attachments/assets/792d1a97-87c0-435f-b2d8-51db755e4116" />
<img width="1073" height="703" alt="placement1" src="https://github.com/user-attachments/assets/818d0b0f-8c1c-40b4-a1f0-9d528a5aa8c6" />

---

## 6️⃣ Key Learnings

- ✅ Learned the core/die distinction and how utilization factor and aspect ratio govern how floorplanning sizes the core.
- ✅ Understood preplaced cells (memory, clock-gating cells, comparators, muxes) as fixed obstacles placed before automatic placement, and floorplanning as the process of arranging them.
- ✅ Traced the voltage-droop/ground-bounce problem to non-ideal Rdd/Ldd in the power network, and saw decoupling capacitors and power-mesh planning as the fix.
- ✅ Covered pin placement and placement blockages as the final steps that mark a floorplan ready for placement and routing.
- ✅ Distinguished global placement (rough, wirelength-optimized) from detailed placement (legalized, timing-aware).
- ✅ Learned why NLDM and CCS timing models exist — lookup-table representations of characterized cell delay/slew behavior that timing-driven placement and CTS depend on.
- ✅ Understood Clock Tree Synthesis as building a balanced buffer tree to minimize clock skew across flip-flops.
- ✅ Went one level below the standard-cell abstraction: how a cell itself is designed, from PDK inputs and SPICE characterization through Euler's-path-driven stick diagrams to DRC/LVS-clean layout.
- ✅ Ran the OpenLane flow's floorplan and placement stages on `picorv32a`, inspecting the resulting DEF files and layouts directly in Magic.
## 👤 Author
JAHNAVI SRI BHAVYA
