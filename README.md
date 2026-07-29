# Entropic R32-SC

A single-cycle RISC-V (RV32I) CPU, designed and built completely from scratch in Verilog, fully verified through self written testbenches in SystemVerilog and RISC-V Assembly, and synthesized to a physical ASIC layout through OpenLane2 on the SkyWater 130nm open-source PDK.

`R32-SC` = **R**V**32**I, **S**ingle-**C**ycle. First entry in the Entropic core lineup, a pipelined successor is coming next (see [Roadmap](#roadmap)).

---

## Overview

Entropic R32-SC is a complete RV32I implementation built and verified from the ground up as a passion project ahead of starting Electrical Engineering at the University of Waterloo.

It covers the full base instruction set: arithmetic/logic, immediates, loads/stores (including byte/halfword variants), branches, jumps, and upper-immediate instructions.


The core has been:
- Top level module verified through directed, self-checking assembly tests
- Submodule-level verified through both constrained-random testing against golden models as well as directed tests for edge case testing
- Synthesized and physically implemented through the full RTL-to-GDSII flow using [OpenLane2](https://github.com/efabless/openlane2) and the [SKY130 PDK](https://github.com/google/skywater-pdk), ran locally in a Docker + WSL environment

---

## Architecture

`soc_top` wraps the core (`rv32i_core`) together with separate instruction and data memory modules, connected via a clean memory-mapped interface.

**[diagram]**

**Datapath:**
- **Fetch:** Program Counter → Instruction Memory
- **Decode:** Control Unit + Immediate Generator + ALU Control, driven off the fetched instruction
- **Execute:** Register File → ALU (with Branch Comparator running in parallel for branch instructions)
- **Memory:** Data Memory, with a Load Filter (byte/halfword sign- and zero-extension) and Store Mask (byte/halfword write-enable) handling sub-word accesses
- **Writeback:** A 4-to-1 mux selects between ALU result, filtered memory data, `PC+4` (for `jal`/`jalr` link), and the immediate (for `lui`), based on `mem_to_reg`

Everything happens within a single clock cycle. There's no pipelining (yet!), no hazards to resolve, and no forwarding logic. The tradeoff is clock speed: the core's maximum frequency is bounded by its single longest instruction path (see [ASIC Implementation](#asic-implementation) for the critical path).

---

## Instruction Set Coverage

Full RV32I base instruction set — 40/40 instructions implemented.

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

`ECALL`/`EBREAK` currently raise a `halt` signal rather than implementing full trap/CSR machinery, CSR support will be implemented in a future pipelined version.
`FENCE` is currently implemented as `NOP` and will be reworked after pipelining as well.

---

## Design Decisions

A few notable choices worth calling out, since they weren't arbitrary:

- **Active-low, asynchronous reset (`rst_n`)** throughout the design to ensure a stable, consistant reset signal during power-up
- **Combinational register file reads** — `rs1`/`rs2` data is available same-cycle, consistent with a single-cycle architecture where everything must resolve within one clock
- **`x0` hardwired to zero** — writes to `x0` are architecturally discarded, verified explicitly in testing
- **4-bit ALU control encoding**, decoded from opcode/`funct3`/`funct7` via a dedicated `alu_control` module rather than inline in the ALU itself, keeping the ALU's own logic purely arithmetic
- **2-bit `pc_src`** to cleanly distinguish between sequential (`PC+4`), branch, and jump (`jal`/`jalr`) targets in the next-PC mux
- **2-bit `mem_to_reg`** to distinguish between writing ALU result, writing RAM read, writing link address (`PC+4`) for `jal`/`jalr`, and writing directly from `imm_gen` for `lui` to bypass unnecessary data routing
- **Separate `load_filter` / `store_mask` modules** for byte/halfword handling, rather than embedding sign-extension and byte-lane logic directly in `data_mem`. This keeps the memory module itself simple and the addressing logic easily testbench-able
- **`pc.v` broken out as its own module** rather than inlined into `rv32i_core`. This keeps next-PC muxing (sequential/branch/jump) independently testable, same idea as the load/store split above

### Module list (`rtl/`)

Top modules: `soc_top.v` · `rv32i_core.v`

Submodules: `alu.v` · `alu_control.v` · `branch_comp.v` · `control_unit.v` · `data_mem.v` · `imm_gen.v` · `instruction_mem.v` · `load_filter.v` · `pc.v` · `reg_file.v` · `store_mask.v`

---

## Verification

A two-tier verification strategy: module-level constrained-random testing against golden models, plus full-core directed self-checking tests.

### Module-level (golden model + CRT)

Key combinational modules (`alu`, `load_filter`, `store_mask`, `branch_comp`, `imm_gen`, `reg_file`) each have a dedicated SystemVerilog testbench that runs constrained-random testing (10,000+ iterations) against a golden reference model, in addition to directed corner-case tests (sign-extension boundaries, each byte/halfword offset, default fallback cases). Failures report expected vs. actual values directly via `$fatal`.

### Full-core (self-checking assembly)

`tb/tb_top_soc.s` is a self-checking RISC-V assembly program exercising every instruction in the ISA. Each instruction's result is checked against a hand-computed expected value; on failure, a unique error code (2–36) is written to `x10` and the core halts, making failures immediately diagnosable without digging through waveforms.

### Known limitations (honest, documented gaps)

- The `jal`/`jalr` self-check verifies control transfer and link-address correctness together (via a jump-there-and-back pattern), but does **not** specifically exercise `jalr`'s LSB-clearing behavior (`target = (rs1 + imm) & ~1`), since the tested link address is always word-aligned. Flagged here rather than silently claimed as covered.
- Full-core differential testing against a reference ISA simulator (a self-written Python interpreter, and/or Spike) and the official [`riscv-arch-test`](https://github.com/riscv-non-isa/riscv-arch-test) compliance suite are planned but not yet implemented — deferred until after the pipelined core, where the same infrastructure will cover both.

---

## ASIC Implementation

Synthesized end-to-end (RTL → GDSII) using **OpenLane2** against the **SKY130** open-source PDK, run locally via WSL2 + Docker rather than through CI, specifically to work through the toolchain hands-on (synthesis → floorplan → placement → CTS → routing → DRC/LVS → GDS).

The core (`rv32i_core`) was synthesized standalone, independent of the instruction/data memory modules — reflecting how memory is typically implemented as a separate hard macro (SRAM) in real ASIC design rather than synthesized flip-flop arrays.

**[GDS layout screenshot (KLayout, 2D)]**

**[3D rendered layout screenshot]**

### Results

| Metric | Value |
|---|---|
| Clock period | 28 ns (~35.7 MHz) |
| Total cell count | 7,772 |
| Flip-flops | 1,024 *(the full 32×32-bit register file)* |
| DRC | 0 violations |
| LVS | Circuits match |
| Setup timing (signoff) | Met, ~1.2 ns margin |

**Cell breakdown:**

| Category | Count |
|---|---|
| Combinational (AOI/OAI compound gates) | 3,208 |
| NAND | 2,495 |
| Flip-Flops | 1,024 |
| AND | 338 |
| NOR | 244 |
| Inverter | 217 |
| OR | 180 |
| XOR/XNOR | 38 |
| Multiplexer | 28 |

### Critical path

The worst-case timing path runs from instruction fetch (`instruction[15]`, part of the `rs1` field) through decode/control logic and the register-file address-decode path, to the data memory write port — essentially the full single-cycle datapath for a store instruction in one pass. This is architecturally expected for a single-cycle design: the clock period is bounded by the single longest instruction path, with no pipelining to shorten it.

Register-file address decode (`rs1`/`rs2` bit fields) is a naturally high-fanout signal source in this design, which was addressed via `SYNTH_MAX_FANOUT` buffering during synthesis.

### Known limitation

Max slew and max cap violations remain in several timing corners (`ss`, `tt` process corners), stemming from residual high-fanout nets in the decode path. These didn't block DRC/LVS/timing signoff, but are a real signal-integrity consideration for an actual fabricated chip. Revisiting this properly (like thorough RTL-level restructuring of the decode fanout, rather than just tool-level optimization) is planned once I've got more formal training in this area at uWaterloo.

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

## How to Run

### Module-level testbenches (cocotb + Icarus Verilog)
```bash
cd sim/
make  # runs the per-module cocotb tests defined in the Makefile
```

### Module-level golden-model testbenches (SystemVerilog, Vivado xsim)
```bash
cd tb/modules/
# open/run individual testbenches, e.g. tb_load_filter.sv, in Vivado xsim
```

### Full-core self-checking assembly test
```bash
cd tb/top/
# tb_top_soc.s is assembled into tb/programs/, then driven via tb_top_soc.py (cocotb) against tb.v
```

### ASIC flow (OpenLane2 + SKY130)
```bash
python3 -m openlane --dockerized config.json
```
See `rtl/` for source files and the synthesis config used for the standalone core run described above.

---

## Roadmap

- [ ] Pipelined successor — **Entropic R32-P** (5-stage: IF/ID/EX/MEM/WB), with hazard detection, forwarding, and branch handling
- [ ] Full-core differential testing against a Python-based golden ISA model (and/or Spike)
- [ ] Official `riscv-arch-test` compliance suite
- [ ] Compiled C program (via RISC-V GCC) run end-to-end as an integration demo
- [ ] FPGA implementation on a AMD Xilinx Artix-7, eventually building toward a VGA-driven SoC
- [ ] Revisit synthesis fanout/slew optimization at the RTL level

---

## License

Apache-2.0 — see [LICENSE](LICENSE).


Copyright (c) 2026 Zhiyuan (Jerry) Jiang — design, verification and documentation
