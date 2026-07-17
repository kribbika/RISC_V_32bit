# RV32I Single-Cycle RISC-V Processor (SystemVerilog)

A single-cycle implementation of the RISC-V RV32I base integer instruction set, written in SystemVerilog. The design is fully modular — fetch, decode, register file, ALU, branch control, control unit, and data memory are each separate, independently testable blocks connected together in a top-level datapath.

## Features

- Full **RV32I** base ISA support: R-type, I-type (ALU + loads), S-type, B-type, U-type (`LUI`/`AUIPC`), and J-type (`JAL`/`JALR`) instructions
- Single-cycle (non-pipelined) datapath — one instruction completes per clock cycle
- Byte-addressable data memory with configurable access size (byte/half-word/word) and sign/zero-extension for loads
- Parameterized instruction and data memories (depth/width configurable)
- Centralized `risc_pkg` package defining opcodes, ALU operations, funct3/funct7 mappings, and control signal types — used across all modules for consistency
- Self-checking testbench with clock generation, reset sequencing, and cycle-limited simulation

## Architecture
<img width="646" height="367" alt="image" src="https://github.com/user-attachments/assets/5c7b0148-de56-4b4c-8310-0fa6bee172ad" />



The next PC is computed combinationally each cycle from the current PC, branch/jump resolution (`branch_taken` / `pc_sel`), and the ALU result (used as the jump/branch target), then registered on the next clock edge.

## Module Overview

| Module | File | Description |
|---|---|---|
| `top` | `top.sv` | Top-level module; instantiates and wires together all datapath and control blocks, and implements PC update logic |
| `risc_pkg` | `risc_pkg.sv` | SystemVerilog package with opcode/funct3/funct7 enums, ALU operation types, memory size types, and the control signal struct |
| `instruction_memory` | `instruction_memory.sv` | Parameterized byte-addressable ROM, loaded from `machine_code.mem` via `$readmemh` |
| `fetch` | `fetch.sv` | Drives the instruction memory request/address and captures the fetched instruction |
| `decode` | `decode.sv` | Extracts opcode, funct3/funct7, register addresses, instruction-type flags, and the sign-extended immediate |
| `register_file` | `register_file.sv` | 32×32-bit register file with two async read ports and one synchronous write port (x0 hardwired to zero) |
| `control` | `control.sv` | Main control unit; decodes instruction type/opcode/funct3/funct7 into ALU operation, operand selects, memory control, and writeback source signals |
| `branch_control` | `branch_control.sv` | Evaluates branch conditions (`BEQ`, `BNE`, `BLT`, `BGE`, `BLTU`, `BGEU`) and produces `branch_taken` |
| `alu` | `alu.sv` | Arithmetic/logic unit supporting ADD, SUB, SLL, SLT, SLTU, XOR, SRL, SRA, OR, AND |
| `data_memory` | `data_memory.sv` | Byte-addressable data RAM with configurable access width and zero/sign-extension on reads |
| `test_bench` | `test_bench.sv` | Instantiates `top`, generates the clock, applies reset, and runs the simulation for a fixed number of cycles |

## Instruction Set Coverage

Implemented via `risc_pkg`, the design supports the standard RV32I opcode map:

- **R-type** (`0x33`): ADD, SUB, SLL, SLT, SLTU, XOR, SRL, SRA, OR, AND
- **I-type ALU** (`0x13`): ADDI, SLTI, SLTIU, XORI, ORI, ANDI, SLLI, SRLI, SRAI
- **I-type Load** (`0x03`): LB, LH, LW, LBU, LHU
- **I-type JALR** (`0x67`)
- **S-type** (`0x23`): SB, SH, SW
- **B-type** (`0x63`): BEQ, BNE, BLT, BGE, BLTU, BGEU
- **U-type**: LUI (`0x37`), AUIPC (`0x17`)
- **J-type**: JAL (`0x6F`)

## Getting Started

### Prerequisites

Any SystemVerilog-capable simulator, e.g.:
- [Icarus Verilog](http://iverilog.icarus.com/) (`iverilog` + `vvp`)
- [Verilator](https://www.veripool.org/verilator/)
- Vivado / Questa / VCS

### Running the Simulation

<img width="911" height="336" alt="image" src="https://github.com/user-attachments/assets/4fd9b9ab-21b7-4388-ac41-891739e2a5a1" />
<img width="890" height="98" alt="image" src="https://github.com/user-attachments/assets/52ac6789-7b52-449c-850d-3bbe8193e0e8" />





# Compile
iverilog -g2012 -o sim.out \
    risc_pkg.sv alu.sv branch_control.sv control.sv \
    data_memory.sv decode.sv fetch.sv instruction_memory.sv \
    register_file.sv top.sv test_bench.sv

# Run
vvp sim.out
'''

> **Note:** `instruction_memory.sv` loads program contents from `machine_code.mem` via `$readmemh`. Make sure `machine_code.mem` is in the simulator's working directory (or update the path in `instruction_memory.sv`).

### Program Memory Format

`machine_code.mem` contains hex-encoded instruction bytes (one byte per line/entry, big-endian word order), consumed directly by `$readmemh` into the instruction ROM.

### Waveform Debugging

Add a `$dumpfile` / `$dumpvars` block to `test_bench.sv` to generate a VCD for waveform inspection in GTKWave or your simulator's waveform viewer:

```systemverilog
initial begin
    $dumpfile("dump.vcd");
    $dumpvars(0, test_bench);
end
```

## Project Structure

```
.
├── risc_pkg.sv            # Shared types, enums, and control struct
├── top.sv                 # Top-level datapath
├── fetch.sv                # Fetch stage
├── decode.sv               # Decode stage
├── register_file.sv        # Register file
├── control.sv               # Main control unit
├── branch_control.sv        # Branch condition evaluation
├── alu.sv                   # Arithmetic/logic unit
├── instruction_memory.sv    # Instruction ROM
├── data_memory.sv           # Data RAM
├── machine_code.mem         # Program image (hex)
└── test_bench.sv            # Testbench / simulation driver
```

## Design Notes

- This is a **single-cycle** design: every instruction executes fully within one clock period, so there is no pipelining, hazard detection, or forwarding logic.
- The PC update path selects between the sequential `pc + 4` and a computed target (`branch_taken` or `pc_sel`, e.g. for `JAL`/`JALR`), with the target's LSB forced to `0` per the RISC-V spec.
- All control decisions (ALU operation, operand muxing, memory access, and register writeback source) are derived combinationally from the decoded instruction in `control.sv`.

