# 🔧 Session 3 — Gate-Level Simulation of good_mux and bad_mux, and BabySoC Post-Synthesis Simulation

<p>
  <img src="https://img.shields.io/badge/Tool-Icarus%20Verilog-blue" alt="Icarus Verilog">
  <img src="https://img.shields.io/badge/Tool-GTKWave-orange" alt="GTKWave">
  <img src="https://img.shields.io/badge/Tool-Yosys-green" alt="Yosys">
  <img src="https://img.shields.io/badge/PDK-SKY130-red" alt="SKY130">
  <img src="https://img.shields.io/badge/Language-Verilog-9cf" alt="Verilog">
</p>

## 📖 Summary

This session closed the loop opened in Session 2: instead of just synthesizing designs and reading the resulting netlist, the synthesized netlists were actually **simulated** — Gate-Level Simulation (GLS) — to confirm that what synthesis produced behaves the same way the RTL was meant to. This was done first on the small `good_mux` / `bad_mux` pair, then at full scale on VSDBabySoC: synthesizing the entire SoC in Yosys, writing out its gate-level netlist, and running BabySoC's post-synthesis simulation for direct comparison against the earlier pre-synthesis run.

| | |
|---|---|
| 🛠️ **Tools used** | Icarus Verilog, GTKWave, Yosys, ABC (technology mapper) |
| 🧩 **Example designs** | `good_mux`, `bad_mux`, VSDBabySoC (`vsdbabysoc`, `rvmyth`, `clk_gate`, `avsdpll`, `avsddac`) |
| 📋 **Prerequisites** | Session 1 & 2 material — Iverilog/GTKWave basics, Yosys synthesis flow, blocking vs. non-blocking assignments |

## 📑 Contents

- 1. Gate-Level Simulation — Purpose and Motivation
- 2. Gate-Level Simulation of good_mux
  - 2.1 Simulating the Netlist
  - 2.2 Verifying Against the Synthesized Schematic
- 3. Gate-Level Simulation of bad_mux
  - 3.1 Simulating the Netlist
  - 3.2 Contrasting with the RTL Simulation Mismatch
- 4. Synthesis of VSDBabySoC
  - 4.1 Loading the Design and Liberty Files
  - 4.2 Generic Synthesis — Hierarchy and Cell Statistics
  - 4.3 Technology Mapping Using ABC
  - 4.4 Examining the Synthesized Netlist
  - 4.5 Viewing the Synthesized Top-Level Schematic
- 5. Post-Synthesis Simulation of BabySoC
  - 5.1 Executing POST_SYNTH_SIM
  - 5.2 Pre-Synthesis vs. Post-Synthesis Waveforms
- 6. Key Learnings

---

## 1️⃣ Gate-Level Simulation — Purpose and Motivation

### 🧠 Theory: Why Simulate the Netlist?

RTL simulation checks that the *source code* behaves as intended. Synthesis then transforms that RTL into a structural netlist of standard cells. **Gate-Level Simulation (GLS)** re-runs the *same testbench*, unchanged, but against that structural netlist instead of the RTL — the actual `sky130_fd_sc_hd__...` cell instances synthesis produced, rather than the abstract `if`/`else`/`always` description.

To simulate a gate-level netlist, the simulator needs to know what each cell instance actually *does*. This requires two extra files that plain RTL simulation never needed:
- A **primitives file** (`primitives.v`) defining Verilog behavioral models for basic gate primitives.
- The **standard-cell Verilog models** (`sky130_fd_sc_hd.v`) — behavioral Verilog for every cell in the library synthesis was allowed to use (`mux2_1`, `nand2_1`, `nor2_1`, and so on), so that `iverilog` can simulate a `sky130_fd_sc_hd__mux2_1` instance exactly as if it were a real gate.

GLS is the closest a simulation gets to "does the actual hardware do what I intended?" — it's the standard sign-off step for catching synthesis-simulation mismatches (like the blocking/non-blocking issue from Session 2) before tape-out, because by this point there's no more abstraction left to hide behind: either the gates behave correctly or they don't.

---

## 2️⃣ Gate-Level Simulation of good_mux

### 2.1 Simulating the Netlist

The synthesized netlist (`good_mux_net.v`, from Session 2) was compiled and simulated together with the library models, and the same `tb_good_mux` testbench used for RTL simulation:

```bash
iverilog ../my_lib/verilog_model/primitives.v ../my_lib/verilog_model/sky130_fd_sc_hd.v good_mux_net.v tb_good_mux.v
./a.out
gtkwave tb_good_mux.vcd
```

<img width="1599" height="899" alt="WhatsApp Image 2026-11-01 at 10 00 32 AM" src="https://github.com/user-attachments/assets/a5e92518-0dfd-454c-8266-a69855b58feb" />



The waveform shows `i0`, `i1`, `sel`, and `y` toggling exactly as expected for a 2:1 MUX, confirming the netlist reproduces `good_mux`'s intended behavior. One visible difference from the RTL-simulation hierarchy: under `tb_good_mux → uut`, the signal list now shows `i0`, `i1`, and `sel` as plain **wires** and `y` as a **reg** — reflecting that these are now ports/nets on an actual gate instance rather than RTL-level `reg`/`wire` declarations from the original behavioral code.

### 2.2 Verifying Against the Synthesized Schematic

<img width="1501" height="746" alt="Screenshot 2026-11-01 100246" src="https://github.com/user-attachments/assets/71412a4e-0c6e-4199-b9bb-9798a27da549" />


Viewing the GLS waveform side-by-side 
---

## 3️⃣ Gate-Level Simulation of bad_mux

### 🧠 Theory: Why This Comparison Is Important

Session 2 showed that `bad_mux` (non-blocking assignments inside a combinational `always @(*)` block) produced a **simulation-only** timing lag in RTL simulation — the output appeared to trail the inputs by a delta cycle. The open question that GLS answers is: *does that lag actually exist in the synthesized hardware, or was it purely an artifact of how the simulator scheduled non-blocking updates?* Running the exact same testbench against `bad_mux`'s synthesized netlist settles this with certainty, because a gate-level netlist has no concept of blocking vs. non-blocking assignment — by the time synthesis is done, only physical gates and their propagation delays remain.

### 3.1 Simulating the Netlist

```bash
iverilog ../my_lib/verilog_model/primitives.v ../my_lib/verilog_model/sky130_fd_sc_hd.v bad_mux_netlist.v tb_bad_mux.v
./a.out
gtkwave tb_bad_mux.vcd
```

<img width="1407" height="717" alt="Screenshot 2026-11-01 100302" src="https://github.com/user-attachments/assets/0577cef6-dfef-4904-9202-fd9289a0c4c9" />


### 3.2 Contrasting with the RTL Simulation Mismatch

The GLS waveform for `bad_mux` tracks `i0`, `i1`, `sel`, and `y` cleanly, matching the behavior seen for `good_mux`'s netlist rather than reproducing the lagging output seen in `bad_mux`'s RTL simulation. This is the concrete illustration of the theory above: for this particular design, Yosys synthesized `bad_mux`'s `if`/`else` structure onto the same kind of MUX cell as `good_mux`, so the netlist itself carries no memory of which assignment operator the RTL used. The mismatch flagged in Session 2 was real and worth fixing — relying on it is bad practice and isn't guaranteed to synthesize safely in general — but the GLS run here is exactly how that guarantee (or lack of one) gets checked, rather than assumed.

---

## 4️⃣ Synthesis of VSDBabySoC

### 🧠 Theory: Synthesis of a Multi-Library, Multi-Module Design

BabySoC combines ordinary digital RTL (`rvmyth`, `clk_gate`) with two blocks that are only available as **pre-characterized black boxes**: `avsdpll` and `avsddac`. Yosys can't synthesize gates for these — it doesn't have their internal RTL — so instead of a `.lib` describing thousands of standard cells, their Liberty files each describe a single black-box cell with its own pins and timing, which synthesis treats as a leaf cell no different in principle from a `sky130_fd_sc_hd` gate. Reading in *three* Liberty files (the SKY130 standard-cell library, plus one each for the PLL and DAC) before synthesis is what lets `synth -top vsdbabysoc` treat the whole hierarchy — real gates and analog black boxes alike — as one consistent technology-mapping problem.

### 4.1 Loading the Design and Liberty Files

```bash
cd BabySoC_Simulation
yosys

read_verilog src/module/vsdbabysoc.v
read_verilog -I src/include/ src/module/rvmyth.v
read_verilog -I src/include/ src/module/clk_gate.v

read_liberty -lib src/lib/avsddac.lib
read_liberty -lib src/lib/avsdpll.lib
read_liberty -lib src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```
<img width="830" height="762" alt="Screenshot 2026-11-01 100322" src="https://github.com/user-attachments/assets/969e6251-0f5e-4d27-99f1-f8387d4dacfc" />



Reading `rvmyth.v` emits warnings about replacing several `CPU_Xreg_value`/`CPU_Dmem_value` memories with lists of registers — Yosys's way of flagging that array-style memory declarations in the generated RVMYTH source are being converted into individual flip-flops rather than a dedicated memory macro, since no memory-compiler cell was targeted for this flow.

### 4.2 Generic Synthesis — Hierarchy and Cell Statistics

```bash
synth -top vsdbabysoc
```

<img width="837" height="557" alt="Screenshot 2026-11-01 100335" src="https://github.com/user-attachments/assets/465143b3-140c-4be2-97ab-3c53dce4b154" />


The reported hierarchy — `vsdbabysoc → rvmyth → clk_gate` (×7) — confirms RVMYTH instantiates seven separate clock-gate cells internally (one per pipeline stage/register bank needing its own gated clock), all rolled up under the single `rvmyth` core. At this stage the cell counts (`$_AND_`, `$_OR_`, `$_MUX_`, `$_DFF_P_`, `$_SDFFE_...`, and so on) are still Yosys's **generic, technology-independent** logic and flip-flop types — not yet SKY130 cells — plus `avsddac` and `avsdpll` sitting in the count as single black-box instances. The `CHECK` pass confirms 0 problems across `clk_gate`, `rvmyth`, and `vsdbabysoc`.


A later statistics dump shows noticeably larger generic-cell counts for the same categories. This is consistent with what happens once RVMYTH's internal hierarchy (its seven `clk_gate` instances and associated logic) gets folded/flattened into the top level for technology mapping — the same logic, now counted without sub-module boundaries to divide it, the same way `multiple_modules` flattening in Session 2 multiplied out gate counts once module boundaries were removed.

### 4.3 Technology Mapping Using ABC

```bash
abc -liberty src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

<img width="1237" height="620" alt="Screenshot 2026-11-01 100401" src="https://github.com/user-attachments/assets/d44b82f6-b72d-4e28-bd9a-b0bf74717c31" />



This is where the generic logic from step 4.2 becomes real SKY130 cells: the `ABC RESULTS` listing enumerates exactly how many instances of each cell type (`nand2_1`, `nor2_1`, `a211oi_1`, `a222oi_1`, `mux2_1`, `xor2_1`, `clkinv_1`, and dozens more) the final netlist uses, down to constant-value cells (`_const0_`, `_const1_`) inserted wherever tied-high/tied-low logic was needed. The `nand2_1` and `a21oi_1` cells dominate by count (1272 and 754 respectively) — unsurprising for a design this size, since simple NAND/AOI-style cells are the most area-efficient building blocks for general combinational logic in a standard-cell library. The follow-up message about extracting "0 gates and 0 wires" and *"Don't call ABC as there is nothing to map"* is Yosys reporting, harmlessly, that a subsequent top-level mapping pass found nothing left to do — the real mapping work had already been completed against the sub-modules in the pass shown above.

### 4.4 Examining the Synthesized Netlist

```bash
write_verilog baby_soc_netlist.v
```

<img width="913" height="541" alt="Screenshot 2026-11-01 100411" src="https://github.com/user-attachments/assets/51fe3e7f-8860-4f6c-95d5-fc57cf1f86a2" />



Near the top of the generated netlist, `clk_gate` appears as a plain behavioral pass-through (`assign gated_clk = free_clk;`) rather than a mapped SKY130 cell — confirming clock gating in this design is modeled functionally rather than implemented with a dedicated integrated clock-gating cell. Just below it, `rvmyth`'s section opens with a long run of auto-generated internal wire names (`_00000`, `_00001`, ...) — the inevitable result of flattening a TL-Verilog-generated core down to gate level, where Yosys has no meaningful original signal names left to preserve for most internal nets.

Further into the file, the netlist reads exactly like the `good_mux`/`good_counter` netlists from Session 2, just at much greater scale — `nand2_1`, `nor2_1`, `a311oi_1`, `o211ai_1` instances chained together through numbered internal nets (`_07724_`, `_07725_`, ...), each one a fully specified SKY130 cell with its pins connected by name.

### 4.5 Viewing the Synthesized Top-Level Schematic

```bash
show vsdbabysoc
```

<img width="1251" height="318" alt="Screenshot 2026-11-01 100421" src="https://github.com/user-attachments/assets/9eeb6860-7261-4495-b822-0fe794a61c18" />


At the top level, the synthesized schematic reads exactly like the SoC's intended block diagram: `ENb_CP`, `ENb_VCO`, `REF`, and `VCO_IN` feed the `pll` (`avsdpll`) block, whose `CLK` output drives the `core` (`rvmyth`), which also takes `reset` and produces `OUT` — routed out as `RV_TO_DAC` into the `dac` (`avsddac`) block alongside `VREFH`, producing the SoC's final analog `OUT`. Seeing the two analog black boxes sit cleanly alongside the synthesized digital core, with the same signal names used throughout RTL, netlist, and schematic, confirms synthesis preserved the design's intended data flow.

---

## 5️⃣ Post-Synthesis Simulation of BabySoC

### 🧠 Theory: Why Directly Compare Pre- and Post-Synthesis Waveforms

Section 5.3 of Session 2 ran BabySoC's **pre-synthesis** simulation directly against the RTL. Now that a real gate-level netlist (`baby_soc_netlist.v`) exists, the same testbench can be recompiled with `POST_SYNTH_SIM` defined instead of `PRE_SYNTH_SIM`, swapping in the netlist plus `primitives.v`/`sky130_fd_sc_hd.v` in place of the original RTL sources — without touching a single line of the testbench itself. Placing both VCD files' waveforms side by side is the definitive check: if the DAC's analog approximation of `RV_TO_DAC` traces the same shape before and after synthesis, the netlist is behaviorally equivalent to the RTL it was generated from, at the level this testbench can observe.

### 5.1 Executing POST_SYNTH_SIM

```bash
iverilog -DPOST_SYNTH_SIM -o post_synth_sim.out testbench.v
vvp post_synth_sim.out
gtkwave post_synth_sim.vcd
```

### 5.2 Pre-Synthesis vs. Post-Synthesis Waveforms

<img width="1599" height="899" alt="vdddd" src="https://github.com/user-attachments/assets/36c22baa-928e-4c37-b657-edadc6560a69" />

<img width="1240" height="510" alt="Screenshot 2026-11-01 100443" src="https://github.com/user-attachments/assets/df76c018-9ba2-4267-9838-4f827d83e3b1" />



With `pre_synth_sim.vcd` and `post_synth_sim.vcd` open as separate tabs in the same GTKWave session, `post_synth_sim.vcd` shows `RV_TO_DAC[9:0]` counting through the same kind of repeating digital pattern seen pre-synthesis, and `OUT` tracing the same rising-and-falling analog approximation — this time produced by the actual gate-level netlist plus the `avsddac`/`avsdpll` behavioral models rather than the original RTL. The match between the two runs is the concrete confirmation that synthesizing BabySoC (Section 4 above) preserved its intended behavior all the way from RTL down to SKY130 gates.

---

## 6️⃣ Key Learnings

- ✅ Learned why Gate-Level Simulation exists: it re-runs the original testbench against the actual synthesized netlist, using `primitives.v` and the SKY130 Verilog cell models, as the closest simulation gets to verifying real hardware behavior.
- ✅ Ran GLS on `good_mux`'s netlist and confirmed its waveform matches the intended MUX behavior, cross-checked directly against the synthesized `mux2_1` schematic.
- ✅ Ran GLS on `bad_mux`'s netlist and found it behaves correctly at the gate level — settling, with a real simulation rather than an assumption, whether the Session 2 blocking/non-blocking mismatch carried through to synthesized hardware for this design.
- ✅ Synthesized the full VSDBabySoC design in Yosys: read in the RTL plus three Liberty libraries (SKY130 standard cells, `avsdpll`, `avsddac`), ran generic synthesis, and mapped the design onto SKY130 cells with ABC.
- ✅ Inspected the resulting `baby_soc_netlist.v`, tracing both the behavioral `clk_gate` pass-through and the fully gate-mapped `rvmyth` core, and viewed the synthesized top-level schematic showing PLL → core → DAC preserved intact.
- ✅ Ran BabySoC's post-synthesis simulation and compared it directly against the earlier pre-synthesis run, confirming the synthesized netlist reproduces the SoC's intended RTL-level behavior.
## Jahnavi Sri Bhavya
