# 🔧 Session 2 — Fixing a Faulty MUX, Yosys Synthesis Walkthroughs, and BabySoC Pre-Synthesis Simulation

<p>
  <img src="https://img.shields.io/badge/Tool-Icarus%20Verilog-blue" alt="Icarus Verilog">
  <img src="https://img.shields.io/badge/Tool-GTKWave-orange" alt="GTKWave">
  <img src="https://img.shields.io/badge/Tool-Yosys-green" alt="Yosys">
  <img src="https://img.shields.io/badge/PDK-SKY130-red" alt="SKY130">
  <img src="https://img.shields.io/badge/Language-Verilog-9cf" alt="Verilog">
</p>

## 📖 Summary

The morning of this session started with a debugging exercise — a MUX that looked reasonable but had a subtle coding mistake, found by comparing simulation behavior against what the code actually says. Once fixed, the same MUX became the reference example for a full Yosys synthesis walkthrough, which was then repeated on a counter and a multi-module design to compare hierarchical and flattened synthesis results. The afternoon moved to a different project entirely — cloning and exploring VSDBabySoC, then running its pre-synthesis simulation.

| | |
|---|---|
| 🛠️ **Tools used** | Icarus Verilog, GTKWave, Yosys |
| 🧩 **Example designs** | `bad_mux`, `good_mux`, `good_counter`, `multiple_modules`, VSDBabySoC |
| 📋 **Prerequisites** | Basic familiarity with digital logic and Linux terminal |

## 📑 Contents

- 1. Finding and Fixing the bad_mux Bug
  - 1.1 Identifying the Error
  - 1.2 The Correction
- 2. Synthesizing good_mux with Yosys
- 3. Repeating the Flow on good_counter
- 4. Repeating the Flow on multiple_modules
  - 4.1 Hierarchical Synthesis
  - 4.2 Flattened Synthesis
- 5. BabySoC — Cloning, RVMYTH Exploration, and Pre-Synthesis Simulation
  - 5.1 Cloning VSDBabySoC
  - 5.2 Inspecting the RVMYTH Core
  - 5.3 Simulating Before Synthesis
- 6. Key Learnings

---

## 1️⃣ Finding and Fixing the bad_mux Bug

### 1.1 Identifying the Error

```verilog
module bad_mux (input i0 , input i1 , input sel , output reg y);
always @ (*)
begin
	if(sel)
		y <= i1;
	else
		y <= i0;
end
endmodule
```


<img width="792" height="531" alt="Screenshot 2026-11-01 093911" src="https://github.com/user-attachments/assets/f959a45d-7d67-456f-98c8-3c4d29c6544e" />


At first glance this looks fine — both branches of the `if` are assigned, so there's no missing-else latch issue. The actual mistake is more subtle: it uses **non-blocking assignments (`<=`)** inside a **combinational** `always @(*)` block. Non-blocking assignments schedule their update to happen at the end of the current simulation time step rather than immediately, which is exactly the right behavior for sequential (clocked) logic — but inside combinational logic it can introduce a one-delta-cycle lag that doesn't match what a synthesis tool will actually build.

```bash
iverilog -o bad_mux bad_mux.v tb_bad_mux.v
gtkwave bad_mux.vcd
```


<img width="517" height="357" alt="Screenshot 2026-11-01 094103" src="https://github.com/user-attachments/assets/230759b1-83f2-46b1-b4e5-b3f7303fdef7" />


The waveform (`i0=1`, `i1=0`, `sel=0`, `y=0`) shows the output lagging behind what the select and data lines say it should be at that instant — the visible symptom of using `<=` where `=` belongs.

### 1.2 The Correction

```verilog
module good_mux (input i0 , input i1 , input sel , output reg y);
always @ (*)
begin
	if(sel)
		y = i1;
	else
		y = i0;
end
endmodule
```

Swapping `<=` for `=` is the entire fix. Blocking assignments execute immediately and are read immediately by whatever comes next in the same block — the correct behavior for combinational logic, where there's no clock edge to wait for.

---

## 2️⃣ Synthesizing good_mux with Yosys

With the corrected `good_mux` design in hand, this became the reference walkthrough for the rest of the day's synthesis work:

```bash
yosys
read_verilog good_mux.v
synth -top good_mux
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
show
write_verilog good_mux_net.v
```


<img width="1512" height="770" alt="Screenshot 2026-11-01 094123" src="https://github.com/user-attachments/assets/8bf6737a-a02b-4440-962c-d0421c8c0931" />



<img width="437" height="801" alt="Screenshot 2026-11-01 094154" src="https://github.com/user-attachments/assets/934904b9-159d-4ab9-8996-54daf794e155" />


---

## 3️⃣ Repeating the Flow on good_counter

The identical compile → synthesize → inspect flow was repeated on a counter design:

```bash
yosys
read_verilog good_counter.v
synth -top good_counter
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
show
write_verilog -noattr good_counter_netlist.v
```
<img width="1368" height="317" alt="Screenshot 2026-11-01 094209" src="https://github.com/user-attachments/assets/7ebc12c0-c6c8-4230-9eb8-708d0630bfbc" />

<img width="1481" height="767" alt="Screenshot 2026-11-01 094225" src="https://github.com/user-attachments/assets/f3f616a3-8dec-4135-ab5a-1c58e89ca22c" />



The counter's two bits (`cnt[0]`, `cnt[1]`) each map onto their own `$_DFF_PP0_` flip-flop, with `sky130_fd_sc_hd__nor2_1` and `nor2b_1` cells forming the next-state logic feeding them.

Reading the generated netlist directly confirms what the schematic shows: two separate `always @(posedge clk, posedge reset)` blocks, one per bit of `cnt`, each driven by its own NOR-based next-state logic rather than one shared 2-bit register update.

---

## 4️⃣ Repeating the Flow on multiple_modules

Repeating the flow once more on a multi-module design (`sub_module1` = AND, `sub_module2` = OR, wired together as `multiple_modules`) surfaced a clear difference depending on whether the design is synthesized hierarchically or flattened first.

### 4.1 Hierarchical Synthesis

```bash
read_verilog multiple_modules.v
synth -top multiple_modules
show multiple_modules
```

<img width="482" height="212" alt="Screenshot 2026-11-01 094235" src="https://github.com/user-attachments/assets/58b7eb2b-5074-4767-ba99-9dabbf6bd877" />


At this level, `sub_module1` and `sub_module2` still appear as their own labeled blocks (`u1`, `u2`) in the schematic — the module boundaries from the RTL are preserved rather than merged.

### 4.2 Flattened Synthesis

```bash
flatten
show multiple_modules
```

<img width="1352" height="227" alt="Screenshot 2026-11-01 094244" src="https://github.com/user-attachments/assets/20aafe7d-26cf-4b42-8650-3a5594b7285c" />


After flattening, those same two blocks disappear — `u1.a`, `u1.b`, `u1.y` and `u2.a`, `u2.b`, `u2.y` now sit directly on the gates themselves (`sky130_fd_sc_hd__and2_0` and `sky130_fd_sc_hd__or2_0`), with the `$scopeinfo` markers being the only trace left of the original module names. This is the concrete difference between hierarchical and flat synthesis: same logic, same two gates either way, but the flattened netlist has no sub-module boundaries left to read.

---

## 5️⃣ BabySoC — Cloning, RVMYTH Exploration, and Pre-Synthesis Simulation

### 5.1 Cloning VSDBabySoC

The afternoon moved to a different project — VSDBabySoC, a small RISC-V-based SoC design:

```bash
git clone https://github.com/manili/VSDBabySoC
cd VSDBabySoC
```


The `module/` directory contains the SoC's building blocks: `avsddac.v` and `avsdpll.v` (analog DAC and PLL models), `clk_gate.v`, the RVMYTH core (`rvmyth.v`, `rvmyth_gen.v`, `rvmyth.tlv`), and both pre-synthesis and post-synthesis testbenches.

### 5.2 Inspecting the RVMYTH Core

<img width="457" height="756" alt="Screenshot 2026-11-01 094256" src="https://github.com/user-attachments/assets/7158b18b-5e97-4499-bba7-0c245fc05f8d" />



RVMYTH is a small RISC-V core written in TL-Verilog and generated into plain Verilog (`rvmyth_gen.v`). The comments in the generated file trace out its instruction sequence directly — a small loop built from `ADDI`, `ADD`, `SUB`, and `BNE`/`BEQ` — which is what eventually drives the `OUT` register that feeds the rest of the SoC.

### 5.3 Simulating Before Synthesis

<img width="307" height="876" alt="Screenshot 2026-11-01 094307" src="https://github.com/user-attachments/assets/85b643c0-74d3-4dc2-a928-859427c8a5f4" />



The testbench uses a `PRE_SYNTH_SIM` / `POST_SYNTH_SIM` macro switch to include either the original RTL sources or the synthesized netlist plus SKY130 primitives — the same testbench works for both stages just by defining a different macro at compile time.

```bash
iverilog -DPRE_SYNTH_SIM -o pre_synth_sim.out testbench.v
vvp pre_synth_sim.out
gtkwave pre_synth_sim.vcd
```

<img width="1599" height="899" alt="vdddd" src="https://github.com/user-attachments/assets/b8f669ec-cb99-49df-a4b0-6b66232b79e1" />


**Observation:** the waveform shows `CLK` toggling, `RV_TO_DAC[9:0]` counting through values (`334` at the marker), and `OUT` responding as the DAC's analog approximation of that digital value rises and falls — confirming the RVMYTH core, clock gating, and DAC model are all functioning together correctly at the RTL level, before synthesis is involved at all.

---

## 6️⃣ Key Learnings

- ✅ Diagnosed a synthesis-simulation risk in `bad_mux` caused by non-blocking assignments inside a combinational `always` block, and fixed it by switching to blocking assignments.
- ✅ Ran a complete Yosys synthesis flow on `good_mux`, down to reading the actual `.lib` cell definition Yosys selected.
- ✅ Repeated the same flow on `good_counter`, confirming each counter bit gets its own flip-flop and NOR-based next-state logic.
- ✅ Repeated the flow again on `multiple_modules`, directly comparing hierarchical synthesis (module boundaries preserved) against flattened synthesis (module boundaries dissolved into raw gates).
- ✅ Cloned and explored VSDBabySoC, including reading through the RVMYTH RISC-V core's generated instruction trace.
- ✅ Ran BabySoC's pre-synthesis simulation and confirmed correct RTL-level behavior across the RVMYTH core, clock gating, and DAC model before moving on to synthesis.
## Jahnavi Sri Bhavya
