# Entropic R32-SC

A single-cycle RISC-V (RV32I) CPU, designed and built completely from scratch in Verilog, fully verified through self written testbenches in SystemVerilog and RISC-V Assembly, and synthesized to a physical ASIC layout through OpenLane2 on the SkyWater 130nm open-source PDK.

`R32-SC` = **R**V**32**I, **S**ingle-**C**ycle. First entry in the Entropic core lineup — the pipelined successor is next (see [Roadmap](#roadmap)).

---

## Overview

Entropic R32-SC is a complete RV32I implementation covering the full base instruction set — arithmetic/logic, immediates, loads/stores (including byte/halfword variants), branches, jumps, and upper-immediate instructions — built and verified from the ground up as a personal project ahead of starting Electrical Engineering at the University of Waterloo.

The core has been:
- Verified through directed, self-checking assembly tests and module-level constrained-random testing against golden models
- Synthesized and physically implemented through the full RTL-to-GDSII flow using [OpenLane2](https://github.com/efabless/openlane2) and the [SKY130 PDK](https://github.com/google/skywater-pdk), run locally

---

## Architecture

`soc_top` wraps the core (`rv32i_core`) together with separate instruction and data memory modules, connected via a clean memory-mapped interface.

**[ Architecture / microarchitecture diagram — insert here ]**

**Datapath at a glance:**
- **Fetch:** Program Counter → Instruction Memory
- **Decode:** Control Unit + Immediate Generator + ALU Control, driven off the fetched instruction
- **Execute:** Register File → ALU (with Branch Comparator running in parallel for branch instructions)
- **Memory:** Data Memory, with a Load Filter (byte/halfword sign- and zero-extension) and Store Mask (byte/halfword write-enable) handling sub-word accesses
- **Writeback:** A 4-to-1 mux selects between ALU result, filtered memory data, `PC+4` (for `jal`/`jalr` link), and the immediate (for `lui`), based on `mem_to_reg`

Everything happens within a single clock cycle — there's no pipelining, no hazards to resolve, and no forwarding logic. The tradeoff is clock speed: the core's maximum frequency is bounded by its single longest instruction path (see [ASIC Implementation](#asic-implementation) for the actual measured critical path).

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

`ECALL`/`EBREAK` currently raise a `halt` signal rather than implementing full trap/CSR machinery — CSR support is deferred to the pipelined successor.

---

## Design Decisions

A few notable choices worth calling out, since they weren't arbitrary:

- **Active-low, asynchronous reset (`rst_n`)** throughout the design
- **Combinational register file reads** — `rs1`/`rs2` data is available same-cycle, consistent with a single-cycle architecture where everything must resolve within one clock
- **`x0` hardwired to zero** — writes to `x0` are architecturally discarded, verified explicitly in testing
- **4-bit ALU control encoding**, decoded from opcode/`funct3`/`funct7` via a dedicated `alu_control` module rather than inline in the ALU itself, keeping the ALU's own logic purely arithmetic
- **2-bit `pc_src`** to cleanly distinguish between sequential (`PC+4`), branch, and jump (`jal`/`jalr`) targets in the next-PC mux
- **Separate `load_filter` / `store_mask` modules** for byte/halfword handling, rather than embedding sign-extension and byte-lane logic directly in `data_mem` — keeps the memory module itself simple and the addressing logic independently testable
- **`pc.v` broken out as its own module** rather than inlined into `rv32i_core` — keeps next-PC muxing (sequential/branch/jump) independently testable, same rationale as the load/store split above

### Module list (`rtl/`)

`alu.v` · `alu_control.v` · `branch_comp.v` · `control_unit.v` · `data_mem.v` · `imm_gen.v` · `instruction_mem.v` · `load_filter.v` · `pc.v` · `reg_file.v` · `rv32i_core.v` · `soc_top.v` · `store_mask.v`

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

**[ GDS layout screenshot (KLayout, 2D) — insert here ]**

**[ 3D rendered layout screenshot — insert here ]**

### Results

| Metric | Value |
|---|---|
| Clock period | 28 ns (~35.7 MHz) |
| Total cell count | 7,772 |
| Flip-flops | 1,024 *(the full 32×32-bit register file)* |
| DRC | 0 violations ✅ |
| LVS | Circuits match ✅ |
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
rtl/            RTL source (see Module list above)
sim/            Per-module cocotb + Icarus Verilog testbenches
  cocotb_sim/       cocotb test infrastructure
  *_test            individual module test scripts (alu_test, control_unit_test, etc.)
  Makefile          cocotb/Icarus build + run
tb/             SystemVerilog golden-model testbenches + full-core assembly test
  modules/          per-module SV testbenches (tb_alu.sv, tb_load_filter.sv, etc.),
                     constrained-random + golden-model, run via Vivado xsim
  programs/         assembled full-core test program artifacts (.s/.o/.elf/.hex)
  top/              tb_top_soc.s (self-checking assembly), tb_top_soc.py (cocotb driver),
                     tb.v (top-level testbench wrapper)
```

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
- [ ] `jalr` LSB-clear directed test
- [ ] Full-core differential testing against a Python-based golden ISA model (and/or Spike)
- [ ] Official `riscv-arch-test` compliance suite
- [ ] Compiled C program (via RISC-V GCC toolchain) run end-to-end as an integration demo
- [ ] FPGA bring-up on a Basys3 (Artix-7), eventually building toward a VGA-driven SoC
- [ ] Revisit synthesis fanout/slew optimization at the RTL level

---

## License

Apache-2.0 — see [LICENSE](LICENSE).


Copyright (c) 2026 Zhiyuan (Jerry) Jiang — design, verification and documentation
