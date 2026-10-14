# 🔬 Module 3 — CMOS Inverter Design, Fabrication, Layout & Characterization

<p>
  <img src="https://img.shields.io/badge/PDK-SKY130A-blue" alt="SKY130A">
  <img src="https://img.shields.io/badge/Simulator-ngspice-orange" alt="ngspice">
  <img src="https://img.shields.io/badge/Layout-Magic_VLSI-green" alt="Magic VLSI">
  <img src="https://img.shields.io/badge/Format-SPICE-red" alt="SPICE">
  <img src="https://img.shields.io/badge/Verification-DRC-yellow" alt="DRC">
  <img src="https://img.shields.io/badge/VCS-Git%20%26%20GitHub-black" alt="Git & GitHub">
  <img src="https://img.shields.io/badge/OS-Linux-lightgrey" alt="Linux">
</p>

> Part of the SKY130 VLSI Physical Design module series.

## 📖 Overview

This document covers Module 3 of the SKY130 VLSI design flow: designing, simulating, laying out, extracting, and characterizing a CMOS inverter standard cell. It follows the complete path from a transistor-level SPICE circuit through the Voltage Transfer Characteristic (VTC) and switching-threshold analysis, into the 16-mask CMOS fabrication process, a Magic-based standard-cell layout, Design Rule Checking (DRC), SPICE extraction, and final post-layout characterization using ngspice.

|  |  |
|---|---|
| 🛠️ **Tools used** | ngspice, Magic VLSI, SKY130A PDK, SPICE/NGSPICE, Git & GitHub, Linux terminal |
| 🧩 **Example design(s)** | CMOS inverter standard cell (`vsdstdcelldesign`) |
| 📋 **Prerequisites** | SKY130 PDK installed, Magic and ngspice set up, basic MOSFET/CMOS theory, Git installed |

## 📑 Table of Contents

1. CMOS Inverter Circuit Design and SPICE Simulation
   1.1 SPICE Deck Creation
   1.2 DC and Transient Simulation
2. Voltage Transfer Characteristic and Switching Threshold
   2.1 VTC Generation
   2.2 Switching Threshold Vm
   2.3 Transistor Sizing and Its Effect on Vm
3. Static and Dynamic Analysis of the Inverter
4. CMOS Fabrication Process (16-Mask Flow)
   4.1 Active Region Formation (LOCOS)
   4.2 N-Well and P-Well Formation
   4.3 Threshold Voltage and Body Effect
   4.4 Gate Formation
   4.5 LDD Formation and Side-Wall Spacers
   4.6 Source and Drain Formation
   4.7 Contacts and Local Interconnect
   4.8 Higher-Level Metal Formation and Complete Structure
5. Standard Cell Layout Design in Magic
   5.1 Cloning the Design Repository
   5.2 Loading SKY130 Technology Files
   5.3 Cell Boundary, Power and Ground Connectivity
   5.4 CMOS Inverter Layout and Abstract View
6. Design Rule Checking (DRC)
   6.1 DRC Concepts and Common Rules
   6.2 Fixing the poly.9 DRC Error
   6.3 Poly Resistor Spacing to Diffusion and Tap
   6.4 Debugging DRC Errors as Geometrical Constructs
7. SPICE Extraction from the Layout
   7.1 Extraction Flow
   7.2 Extracted Netlist and SPICE File Generation
8. Post-Layout Characterization Using ngspice
   8.1 Transient Simulation of the Extracted Netlist
   8.2 Input/Output Waveforms
   8.3 Timing and Parasitic Effects
9. Overall Module 3 Flow
10. Tools and Technologies Used
- Author

---

## 1️⃣ CMOS Inverter Circuit Design and SPICE Simulation

A CMOS inverter consists of a PMOS transistor (connected to VDD) and an NMOS transistor (connected to GND/VSS), with their gates tied together to form the input and their drains tied together to form the output.

```text
             VDD
              |
             PMOS
              |
Vin ----------|------ Vout
              |
             NMOS
              |
             GND
```

### 1.1 SPICE Deck Creation

The SPICE deck contains transistor definitions, SKY130A device models, circuit connections, the power supply, an input stimulus, simulation commands, and measurement statements. The initial transistor dimensions used were:

$$W_n = W_p = 0.375\mu m, \quad L_n = L_p = 0.25\mu m$$

giving $W_n/L_n = W_p/L_p = 1.5$.


<img width="1225" height="572" alt="Screenshot 2026-10-14 185605" src="https://github.com/user-attachments/assets/37322f0d-8532-4e59-8160-df27009c4203" />


### 1.2 DC and Transient Simulation

The circuit was simulated in ngspice to observe input voltage, output voltage, current, and switching behavior. The inverter operates through complementary switching:

- When **Vin is LOW** → PMOS is ON, NMOS is OFF → **Vout is HIGH**
- When **Vin is HIGH** → NMOS is ON, PMOS is OFF → **Vout is LOW**
- During the transition region, both transistors influence the output.


<img width="1250" height="553" alt="Screenshot 2026-10-14 185628" src="https://github.com/user-attachments/assets/adade8b3-2ad7-4172-86df-534796a2f2f2" />


---

## 2️⃣ Voltage Transfer Characteristic and Switching Threshold

### 2.1 VTC Generation

The VTC represents $V_{out} = f(V_{in})$ and can be divided into three regions: HIGH output, transition, and LOW output. During the HIGH region $V_{out} \approx V_{DD}$, and during the LOW region $V_{out} \approx 0$. The steep transition region indicates high voltage gain around the switching point.


<img width="1050" height="848" alt="Screenshot 2026-10-14 185645" src="https://github.com/user-attachments/assets/b0a83b4f-e358-4608-8c93-9dda5e348b0a" />


### 2.2 Switching Threshold Vm

The switching threshold **Vm** is the input voltage at which $V_{in} = V_{out}$, i.e. where the inverter transitions between logic states. Key VTC parameters are:

| Parameter | Meaning |
|---|---|
| VOH | Output High Voltage |
| VOL | Output Low Voltage |
| VIH | Input High Voltage |
| VIL | Input Low Voltage |
| Vm | Switching Threshold |

Observed values across different sizing configurations were approximately **Vm ≈ 0.98 V** and **Vm ≈ 1.2 V**.

<img width="1050" height="547" alt="Screenshot 2026-10-14 185657" src="https://github.com/user-attachments/assets/d5d5eeb7-b3f3-48b6-8c88-ce44c8103ffe" />


Vm can also be derived analytically from the relative drive strengths of the NMOS/PMOS devices, considering $W_n/L_n$, $W_p/L_p$, $K_n$, $K_p$, saturation voltage, and device threshold parameters.


<img width="1115" height="536" alt="Screenshot 2026-10-14 185711" src="https://github.com/user-attachments/assets/bafa5e0f-937c-4907-87cc-8aa69a4e10df" />



### 2.3 Transistor Sizing and Its Effect on Vm

Two configurations were compared:

| Config | $W_n/L_n$ | $W_p/L_p$ | $W_p$ |
|---|---|---|---|
| 1 | 1.5 | 1.5 | 0.375 µm |
| 2 | 1.5 | 3.75 | 0.9375 µm |

Increasing PMOS width increases its relative drive strength, shifting the switching threshold and changing the timing characteristics.

<img width="1081" height="577" alt="Screenshot 2026-10-14 185724" src="https://github.com/user-attachments/assets/df52a498-2cc6-4f3b-a53f-71c751d3a434" />



<img width="1052" height="571" alt="Screenshot 2026-10-14 185734" src="https://github.com/user-attachments/assets/cf105af2-cb7d-4328-9462-5d18409a9fd6" />


---

## 3️⃣ Static and Dynamic Analysis of the Inverter

**Static analysis** studies DC characteristics: VOH, VOL, VIH, VIL, Vm, noise margins, and the DC transfer curve (VTC).

**Dynamic analysis** studies time-dependent behavior: propagation delay, rise time, fall time, charging/discharging behavior, and dynamic power consumption — important because standard cells must meet timing at the target operating frequency.

---

## 4️⃣ CMOS Fabrication Process (16-Mask Flow)

The physical implementation of a standard cell is built through a sequence of fabrication steps that define what each layout layer physically represents.

### 4.1 Active Region Formation (LOCOS)

The active region defines where transistor source/drain regions are formed — NMOS in the P-type region, PMOS inside the N-well. The process used is **LOCOS (Local Oxidation of Silicon)**, involving a P-type substrate, silicon nitride masking, photoresist, field oxide, and the resulting bird's-beak effect, used to isolate active regions from surrounding silicon.

<img width="897" height="451" alt="Screenshot 2026-09-26 073145" src="https://github.com/user-attachments/assets/52983bbf-86aa-46ea-89f5-6994c33e278d" />

### 4.2 N-Well and P-Well Formation

Ion implantation creates the N-well (where PMOS is formed) and P-well (where NMOS is formed), establishing the correct body environment, isolation, and substrate biasing for CMOS operation.

<img width="952" height="436" alt="Screenshot 2026-09-26 073156" src="https://github.com/user-attachments/assets/25bec299-7b60-4d0b-9db4-2a2234b0d286" />

### 4.3 Threshold Voltage and Body Effect

MOS threshold voltage depends on:

- $V_{T0}$ — threshold voltage at zero body bias
- $\gamma$ — body-effect coefficient
- $V_{SB}$ — source-to-body voltage
- $\Phi_F$ — Fermi potential
- $N_A$ — doping concentration
- $C_{ox}$ — oxide capacitance


<img width="1197" height="565" alt="Screenshot 2026-10-14 225855" src="https://github.com/user-attachments/assets/383900a4-19dc-4017-9b32-85ae23b93cd9" />


### 4.4 Gate Formation

Polysilicon deposited over the active region forms the gate; the poly/active intersection forms the channel. In a CMOS inverter, the PMOS and NMOS gates are tied together to form the input, and the drains are tied together to form the output.

<img width="1087" height="542" alt="Screenshot 2026-10-14 225906" src="https://github.com/user-attachments/assets/cfe52062-ec05-4718-8227-07628880486f" />


<img width="1032" height="540" alt="Screenshot 2026-10-14 225917" src="https://github.com/user-attachments/assets/5c0506f5-27b9-4b62-a269-02aaa61e9ef0" />


### 4.5 LDD Formation and Side-Wall Spacers

Lightly Doped Drain (LDD) regions near the source/drain reduce the electric field at the drain, improving reliability and hot-carrier performance. Phosphorus implantation is used to set the required doping profile, followed by side-wall spacer formation, which sets the separation between the gate and the later heavily-doped source/drain regions.


<img width="966" height="545" alt="Screenshot 2026-10-14 225929" src="https://github.com/user-attachments/assets/3929cf37-ea18-4402-8782-c4309ce449e7" />



<img width="1105" height="538" alt="Screenshot 2026-10-14 225942" src="https://github.com/user-attachments/assets/3a28cd77-6d79-41ec-9d7b-823d34152b3a" />


<img width="938" height="538" alt="Screenshot 2026-10-14 230013" src="https://github.com/user-attachments/assets/19575255-470e-4698-9235-683e72f90ec1" />



### 4.6 Source and Drain Formation

Final source/drain regions are formed by high-temperature doping — N-type for NMOS, P-type for PMOS — completing the basic transistor structure:

```text
        Gate
         │
    ┌────┴────┐
    │ Channel │
────┴─────────┴────
 Source       Drain
```


<img width="1133" height="562" alt="Screenshot 2026-10-14 230028" src="https://github.com/user-attachments/assets/478430ef-154d-4c6b-a33c-4a59b4e8fa43" />


### 4.7 Contacts and Local Interconnect

Titanium is sputtered onto the wafer to prepare low-resistance connections, followed by contact formation linking source, drain, and gate to the local interconnect layer — turning isolated transistors into electrically accessible devices.

<img width="948" height="541" alt="Screenshot 2026-10-14 230039" src="https://github.com/user-attachments/assets/b03a3999-4b2f-4fb3-bfea-7eedfebf8c9d" />


<img width="990" height="538" alt="Screenshot 2026-10-14 230048" src="https://github.com/user-attachments/assets/d76b8fbb-2806-4ee5-b9dc-a6e808c17ae8" />


### 4.8 Higher-Level Metal Formation and Complete Structure

Higher-level metal layers route signals, VDD, and VSS across the chip and connect individual cells into a complete circuit network.

<img width="1036" height="545" alt="Screenshot 2026-10-14 230102" src="https://github.com/user-attachments/assets/4d10ad65-e0bc-4816-ba6e-d096b8825d37" />




<img width="772" height="525" alt="Screenshot 2026-10-14 230114" src="https://github.com/user-attachments/assets/4bceca09-1fb4-4ee5-b17d-e060a6e99661" />


---

## 5️⃣ Standard Cell Layout Design in Magic

### 5.1 Cloning the Design Repository

```bash
git clone <repository-url>
cd <repository-directory>
```

The cloned `vsdstdcelldesign` repository provides the starting environment for the layout and characterization flow.

### 5.2 Loading SKY130 Technology Files

Magic requires the SKY130A technology file (`sky130A.tech`) to correctly interpret layer names, connectivity, design rules, and DRC/extraction rules.

<img width="1257" height="736" alt="Screenshot 2026-10-14 230135" src="https://github.com/user-attachments/assets/6220caa2-97bd-420f-8e86-f1aa1324a506" />


<img width="1235" height="720" alt="Screenshot 2026-10-14 230147" src="https://github.com/user-attachments/assets/ff7267ca-fe55-42e6-a2a2-ab69d8302605" />


### 5.3 Cell Boundary, Power and Ground Connectivity

A cell boundary defines the exact width/height occupied by the cell, ensuring consistent dimensions, accurate placement, alignment with neighboring cells, and correct VDD/GND rail locations for library compatibility.

VDD and GND rails are then connected: the PMOS network toward VDD and the NMOS network toward GND, matching the pull-up/pull-down structure of the inverter.

### 5.4 CMOS Inverter Layout and Abstract View

The completed layout contains the PMOS/NMOS transistors, input/output pins, VDD/VSS connections, well/tap structures, and metal routing, built using SKY130 layers (active/diffusion, poly, contact, metal1/2, N-well, implant, and tap layers).

<img width="1600" height="949" alt="WhatsApp Image 2026-09-25 at 7 51 30 PM (15)" src="https://github.com/user-attachments/assets/8aacbd4a-50e4-4fd3-bddc-2eeed4cd1412" />


An **abstract view** (LEF — Library Exchange Format) provides a simplified physical representation containing cell dimensions, pin locations/names, routing layers, obstructions, and placement information, without full transistor geometry.


---

## 6️⃣ Design Rule Checking (DRC)

### 6.1 DRC Concepts and Common Rules

DRC verifies that a layout satisfies the physical design rules of the technology before it can be considered valid. Common rule types include:

- Minimum width
- Minimum spacing
- Minimum enclosure
- Minimum overlap
- Minimum extension
- Layer-specific spacing requirements

<img width="693" height="865" alt="Screenshot 2026-09-26 081726" src="https://github.com/user-attachments/assets/9cb0cc71-c60c-462f-b3ca-072edb586d3d" />

### 6.2 Fixing the poly.9 DRC Error

Debugging flow used for the `poly.9` violation:

1. Identify the reported DRC location.
2. Inspect the affected geometry in Magic.
3. Understand the rule associated with the error.
4. Determine the required geometrical relationship.
5. Modify the layout or the relevant technology rule.
6. Re-run DRC and verify the error is resolved.

### 6.3 Poly Resistor Spacing to Diffusion and Tap

This exercise checks the minimum spacing between polysilicon resistor structures, diffusion regions, and tap regions — incorrect spacing here produces DRC violations, so the layout must meet SKY130's minimum spacing rules.

### 6.4 Debugging DRC Errors as Geometrical Constructs

Rather than treating a DRC error as just a message, it should be analyzed as a geometrical violation by identifying: the affected layers, their geometrical relationship, the required rule condition, the actual layout condition, and the needed correction.

<img width="1600" height="947" alt="WhatsApp Image 2026-09-25 at 7 51 30 PM (8)" src="https://github.com/user-attachments/assets/71ded9d7-0289-4ac5-b453-bec84074ca8c" />

---

## 7️⃣ SPICE Extraction from the Layout

### 7.1 Extraction Flow

```text
Circuit Design
      ↓
Physical Layout
      ↓
DRC Verification
      ↓
SPICE Extraction
      ↓
Extracted SPICE Netlist
      ↓
Simulation
```

Extraction identifies the transistors, connections, nodes, device parameters, and parasitic components implied by the physical geometry, converting it into an electrical (SPICE-compatible) representation.

<img width="882" height="855" alt="Screenshot 2026-09-26 081917" src="https://github.com/user-attachments/assets/07db20c5-d8a9-4b3f-828f-8fece6df6223" />

### 7.2 Extracted Netlist and SPICE File Generation

The extracted `.ext`/SPICE files are generated and inspected, then combined with SKY130 model files and a standard-cell subcircuit definition (input/output nodes, VDD, GND) into a simulation-ready SPICE deck.

<img width="1600" height="953" alt="WhatsApp Image 2026-09-25 at 7 51 30 PM (13)" src="https://github.com/user-attachments/assets/fb092721-f8ac-422e-a1e4-84210e0b3bd1" />

<img width="1600" height="947" alt="WhatsApp Image 2026-09-25 at 7 51 30 PM (12)" src="https://github.com/user-attachments/assets/1530d294-cf8e-4a01-93b5-bd30e57ee0a5" />

---

## 8️⃣ Post-Layout Characterization Using ngspice

### 8.1 Transient Simulation of the Extracted Netlist

The extracted netlist is simulated in ngspice with a time-varying input while monitoring Input, Output, VDD, and GND to verify correct connectivity and switching behavior.

<img width="1600" height="951" alt="WhatsApp Image 2026-09-25 at 7 51 30 PM (11)" src="https://github.com/user-attachments/assets/f5e4238c-9524-49ca-9e1c-4c97488902e9" />

<img width="1600" height="945" alt="WhatsApp Image 2026-09-25 at 7 51 30 PM (10)" src="https://github.com/user-attachments/assets/a67648d4-360e-4dd8-b082-ad9ffc9f0278" />


### 8.2 Input/Output Waveforms

| Input | Output |
|---|---|
| LOW | HIGH |
| HIGH | LOW |

The fundamental relationship confirmed is **Output = NOT(Input)**, with the output approaching the expected VDD/GND levels.

<img width="1600" height="949" alt="WhatsApp Image 2026-09-25 at 7 51 30 PM (9)" src="https://github.com/user-attachments/assets/07966352-3fef-4048-8e26-8238a008cb9b" />

### 8.3 Timing and Parasitic Effects

Extracted parasitic elements (from real layout geometry) can influence propagation delay, rise time, fall time, and output transition speed — making post-layout simulation more realistic than an ideal schematic-level simulation. Key timing parameters tracked: rise time, fall time, propagation delay, and input/output transition time.

---

## 9️⃣ Overall Module 3 Flow

```text
                    SKY130 MODULE 3
                           |
                           ↓
              CMOS Inverter Design
                           |
                           ↓
                 SPICE Deck Creation
                           |
                           ↓
                  ngspice Simulation
                           |
                           ↓
                Switching Threshold Vm
                           |
                           ↓
             Static & Dynamic Analysis
                           |
                           ↓
                 Sky130 PDK Models
                           |
                           ↓
              CMOS Fabrication Process
                           |
                           ↓
                 Magic Layout Design
                           |
                           ↓
              Standard Cell Layout
                           |
                           ↓
                 DRC Verification
                           |
                           ↓
              SPICE Netlist Extraction
                           |
                           ↓
               Inverter Characterization
                           |
                           ↓
                    LEF Generation
                           |
                           ↓
              Standard Cell Library
```

---

## 🔟 Tools and Technologies Used

| Tool / Technology | Purpose |
|---|---|
| ngspice / NGSPICE | CMOS circuit simulation and characterization |
| Magic VLSI | Layout creation, editing, DRC, and SPICE extraction |
| SKY130A PDK | Technology rules, device models, and design information |
| SPICE | Circuit-level simulation |
| Git & GitHub | Version control, repository management, documentation |
| LEF | Abstract physical representation of standard cells |
| DRC | Physical design-rule verification |
| SKY130 Model Files | Device-level simulation and characterization |
| Standard Cell Library | Reusable digital logic cells |

---

## ✅ Takeaways

- ✅ Understood the working principle of a CMOS inverter through complementary PMOS/NMOS switching.
- ✅ Built and simulated a transistor-level CMOS inverter using SPICE and ngspice.
- ✅ Generated the VTC and identified the switching threshold Vm both from simulation and analytically.
- ✅ Studied how transistor sizing ($W/L$ ratios) affects Vm, drive strength, and timing.
- ✅ Learned the 16-mask CMOS fabrication sequence — from LOCOS active-region formation through wells, gate, LDD, source/drain, contacts, and metallization.
- ✅ Created a CMOS inverter standard-cell layout in Magic using SKY130A technology files, including cell boundary and power/ground connectivity.
- ✅ Ran DRC, interpreted violations (including the `poly.9` error) as geometrical constructs, and resolved them.
- ✅ Extracted a SPICE netlist directly from the physical layout and generated an LEF abstract view.
- ✅ Performed post-layout ngspice transient simulation, verifying `Output = NOT(Input)` and studying parasitic/timing effects.
- ✅ Connected the full flow: **Simulation → Fabrication Understanding → Layout → DRC → Extraction → Characterization → Standard Cell Library**, forming a foundation for RTL-to-GDSII physical design.

## 👤 Author

**Arpitha**
B.Tech, Electronics and Communication Engineering
Anurag University, Hyderabad
