# 🔧 Session 1 — Environment Setup: RISC-V Toolchain, RTL Simulation, and Physical Design Flow

<p>
  <img src="https://img.shields.io/badge/Tool-RISC--V%20GCC-blue" alt="RISC-V GCC">
  <img src="https://img.shields.io/badge/Tool-Spike-orange" alt="Spike">
  <img src="https://img.shields.io/badge/Tool-Icarus%20Verilog-blue" alt="Icarus Verilog">
  <img src="https://img.shields.io/badge/Tool-GTKWave-orange" alt="GTKWave">
  <img src="https://img.shields.io/badge/Tool-Yosys-green" alt="Yosys">
  <img src="https://img.shields.io/badge/Tool-OpenROAD-purple" alt="OpenROAD">
  <img src="https://img.shields.io/badge/PDK-SKY130-red" alt="SKY130">
</p>

## 📖 Overview

This document covers the first environment-setup session, done almost entirely across separate GitHub Codespaces: the RISC-V toolchain, an RTL simulation sanity check, and a full SKY130 physical design flow. Alongside that, the vsdiat course dashboard and lab file downloads were handled locally on Ubuntu.

| | |
|---|---|
| 🛠️ **Environments used** | GitHub Codespaces (×3), local Ubuntu |
| 🧩 **Tools covered** | `riscv64-unknown-elf-gcc`, `spike`, `git`, `iverilog`, `gtkwave`, `yosys`, OpenROAD/ORFS, KLayout |
| 📋 **Prerequisites** | A GitHub account, a local Ubuntu machine or VM |

## 📑 Table of Contents

- 1. GitHub Codespaces — RISC-V Toolchain
  - 1.1 Creating the Codespace
  - 1.2 Compiling and Running with the RISC-V Toolchain
  - 1.3 Comparing Against Native Compilation
- 2. GitHub Codespaces — RTL Simulation Check
- 3. GitHub Codespaces — SKY130 Physical Design Flow
  - 3.1 Running the OpenROAD Flow
  - 3.2 Viewing the Final GDS in KLayout
- 4. Local Ubuntu — vsdiat Dashboard and Downloads
  - 4.1 Browsing the vsdiat Course Dashboard
  - 4.2 Downloading Lab Files to Local Ubuntu
- 5. Takeaways

---

## 1️⃣ GitHub Codespaces — RISC-V Toolchain

**Background**

RISC-V is an open, royalty-free instruction set architecture (ISA) — a specification for what instructions a processor understands, not a specific chip. Because the ISA is open, anyone can build a compiler, simulator, or actual silicon that implements it, which is why an entire open-source toolchain exists around it.

Two tools matter here: `riscv64-unknown-elf-gcc` is a **cross-compiler** — it runs on one architecture (the Codespace's underlying x86/ARM machine) but produces machine code for a *different* architecture (RISC-V). That output can't be run directly on the machine that compiled it. This is where `spike` comes in — it's a **functional ISA simulator**, meaning it emulates a RISC-V CPU in software, executing the cross-compiled RISC-V binary instruction by instruction and reporting what a real RISC-V chip would have done.

### 1.1 Creating the Codespace

The RISC-V toolchain session runs inside a GitHub Codespace created directly (not from a local `git clone`) — the Codespace environment comes pre-configured with the RISC-V toolchain and the `samples` directory already in place.

### 1.2 Compiling and Running with the RISC-V Toolchain

Navigate to the `samples` folder and compile `sum1ton.c` using the RISC-V cross-compiler:

```bash
cd samples
riscv64-unknown-elf-gcc -o sum1ton.o sum1ton.c
```

Run the compiled RISC-V binary using the `spike` ISA simulator:

```bash
spike pk sum1ton.o
```

**Expected output:**

```
Sum from 1 to 9 is 45
```

<img width="457" height="582" alt="Screenshot 2026-10-15 234020" src="https://github.com/user-attachments/assets/354d5f93-c3b3-4339-90f2-a61cf62bcfb3" />

<img width="1142" height="638" alt="Screenshot 2026-10-15 234111" src="https://github.com/user-attachments/assets/8d2f4623-b066-4826-903b-e844dc6fa46a" />


The disassembly view shows the compiled RISC-V assembly instructions, followed by `spike pk sum1ton.o` producing `Sum from 1 to 9 is 45`. Reading the disassembly is worthwhile in itself — it's the actual RISC-V machine instructions the compiler generated from the C source, which is normally hidden behind the compilation step entirely.

### 1.3 Comparing Against Native Compilation

The same source file was also compiled natively (not cross-compiled for RISC-V) to confirm the program's logic is correct independent of the target architecture:

```bash
cc -o sum1ton.o sum1ton.c
./sum1ton.o
```

**Output:**

```
Sum from 1 to 9 is 45
```
<img width="1533" height="420" alt="Screenshot 2026-10-15 234123" src="https://github.com/user-attachments/assets/5010e75d-2a29-41fc-a960-2d2fbc905078" />



Both the RISC-V cross-compiled version (run through `spike`) and the natively-compiled version produce identical output, confirming the C program itself is correct and that the RISC-V toolchain is functioning as expected end-to-end. This distinction matters: a mismatch here would point to a toolchain or simulator problem, not a bug in the C code, since the *same* source produced *different* results depending only on which compiler and execution environment processed it.

---

## 2️⃣ GitHub Codespaces — RTL Simulation Check

**Background**

RTL simulation checks whether a hardware description, written in a language like Verilog, actually behaves the way it's supposed to — before any of it becomes real silicon. Icarus Verilog (`iverilog`) compiles the design together with a testbench (a piece of code that drives inputs and observes outputs) into a runnable simulation. Running that simulation produces a VCD (Value Change Dump) file — a timestamped record of every signal's value over time — which GTKWave then renders as a waveform for visual inspection.

A separate Codespace (`vsd-rtl`), inside the `sky130RTLDesignAndSynthesisWorkshop` directory, was used to confirm the RTL simulation toolchain works end-to-end using the `good_mux` design from the RTL workshop:

```bash
ls -ltr
gtkwave tb_good_mux.vcd
```
<img width="1567" height="647" alt="Screenshot 2026-10-15 234141" src="https://github.com/user-attachments/assets/55398c5b-bc84-47ce-94c3-559bba818cdd" />



**Observation:** the waveform confirms Icarus Verilog and GTKWave are correctly set up in this Codespace and producing readable results, ahead of the rest of the RTL workshop content.

---

## 3️⃣ GitHub Codespaces — SKY130 Physical Design Flow

**Background**

RTL simulation confirms *behavior*, but it doesn't produce anything manufacturable. The physical design flow — often called RTL-to-GDS — is the sequence of steps that turns a logical description into an actual chip layout:

- **Synthesis** maps the RTL onto real standard cells from a technology library (SKY130HD here).
- **Floorplanning** decides the chip's physical dimensions and where major blocks and power structures go.
- **Placement** positions every individual standard cell within that floorplan.
- **Clock-tree synthesis (CTS)** builds the wiring that distributes the clock signal to every flip-flop with minimal skew.
- **Routing** draws the actual metal wires connecting every cell according to the design's logical connections.
- **Fill** inserts filler cells into unused space to satisfy manufacturing density rules.

The final output is a **GDSII** file — the industry-standard format describing every physical shape on every layer of the chip, ready to be sent to a fabrication facility.

### 3.1 Running the OpenROAD Flow

A third Codespace (`vsd-scl180-orfs`) runs the full RTL-to-GDS physical design flow using OpenROAD's flow scripts (ORFS) against the SKY130HD standard-cell library, using `gcd` as the example design.

```bash
cd orfs/flow
make
```

The flow runs through synthesis, floorplanning, placement, clock-tree synthesis, routing, and fill in sequence, logging elapsed time and peak memory for each stage:


### 3.2 Viewing the Final GDS in KLayout

Once the flow completes, the final GDSII layout can be opened directly in KLayout for visual inspection. KLayout is a layout viewer/editor for exactly this kind of file — it renders each mask layer (metal, poly, diffusion, etc.) so the physical result of the flow can be checked visually rather than only trusting the log output.

```bash
klayout ./results/sky130hd/gcd/base/6_final.gds
```

<img width="837" height="723" alt="Screenshot 2026-10-15 234156" src="https://github.com/user-attachments/assets/9806f4bc-2f50-4daf-b81d-6e2504beeb7b" />




**Observation:** this confirms the full flow — RTL in, physical GDSII layout out — completes successfully on the SKY130HD platform.

---

## 4️⃣ Local Ubuntu — vsdiat Dashboard and Downloads

**Background**

Codespaces are convenient for running a fixed toolchain in the browser, but they're ephemeral and depend on an internet connection. A local Ubuntu setup — whether a native install or a VM — provides a persistent environment for course materials and offline work, independent of any single Codespace's lifecycle.

### 4.1 Browsing the vsdiat Course Dashboard

Course content and lesson navigation were handled through the vsdiat dashboard, accessed from a browser on the local Ubuntu machine — used to work through each module's lessons and locate the corresponding lab files.

### 4.2 Downloading Lab Files to Local Ubuntu

After the local Ubuntu environment was set up, the workshop repository was cloned directly:

```bash
git clone https://github.com/kunalg123/sky130RTLDesignAndSynthesisWorkshop.git
cd sky130RTLDesignAndSynthesisWorkshop
```

Lab files and setup scripts referenced from the dashboard were downloaded locally, alongside a local workshop VM used for reference:

<img width="640" height="512" alt="Screenshot 2026-10-15 234205" src="https://github.com/user-attachments/assets/3fa1c788-f5f1-46e8-869b-7292c3cc9524" />


<img width="482" height="552" alt="Screenshot 2026-10-15 234221" src="https://github.com/user-attachments/assets/709ef9ea-a374-4181-bba8-d2cc2f482d60" />


The local environment mirrors the workshop's directory structure — including `sky130RTLDesignAndSynthesisWorkshop`, 'iverilog','gtkwave','yosys'— used for any work done offline rather than in a Codespace.

---

## 5️⃣ Takeaways

- ✅ Set up and used the RISC-V toolchain (`riscv64-unknown-elf-gcc` + `spike`) inside a GitHub Codespace, using Kunal Ghosh's workshop repository.
- ✅ Cross-checked RISC-V execution against a native compilation of the same C program to confirm correctness independent of target architecture.
- ✅ Verified the RTL simulation toolchain end-to-end using the `good_mux` waveform in GTKWave, in a separate Codespace.
- ✅ Ran a complete SKY130 physical design flow (OpenROAD/ORFS) in a third Codespace, from RTL through to a final routed GDSII layout.
- ✅ Visually confirmed the final layout in KLayout.
- ✅ Used the vsdiat dashboard locally on Ubuntu to navigate course content and download lab files for offline reference.

## Jahnavi Sri Bhavya
