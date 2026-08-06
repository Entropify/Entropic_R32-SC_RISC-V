
<img width="1702" height="630" alt="Entropic R32-SC" src="https://github.com/user-attachments/assets/d4081ac6-d2c4-4241-be71-92c804438ccd" />

<h1 align="center">Entropic R32-SC (RV32I CPU)</h1>
<p align="center"><i>(image above is the real GDS render of the CPU on a SKY130 ASIC)</i></p>

<div align="center">

![RISC-V](https://img.shields.io/badge/riscv-%23283272.svg?style=for-the-badge&logo=riscv&logoColor=white) ![Static Badge](https://img.shields.io/badge/V-Verilog-blue?style=for-the-badge) ![Static Badge](https://img.shields.io/badge/SV-SystemVerilog-darkblue?style=for-the-badge) ![Static Badge](https://img.shields.io/badge/C-cocotb-yellow?style=for-the-badge)
 ![Python](https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54) ![AssemblyScript](https://img.shields.io/badge/assembly-%23000000.svg?style=for-the-badge&logo=assemblyscript&logoColor=white) ![Static Badge](https://img.shields.io/badge/IV-Icarus%20Verilog-lightblue?style=for-the-badge)
![Static Badge](https://img.shields.io/badge/OP2-OpenLane2-black?style=for-the-badge) ![Static Badge](https://img.shields.io/badge/M-Makefile-orange?style=for-the-badge) ![Static Badge](https://img.shields.io/badge/SKY-SKY130-blue?style=for-the-badge)


</div>






**Entropic R32-SC** is a single-cycle **RISC-V** (RV32I) CPU that was:
- **designed** and **built completely from scratch** in **Verilog**
- **fully verified** through **self-written testbenches** in **SystemVerilog** and **RISC-V Assembly**, co-driven by **cocotb**
- **synthesized** to a **physical ASIC layout** through **OpenLane2** on the **SkyWater 130nm** open-source PDK

`R32-SC` = **R**V**32**I, **S**ingle-**C**ycle. First entry in the Entropic (I like coming up with cool names) core lineup, a pipelined successor is coming next.

---

## Table of Contents

- [Overview](#Overview)
- [Architecture](#Architecture)
- [GDS Render](#GDS-Render)
- [Instruction Set Coverage](#Instruction-Set-Coverage)
- [Design](#Design)
- [Verification](#Verification)
- [ASIC Implementation](#ASIC-Implementation)
- [Repository Structure](#Repository-Structure)
- [Future Plans](#Future-Plans)
- [Licence](#License)

---

## Overview

Entropic R32-SC is a complete RV32I implementation built and verified from the ground up as a passion project ahead of starting Electrical Engineering at the University of Waterloo.

It covers the full base instruction set: arithmetic/logic, immediates, loads/stores (including byte/halfword variants), branches, jumps, and upper-immediate instructions.


The chip features:
- Top level module verified through **directed**, **self-checking** tests written in **RISC-V Assembly**
- Submodule level verified through both **constrained-random testing** against **golden models** as well as **directed tests** for edge case testing
- Synthesized and physically implemented through the full RTL-to-GDSII flow using [OpenLane2](https://github.com/efabless/openlane2) and the [SKY130 PDK](https://github.com/google/skywater-pdk), ran locally in a Docker + WSL environment

---

## Architecture

`soc_top` wraps the core (`rv32i_core`) together with separate instruction (ROM) and data memory (RAM) modules, connected via a mapping interface.

**Microarchitecture diagram (zoom in if needed Github won't let me make it any larger):**

<img width="2883" height="1914" alt="Untitled Diagram drawio" src="https://github.com/user-attachments/assets/597e9e28-8283-4add-9fd4-9fb336eaa12f" />



**Datapath:**
- **Fetch:** Program Counter → Instruction Memory
- **Decode:** Control Unit + Immediate Generator + ALU Control, driven off the fetched instruction
- **Execute:** Register File → ALU (with Branch Comparator running in parallel for branch instructions)
- **Memory Access:** Data Memory, with a Load Filter (byte/halfword sign- and zero-extension) and Store Mask (byte/halfword write-enable) handling sub-word accesses
- **Writeback:** A 4-to-1 mux selects between ALU result, filtered memory data, `PC+4` (for `jal`/`jalr` link), and the immediate (for `lui`), based on `mem_to_reg`

Everything happens within a single clock cycle. There's no pipelining (yet!) and all the hassle that comes with that. 

The tradeoff is clock speed: the core's maximum frequency is limited by its single longest instruction path (see [ASIC Implementation](#asic-implementation) for the critical path).

---

## GDS Render

<img width="2557" height="1437" alt="Screenshot 2026-07-28 225719" src="https://github.com/user-attachments/assets/da7afc6a-dab7-48a9-8336-674ec5ae6333" />

---

## Instruction Set Coverage

Full RV32I base instruction set: 40/40 instructions implemented.

| Category | Instructions |
|---|---|
| Register-Immediate ALU | `ADDI` `SLTI` `SLTIU` `ANDI` `ORI` `XORI` `SLLI` `SRLI` `SRAI` |
| Register-Register ALU | `ADD` `SUB` `SLL` `SLT` `SLTU` `SRL` `SRA` `XOR` `OR` `AND` |
| Upper Immediate | `LUI` `AUIPC` |
| Loads | `LB` `LH` `LW` `LBU` `LHU` |
| Stores | `SB` `SH` `SW` |
| Jumps | `JAL` `JALR` |
| Branches | `BEQ` `BNE` `BLT` `BGE` `BLTU` `BGEU` |
| System | `FENCE` `ECALL` `EBREAK` |

`ECALL`/`EBREAK` currently raise a `halt` signal rather than implementing full trap/CSR machinery, CSR support will be implemented in a future pipelined CPU.

`FENCE` is currently implemented as `NOP` and will be reworked after pipelining as well.

---

## Design

A few notable design decisions worth calling out:

- **Active-low, asynchronous reset (`rst_n`)** throughout the design to ensure a stable, consistant reset signal during power-up
 
- **Combinational register file reads:** `rs1`/`rs2` data is available same-cycle, consistent with a single-cycle architecture where everything must resolve within one clock period
 
- **`x0` hardwired to zero:** writes to `x0` are architecturally discarded, verified explicitly in testing
 
- **4-bit ALU control encoding**, decoded from opcode/`funct3`/`funct7` via a dedicated `alu_control` module rather than inline in the ALU itself, keeping the ALU's own logic purely arithmetic
 
- **2-bit `pc_src`** to cleanly distinguish between sequential (`PC+4`), branch, and jump (`jal`/`jalr`) targets in the next-PC mux
 
- **2-bit `mem_to_reg`** to distinguish between writing ALU result, writing RAM read, writing link address (`PC+4`) for `jal`/`jalr`, and writing directly from `imm_gen` for `lui` to bypass unnecessary data routing
  
- **Separate `load_filter` / `store_mask` modules** for byte/halfword handling, rather than embedding sign-extension and byte-lane logic directly in `data_mem`; this keeps the memory module itself simple and the addressing logic easily testbench-able
 
- **Detached Branch Comparator:** Branch condition logic is evaluated completely independently from the main ALU arithmetic; this significantly shortens the critical path during branch instructions
 
- **`pc.v` broken out as its own module** rather than inlined into `rv32i_core`. This keeps next-PC muxing (sequential/branch/jump) independently testable, same idea as the load/store split above
 
- **`jalr` hardware bitmask:** The least significant bit of the `jalr` target address is hardwired to zero via a `{[31:1], 1'b0}` concatenation at the PC multiplexer, enforcing the RISC-V specification at the hardware routing level rather than relying on the ALU.

### Module list (`rtl/`)

Top modules: `soc_top.v` · `rv32i_core.v`

Submodules: `alu.v` · `alu_control.v` · `branch_comp.v` · `control_unit.v` · `data_mem.v` · `imm_gen.v` · `instruction_mem.v` · `load_filter.v` · `pc.v` · `reg_file.v` · `store_mask.v`

---

## Verification

A two-tier verification strategy: module-level constrained-random testing, plus full-core directed self-checking tests.

### Module-level (Icarus + golden model + CRT & directed testing)

Key combinational modules (`alu`, `load_filter`, `store_mask`, `branch_comp`, `imm_gen`, `reg_file`) each have its own dedicated SystemVerilog testbench.

These testbenches run constrained-random testings (10,000+ iterations) against a golden reference model unique to each module, in addition to directed corner-case tests (sign-extension boundaries, each byte/halfword offset, default fallback cases). 

Failures report expected vs. actual values directly via `$fatal`.

***Example of module-level constrained random testbench generated waveform (the entire testbench is way longer with 10k+ iterations, example here zoomed in a lot and is the testbench for register file):***

<img width="2241" height="280" alt="Screenshot 2026-07-29 000940" src="https://github.com/user-attachments/assets/5d1c5711-613d-463d-8447-a84c75b14b00" />

### Full-core (Python cocotb + Icarus + Makefile + self-checking RISC-V Assembly)

`tb/tb_top_soc.s` is a self-checking RISC-V assembly program loaded into instruction memory and co-driven by `tb_top_soc.py` through cocotb and Icarus Verilog's VPI.
The assembly program exercises every single instruction in the ISA extensively. Each instruction's result is checked against a hand-computed expected value.

On failure, a self-designed error reporting system consists of unique codes (2–36) writes the error code to reg `x10` and branches the core into a halt label to stop the testbench, making failures immediately diagnosable without tediously digging through waveforms.

***Successful top level testbench generated waveform:***

<img width="2241" height="642" alt="Screenshot 2026-07-28 230323" src="https://github.com/user-attachments/assets/37a200f2-3f98-4eea-9fd2-425d38584c06" />



### Known limitations

- Full-core differential testing against a reference ISA simulator (a self-written Python interpreter, and/or Spike) and the official [`riscv-arch-test`](https://github.com/riscv-non-isa/riscv-arch-test) compliance suite are planned but not yet implemented. I will implement them after the pipelined core is complete.

---

## ASIC Implementation

Synthesized end-to-end (RTL → GDSII) using **OpenLane2** against the **SKY130** open-source PDK, run locally via WSL2 + Docker rather than through a premade CI like TinyTapeout's Github Actions ASIC flow, specifically to work through the toolchain hands-on and understand each step of the standard ASIC flow (synthesis → floorplan → placement → CTS → routing → DRC/LVS → GDS).

The core (`rv32i_core`) was synthesized standalone, independent of the instruction/data memory modules, which is similar to how memory is typically implemented as a separate hard macro (SRAM) in real ASIC design rather than synthesized flip-flop arrays that takes up many more cells.

### Results

| Metric | Value |
|---|---|
| Clock period | 28 ns |
| Clock speed | ~35.7 MHz |
| Total cell count | 7,772 |
| Flip-flops | 1,024 *(the 32×32-bit register file)* |
| Wire count | 7,680 |
| Total wire length | 492,649 μm (~0.49 m) |
| Core Area | 180,939 µm² |
| Logic Area | 85,604 µm² |
| Core Utilization | 47.3% |
| DRC | 0 violations |
| LVS | Circuits match |
| Setup timing | Met, ~1.2 ns margin |


Initially, the design ran into a couple issues I had to manually debug during the ASIC flow:

- An initial 20 ns clock period caused setup timing violations in the `ss` (slow-slow) corner (poorest transistor production, 100°C operating temperature, 1.6v lower than usual operating voltage), which was resolved through a combination of increasing the clock period to 28 ns and setting the `SYNTH_STRATEGY` variable in OpenLane2 to "DELAY 1".
- Antenna rule violations were resolved by setting `DIODE_INSERTION_STRATEGY` to a value of 3 (OpenROAD Antenna Avoidance Flow, which auto-inserts protection diodes during routing).
- Max slew / max cap violations caused by a high-fanout net. I traced a specific problematic net (_00997_) through the synthesized netlist to its source: the instruction[19] bit (part of the rs1 field) fanning out to 115 flip-flop inputs across the register file's address decode. This is partially alleviated via changing the `SYNTH_MAX_FANOUT` variable but not fully resolved yet. I think a full resolution of this in the future would require architecturally reworking the register file addressing logic by adding buffers.


### Disassembled GDS render:

<img width="2557" height="1437" alt="Screenshot 2026-07-29 014316" src="https://github.com/user-attachments/assets/33b3c854-9752-410c-927a-cedfee62a490" />


### Cell breakdown:

| Category | Cell Types | Count |
|---|---|---|
| Combinational (AOI/OAI compound gates) | a2111o, a2111oi, a211o, a211oi, a21bo, a21boi, a21o, a21oi, a221o, a221oi, a22o, a22oi, a2bb2o, a2bb2oi, a311o, a311oi, a31o, a31oi, a32o, a32oi, a41o, a41oi, o2111a, o2111ai, o211a, o211ai, o21a, o21ai, o21ba, o21bai, o221a, o221ai, o22a, o22ai, o2bb2a, o2bb2ai, o311a, o311ai, o31a, o31ai, o32a, o32ai, o41a, o41ai | 3,208 |
| NAND | nand2, nand2b, nand3, nand3b, nand4, nand4b | 2,495 |
| Flip-Flops | dfrtp | 1,024 |
| AND | and2, and2b, and3, and3b, and4, and4b, and4bb | 338 |
| NOR | nor2, nor3, nor3b, nor4, nor4b | 244 |
| Inverter | inv | 217 |
| OR | or2, or3, or3b, or4, or4b, or4bb | 180 |
| XOR/XNOR | xnor2, xor2 | 38 |
| Multiplexer | mux2, mux4 | 28 |
| Total | | 7772 |

### Critical path

The current worst-case clocked datapath runs from instruction fetch (`instruction[24]`, part of the `rs2` field) through decode/control logic and the register-file address-decode path, to the data memory write port, `data_write[25]`. This is essentially the full single-cycle datapath for a store instruction in one pass. This is architecturally expected for a single-cycle design.

In the future, I would like to revisit this project and optimize this datapath as well as the data forwarding / masking modules to hopefully lower the clock period to 20 ns.


### Known limitation

Max slew and max cap violations remain in several timing corners (`ss`, `tt` process corners), stemming from residual high-fanout nets in the decode path. These didn't fail signoffs for DRC / LVS / timing, but are a real signal-integrity consideration for an actual fabricated chip. Revisiting this properly (like thorough RTL-level restructuring of the decode signal fanout, rather than just tool-level optimization) is planned once I've learned more formal knowledge in this area at uWaterloo.

---

## Repository Structure

```
|
├── rtl/
│   ├── rv32i_core.v        # Top level CPU module
│   ├── data_mem.v          # RAM module
│   ├── instruction_mem.v   # ROM module
│   ├── soc_top.v           # SoC routing CPU with RAM and ROM
│   └── all other .v files  # Individual sub modules inside the CPU (rv32i_core.v)
├──sim/
│   ├── cocotb_sim/         # cocotb simulator (Icarus Verilog) build and temp file dump location (includes top level testbench waveform .fst)
│   ├── *_test              # Individual module intermediate simulation file for Icarus
│   ├── Makefile            # cocotb/Icarus build + run for top module (soc_top.v) testing
│   ├── *.vcd               # individual module tesbench waveforms
│   └── *.gtkw              # GTKWave config files for each testbench's waveform
└── tb/
    ├── modules/            #  per-module SystemVerilog testbenches, constrained-random + golden-model, run via Icarus Verilog
    ├── programs/           #  assembled SoC & instruction memory test program artifacts(.s/.o/.elf/.hex)
    └── top/                #  tb_top_soc.s (self-checking assembly), tb_top_soc.py (cocotb driver)
```

---

## Future Plans

- [ ] Pipelined successor with 5-stage: IF/ID/EX/MEM/WB, hazard detection, forwarding, and branch handling
- [ ] Full-core differential testing against a Python-based golden ISA model (and/or Spike)
- [ ] Official `riscv-arch-test` compliance suite
- [ ] Compiled C program (via RISC-V GCC) run end-to-end as an integration demo
- [ ] FPGA implementation on a AMD Xilinx Artix-7, eventually building toward a VGA-driven SoC
- [ ] Revisit synthesis fanout / slew / critical path optimization at the RTL level

---

**To-scale comparison with my previous project just for fun :)**

<img width="1975" height="1360" alt="chip comparison pic" src="https://github.com/user-attachments/assets/d1d90501-1342-4f46-bd63-779a38494bcc" /> 

---

## License

Apache-2.0, see [LICENSE](LICENSE)

Copyright (c) 2026 Zhiyuan (Jerry) Jiang: design, verification and documentation

All credit for architectural and ISA specification goes to the RISC-V International
