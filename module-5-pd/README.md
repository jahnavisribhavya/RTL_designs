# 🧵 Module 5 — Routing Design, DRC Checks, Power Networks & Parasitics

<p>
  <img src="https://img.shields.io/badge/Flow-OpenLane%20%2F%20OpenROAD-green" alt="OpenLane / OpenROAD">
  <img src="https://img.shields.io/badge/Routing-TritonRoute-blueviolet" alt="TritonRoute">
  <img src="https://img.shields.io/badge/STA-OpenSTA-orange" alt="OpenSTA">
  <img src="https://img.shields.io/badge/PDK-SKY130-blue" alt="SKY130">
  <img src="https://img.shields.io/badge/Verification-DRC-yellow" alt="DRC">
  <img src="https://img.shields.io/badge/Format-LEF%20%2F%20DEF%20%2F%20SPEF-red" alt="LEF / DEF / SPEF">
  <img src="https://img.shields.io/badge/Layout-Magic-lightgrey" alt="Magic">
</p>

> Part of the SKY130 VLSI Physical Design module series.

## 📖 Overview

This document covers Module 5 of the SKY130 physical-design flow: global and detailed routing, maze routing via Lee's Algorithm, Power Distribution Network (PDN) construction, Design Rule Checking (DRC), TritonRoute's detailed-routing engine, routing connectivity and topology, parasitic extraction/SPEF generation, and the resulting OpenLane output used for post-route timing verification.

|  |  |
|---|---|
| 🛠️ **Tools used** | OpenLane, OpenROAD, TritonRoute, OpenSTA, Yosys, SKY130 PDK, Magic, LEF, DEF, SPEF, Tcl, Docker |
| 🧩 **Example design(s)** | OpenLane-driven SKY130 physical design run (routing, PDN and DRC labs) |
| 📋 **Prerequisites** | OpenLane/OpenROAD environment running in Docker, SKY130 PDK installed, a placed and clock-tree-synthesized design, LEF/DEF files available |

## 📑 Table of Contents

1. Routing Basics — Global, Fast and Detailed Methods
   1.1 Routing Introduction and Main Objectives
   1.2 Comparing Global and Detailed Routing
   1.3 Fast Routing and Guide Creation
2. Maze Routing with Lee's Algorithm
   2.1 Main Algorithm Steps
   2.2 Benefits and Limitations
3. Power Delivery Network (PDN)
   3.1 PDN Purpose and Main Components
   3.2 PDN Setup Flow and Configuration
   3.3 Power Straps and Cell/Macro Connections
4. Design Rule Check (DRC)
   4.1 Key Layout Design Rules
   4.2 Checking Wire Width
   4.3 Checking Via Spacing
   4.4 Layout After DRC Checks
5. TritonRoute for Detailed Routing
   5.1 TritonRoute Role and Setup
   5.2 Prepared Routing Guides
   5.3 Routing Within and Between Layers
6. Routing Connections and Topology
   6.1 Access Points and Their Clusters
   6.2 Routing Blockages and Optimization
   6.3 Routing Topology Method
7. Parasitic Extraction and SPEF Output
8. OpenLane Physical Design Run and Results
   8.1 Floorplan and Power Setup
   8.2 OpenLane Setup Parameters
   8.3 Routing Results and Run Folders
9. Post-Routing Timing Checks and Final Checklist
10. Complete Physical Design Flow (RTL to GDSII)
11. Tools and Technologies Overview
- Author

---

## 1️⃣ Routing Basics — Global, Fast and Detailed Methods

### 1.1 Routing Introduction and Main Objectives

Routing connects placed standard cells, pins, and other components through the metal layers while satisfying design rules. It proceeds through global routing, fast routing, detailed routing, design-rule-aware routing, and connectivity verification.

<img width="1123" height="523" alt="Screenshot 2026-10-15 181251" src="https://github.com/user-attachments/assets/7b77c21f-fc19-4dfd-a08f-fcece75bc621" />


### 1.2 Comparing Global and Detailed Routing

**Global routing** divides the routing region into a resource grid and finds approximate paths through it, producing route guides for the detailed-routing stage. Its goals: estimate routing paths, manage congestion, allocate routing resources, spot bottlenecks, and generate guides.

**Detailed routing** turns those approximate paths into actual wire segments and vias, working at the level of exact wire locations, layer selection, via placement, design rules, connectivity, and local congestion.

| Feature | Global Routing | Detailed Routing |
|---|---|---|
| Purpose | Approximate routing paths | Actual physical connections |
| Representation | Routing resource grid | Physical wires and vias |
| Output | Routing guides | Detailed routed geometry |
| Main concern | Congestion and resource allocation | Connectivity and design-rule compliance |
| Level of detail | Coarse | Fine-grained |

### 1.3 Fast Routing and Guide Creation

| Stage | Description |
|---|---|
| **Fast Route** | Produces an initial routing solution and generates route guides |
| **Detailed Route** | Converts that initial routing information into physical routing that satisfies detailed design rules |

<img width="1127" height="593" alt="Screenshot 2026-10-15 181307" src="https://github.com/user-attachments/assets/40db181e-4adf-478b-9f45-328eb7706d40" />


---

## 2️⃣ Maze Routing with Lee's Algorithm

Maze routing is a grid-based pathfinding technique for connecting two points while avoiding obstacles. **Lee's Algorithm** does this through wavefront expansion: it explores neighboring grid cells in successive steps, assigning distance values, until it reaches the destination, then backtracks to reconstruct the shortest path.

### 2.1 Main Algorithm Steps

1. Mark the source point as the starting location.
2. Expand the wavefront — explore neighboring grid cells and assign distance values.
3. Exclude blocked or unavailable locations (obstacle avoidance).
4. Continue expanding until the destination is reached.
5. Backtrack along the minimum-distance path.
6. Record the final reconstructed route.


<img width="1130" height="552" alt="Screenshot 2026-10-15 181322" src="https://github.com/user-attachments/assets/bb85bec8-0ba4-4a04-ac92-9d34e8cd52df" />


### 2.2 Benefits and Limitations

**Advantages:** systematic path exploration; guarantees the shortest valid path in an unweighted grid when one exists; naturally handles obstacles by excluding blocked cells; a simple foundation for understanding maze routing.

**Limitations:** memory-intensive for large grids; can explore many unnecessary locations; doesn't inherently account for congestion, wire delay, or layer preference — real VLSI routing needs those factored in on top.

---

## 3️⃣ Power Delivery Network (PDN)

### 3.1 PDN Purpose and Main Components

A PDN distributes power and ground throughout the design, supplying standard cells and macros while keeping voltage levels acceptable. It aims to:

- Distribute power and ground across the chip
- Connect standard cells and macros electrically
- Reduce voltage drop along power paths
- Support current delivery to every region of the design
- Structure the connection between the power source and circuit components

Typical components: power/ground pins, power rings, power straps, standard-cell power rails, macro power connections, and vias linking power conductors across metal layers.

### 3.2 PDN Setup Flow and Configuration

```text
Power and Ground Definition
          |
          v
Power Ring Generation
          |
          v
Power Strap Generation
          |
          v
Power Rail Connection
          |
          v
Standard-Cell and Macro Connections
          |
          v
Power Connectivity Verification
```

In OpenROAD-based flows, PDN generation is scripted in Tcl — e.g. a `pdngen` invocation triggers the configured power-network generation process, with the actual power nets, layers, and geometry defined for the specific design and technology.


<img width="1126" height="745" alt="Screenshot 2026-10-15 181336" src="https://github.com/user-attachments/assets/973d868c-dd93-4330-97cf-2c2319178807" />


### 3.3 Power Straps and Cell/Macro Connections

**Power straps** are wider metal conductors that distribute power/ground across the core, connecting into the PDN and carrying current to cells and macros via designated layers and vias.

**Standard cells** draw power/ground through their power pins and the rails running along their placement rows, which the PDN connects into the broader power network.

**Macros** (e.g. RAM blocks) may need dedicated power pins; the PDN must connect these to the chip-level power network.

Key PDN-design factors: metal-layer selection, strap width/spacing, via connectivity, current demand, voltage drop, macro placement/pin locations, and rail-to-network connectivity.

---

## 4️⃣ Design Rule Check (DRC)

DRC verifies that a physical layout satisfies the fabrication rules of the chosen technology — geometry restrictions that keep the design manufacturable.

### 4.1 Key Layout Design Rules

- Minimum wire width
- Minimum spacing between wires and between vias
- Minimum enclosure and area requirements
- Layer-specific restrictions
- Connectivity constraints, and restrictions on overlapping/improperly connected shapes

A DRC violation surfaces when a route breaks a spacing, width, enclosure, or other physical constraint — routing and verification stages exist to find and resolve these.

### 4.2 Checking Wire Width

Wires thinner than the minimum allowed width can violate manufacturing rules and hurt interconnect reliability.


<img width="1046" height="502" alt="Screenshot 2026-10-15 181346" src="https://github.com/user-attachments/assets/6550038d-a2ea-4aa8-b143-8cb847ae53d6" />


### 4.3 Checking Via Spacing

Correct spacing between vias and neighboring structures prevents manufacturing violations and unintended shorts.


<img width="1017" height="530" alt="Screenshot 2026-10-15 181356" src="https://github.com/user-attachments/assets/7f1525e1-3208-4283-b220-ed545e7b23dc" />


### 4.4 Layout After DRC Checks

A **DRC-clean** result means the layout satisfies every rule checked under the verification conditions used — though that alone doesn't confirm timing, electrical connectivity, or antenna requirements are also met.


<img width="1537" height="727" alt="Screenshot 2026-10-15 181411" src="https://github.com/user-attachments/assets/77e138a2-2752-4c67-acbc-cc71e5c6cd97" />


---

## 5️⃣ TritonRoute for Detailed Routing

### 5.1 TritonRoute Role and Setup

TritonRoute is the detailed-routing engine in the OpenROAD flow. It turns route guides into actual physical connections while respecting routing constraints and design rules — processing guides, connecting pins, routing across metal layers, handling obstacles, resolving conflicts, and enforcing design-rule compliance.

```text
Global Routing → Routing Guides → TritonRoute → Detailed Wire/Via Generation → Routing Verification → Post-Route Design
```

An illustrative OpenROAD command for this stage is `detailed_route`, invoked with the design and routing configuration already loaded; exact options depend on the installed OpenROAD version and flow configuration.

<img width="1822" height="652" alt="Screenshot 2026-10-15 181424" src="https://github.com/user-attachments/assets/2fa05483-4544-44b4-9909-f0280a6e3a7e" />


<img width="1512" height="892" alt="Screenshot 2026-10-15 181442" src="https://github.com/user-attachments/assets/4bac9a4d-a4d1-469a-8a02-700ae4016254" />



### 5.2 Prepared Routing Guides

Route guides mark the preferred regions/directions for routing a net, generated during global routing to steer detailed routing toward the paths chosen earlier. Requirements: guides should have unit width and follow the preferred routing direction. Guides help by providing direction, making use of allocated routing resources, coordinating global and detailed routing, and cutting down unnecessary exploration.


<img width="1392" height="862" alt="Screenshot 2026-10-15 181456" src="https://github.com/user-attachments/assets/b09fccc6-038f-438f-9b80-63e14d9e7aa6" />


<img width="1130" height="630" alt="Screenshot 2026-10-15 181546" src="https://github.com/user-attachments/assets/277cce70-568e-4aa6-ba9a-b81137c90c18" />

<img width="1132" height="661" alt="Screenshot 2026-10-15 181559" src="https://github.com/user-attachments/assets/f2419deb-f0f8-480c-8501-a4f3ea1632f9" />



### 5.3 Routing Within and Between Layers

| Strategy | Description |
|---|---|
| **Intra-Layer (Parallel)** | Connections handled within the same metal layer; multiple routing tasks processed in parallel |
| **Inter-Layer (Sequential)** | Routing proceeds sequentially across layers, using vias to connect conductors, for full connectivity |


<img width="1130" height="567" alt="Screenshot 2026-10-15 181611" src="https://github.com/user-attachments/assets/e67cabeb-1f3d-4ea1-b6f9-06a21c8e29c4" />


A typical multi-layer connection looks like:

```text
Source Pin → Metal Layer 1 → Via → Metal Layer 2 → Via → Destination Pin
```

---

## 6️⃣ Routing Connections and Topology

### 6.1 Access Points and Their Clusters

- **Access Point (AP):** an on-grid point on a route guide's metal layer, used to connect lower-layer segments, upper-layer segments, pins, or I/O ports.
- **Access Point Cluster (APC):** the union of access points that all derive from the same lower-layer segment, upper-layer guide, pin, or I/O port.


<img width="1127" height="585" alt="Screenshot 2026-10-15 181725" src="https://github.com/user-attachments/assets/0f7b3b7d-ada3-4302-923e-4a960ab874a6" />


### 6.2 Routing Blockages and Optimization

**Obstacles** restrict where wires/vias can go — macro blockages, existing routed wires, restricted regions, spacing constraints, and pin-access limits all count. The detailed router must route around these while keeping every net's terminals fully connected — an open connection or incomplete route is a connectivity failure.

**Routing optimization** then improves the physical connections without breaking connectivity, weighing wirelength, congestion, design-rule compliance, via count, signal delay, and routing-resource use.

<img width="1122" height="640" alt="Screenshot 2026-10-15 181743" src="https://github.com/user-attachments/assets/1587dbfc-b58c-4d75-bcb2-b4c71d23dfb2" />


<img width="1128" height="650" alt="Screenshot 2026-10-15 181754" src="https://github.com/user-attachments/assets/6e85ef52-6e7c-4f5b-a1e9-7739e8c6171c" />



### 6.3 Routing Topology Method

The routing-topology algorithm decides how a multi-terminal net's connection points link together efficiently — minimizing routing cost while keeping every terminal connected. The physical router then implements that topology as actual wire segments and vias, respecting all routing constraints; topology choice affects wirelength, resource use, and parasitic characteristics.

<img width="1020" height="500" alt="Screenshot 2026-10-15 181804" src="https://github.com/user-attachments/assets/309aa68a-1c16-4ddd-a55a-82249498d0a8" />



---

## 7️⃣ Parasitic Extraction and SPEF Output

After routing, the resistance, capacitance, interconnect delay, and coupling effects introduced by the physical wires must be extracted for accurate post-layout timing analysis.

<img width="1122" height="502" alt="Screenshot 2026-10-15 181813" src="https://github.com/user-attachments/assets/557b89db-43ce-41ed-91dc-15833d3a983c" />


**SPEF** (Standard Parasitic Exchange Format) carries this extracted parasitic information — resistance, capacitance, nets, interconnect parasitics, and connectivity — generated by an extraction script that processes the routed DEF.

<img width="1130" height="652" alt="Screenshot 2026-10-15 181824" src="https://github.com/user-attachments/assets/628dea05-6421-4d64-b5fc-f9b747586f4b" />

<img width="1128" height="647" alt="Screenshot 2026-10-15 181833" src="https://github.com/user-attachments/assets/db1a9e86-0507-4c5b-909c-fee9a94aadad" />


---

## 8️⃣ OpenLane Physical Design Run and Results

### 8.1 Floorplan and Power Setup

Key floorplanning elements: core/die area, standard-cell rows, I/O placement, power rings and straps, and macro placement — all set up to ensure reliable VPWR/VGND distribution.


<img width="1127" height="638" alt="Screenshot 2026-10-15 181844" src="https://github.com/user-attachments/assets/df02fccb-766c-45a4-b571-d224a2034533" />


### 8.2 OpenLane Setup Parameters

Configuration parameters span placement, CTS, routing, Magic, density, timing, and routing optimization — controlling cell density, clock-tree generation, routing layers/optimization, and layout generation.


<img width="1123" height="492" alt="Screenshot 2026-10-15 181852" src="https://github.com/user-attachments/assets/8b4b5c93-52cc-4d19-bdc8-450606f9695f" />


### 8.3 Routing Results and Run Folders

<img width="1126" height="643" alt="Screenshot 2026-10-15 181902" src="https://github.com/user-attachments/assets/f4845420-db17-4ff9-8162-dac10c4f5bfd" />


Synthesis and routing results are organized into stage-specific run directories:

```text
results/
├── synthesis/
├── routing/
├── placement/
├── cts/
├── floorplan/
└── signoff/
```

---

## 9️⃣ Post-Routing Timing Checks and Final Checklist

<img width="1130" height="670" alt="Screenshot 2026-10-15 181911" src="https://github.com/user-attachments/assets/4b8f08ff-514b-4e8b-bf59-c5b2a84980d4" />



**OpenSTA** evaluates post-route timing using the routed netlist, SDC constraints, and extracted parasitics, computing arrival time, required time, and setup/hold slack:

```text
Routed Netlist
      |
      v
Timing Constraints (SDC)
      |
      v
Parasitic Information
      |
      v
OpenSTA
      |
      v
Arrival Time and Required Time
      |
      v
Setup and Hold Slack
      |
      v
Timing Verification
```

Final implementation checklist:

- [ ] Power distribution network generated
- [ ] Global routing completed
- [ ] Detailed routing completed
- [ ] Routing connectivity checked
- [ ] Design rule checking performed
- [ ] Parasitic information generated
- [ ] Post-route timing analysis performed
- [ ] Final physical design files generated

---

## 🔟 Complete Physical Design Flow (RTL to GDSII)

```text
RTL Design
 ↓
Synthesis
 ↓
Floorplanning
 ↓
Power Distribution Network
 ↓
Standard-Cell Placement
 ↓
Clock Tree Synthesis
 ↓
Global Routing
 ↓
Fast Route / Route Guide Generation
 ↓
Route Guide Preprocessing
 ↓
Detailed Routing (TritonRoute)
 ↓
Design Rule Checking
 ↓
Parasitic Extraction / SPEF Generation
 ↓
Post-Route Timing Analysis (OpenSTA)
 ↓
Physical Verification
 ↓
GDSII
```

---

## 1️⃣1️⃣ Tools and Technologies Overview

| Tool / Technology | Purpose |
|---|---|
| OpenLane | Automated RTL-to-GDSII physical design flow |
| OpenROAD | Physical implementation and optimization |
| TritonRoute | Design-rule-aware detailed routing engine |
| OpenSTA | Static Timing Analysis |
| Yosys | RTL synthesis |
| SKY130 | Open-source 130 nm process technology |
| Magic | Layout viewing and physical verification |
| LEF | Physical cell and technology information |
| DEF | Physical design placement/routing representation |
| SPEF | Standard Parasitic Exchange Format |
| Tcl | Flow configuration and automation |
| Docker | Execution environment |

---

## ✅ Takeaways

- ✅ Understood how global routing (resource grid, route guides) differs from detailed routing (actual wires and vias).
- ✅ Traced Lee's Algorithm's wavefront-expansion approach to maze routing, along with its memory/congestion trade-offs.
- ✅ Built and configured a Power Distribution Network — rings, straps, rails, and macro power connections — via `pdngen`.
- ✅ Applied DRC checks for wire width, via spacing, and overall layout cleanliness.
- ✅ Learned how TritonRoute consumes LEF/DEF/route guides and resolves intra-layer and inter-layer routing to produce a wirelength/via-optimized solution.
- ✅ Understood connectivity handling through Access Points and Access Point Clusters, and how routing topology balances cost against full connectivity.
- ✅ Extracted post-route parasitics into SPEF and used OpenSTA for post-route setup/hold timing verification.
- ✅ Connected the full flow: **PDN → Global Routing → Detailed Routing (TritonRoute) → DRC → Parasitic Extraction → Post-Route STA → GDSII**.

## 👤 Author

**Jahnavi Sri Bhavya**
B.Tech, Electronics and Communication Engineering
Anurag University, Hyderabad

---
