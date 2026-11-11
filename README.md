# KGP-miniRISC: 32-Bit Single-Cycle RISC Processor

A 32-bit single-cycle Reduced Instruction Set Computer (RISC) processor designed and implemented from scratch in **Verilog HDL** as part of the Computer Organization and Architecture (CS39001) Laboratory at **IIT Kharagpur**.

## Overview

**KGP-miniRISC** features a custom 32-bit Instruction Set Architecture (ISA) supporting 23 instructions across arithmetic, logical, shift, memory access, conditional/unconditional branching, and specialized bit-manipulation operations. The processor uses a **32-bit word size**, a **32 × 32-bit general-purpose register file**, and **32-bit memory addressing** backed by Xilinx Block RAM (BRAM) IP cores for separate instruction and data memories (Harvard-style split memory architecture).

### Key Highlights
- **Custom 32-bit ISA**: 23 instructions encoded across 4 instruction formats (R-Format, I-Format, B1-Format, B2-Format) with room for ISA expansion.
- **Hierarchical Carry Lookahead Adder (CLA)**: Fast 32-bit addition and 2's complement computation built using 4-bit CLA blocks and 16-bit Lookahead Carry Units (LCU).
- **Hardware `diff` Instruction**: Dedicated combinational logic (`x ^ y` coupled with `x & ~(x - 1)` and a 32-bit lookup decoder) to compute the index of the least significant bit where two registers differ in a single cycle.
- **Single-Cycle Datapath & Hardwired Control Unit**: Purely combinational control unit generating 9 control signals (`ALUop`, `ALUSrc`, `RegDst`, `RegWrite`, `Jump`, `Branch`, `MemRead`, `MemWrite`, `MemToReg`) alongside a dedicated Branch Control unit and D-Flip-Flop carry flag register.
- **End-to-End Verification**: Tested via unit testbenches for individual hardware modules and verified at the system level by executing **Bubble Sort** and **Linear Search** machine-code programs loaded into BRAM via `.coe` memory initialization files, plus FPGA board deployment support.

---

## Instruction Set Architecture (ISA)

All instructions, registers, and memory words are **32 bits** wide.

### Instruction Formats

1. **R-Format (Register)** — Used for register-based arithmetic, logic, shift, and `diff` operations:
   ```
   +----------+--------+--------+------------+--------+--------+
   |  Opcode  |   rs   |   rt   | Don't Care | shamt  |  func  |
   |  6 bits  | 5 bits | 5 bits |   5 bits   | 5 bits | 6 bits |
   +----------+--------+--------+------------+--------+--------+
    31      26 25    21 20    16 15        11 10     6 5      0
   ```

2. **I-Format (Immediate)** — Used for immediate arithmetic (`addi`, `compi`) and memory access (`lw`, `sw`):
   ```
   +----------+--------+--------+------------------------------+
   |  Opcode  |   rs   |   rt   |          Immediate           |
   |  6 bits  | 5 bits | 5 bits |           16 bits            |
   +----------+--------+--------+------------------------------+
    31      26 25    21 20    16 15                           0
   ```

3. **B1-Format (Register-based Branch)** — Used for conditional register branches (`bltz`, `bz`, `bnz`) and register jump (`br`):
   ```
   +----------+--------+--------+------------------------------+
   |  Opcode  |   rs   |  X     |      Immediate / Offset      |
   |  6 bits  | 5 bits | 5 bits |           16 bits            |
   +----------+--------+--------+------------------------------+
    31      26 25    21 20    16 15                           0
   ```

4. **B2-Format (Unconditional & Flag-based Branch)** — Used for unconditional jumps (`b`, `bl`) and carry-flag branches (`bcy`, `bncy`):
   ```
   +----------+------------------------------------------------+
   |  Opcode  |                Label / Target                  |
   |  6 bits  |                    26 bits                     |
   +----------+------------------------------------------------+
    31      26 25                                             0
   ```

---

### Supported Instructions

| Class | Instruction | Usage | Operation / Meaning | Format | Opcode | Func |
| :--- | :--- | :--- | :--- | :---: | :---: | :---: |
| **Arithmetic** | Add | `add rs, rt` | `rs <- (rs) + (rt)` | R | `000000` | `000000` |
| | Complement | `comp rs, rt` | `rs <- 2's complement of (rt)` | R | `000000` | `000001` |
| | Add Immediate | `addi rs, imm` | `rs <- (rs) + sign_ext(imm)` | I | `000001` | — |
| | Complement Immediate | `compi rs, imm` | `rs <- 2's complement of sign_ext(imm)` | I | `000010` | — |
| **Logic** | Bitwise AND | `and rs, rt` | `rs <- (rs) & (rt)` | R | `000011` | `000000` |
| | Bitwise XOR | `xor rs, rt` | `rs <- (rs) ^ (rt)` | R | `000011` | `000001` |
| **Shift** | Shift Left Logical | `shll rs, sh` | `rs <- (rs) << sh` | R | `000100` | `000000` |
| | Shift Right Logical | `shrl rs, sh` | `rs <- (rs) >> sh` | R | `000100` | `000001` |
| | Shift Left Logical Var | `shllv rs, rt` | `rs <- (rs) << (rt)` | R | `000100` | `000010` |
| | Shift Right Logical Var | `shrlv rs, rt` | `rs <- (rs) >> (rt)` | R | `000100` | `000011` |
| | Shift Right Arithmetic | `shra rs, sh` | `rs <- (rs) >>> sh` | R | `000100` | `000100` |
| | Shift Right Arith Var | `shrav rs, rt` | `rs <- (rs) >>> (rt)` | R | `000100` | `000101` |
| **Memory** | Load Word | `lw rt, imm(rs)` | `rt <- mem[(rs) + sign_ext(imm)]` | I | `000101` | — |
| | Store Word | `sw rt, imm(rs)` | `mem[(rs) + sign_ext(imm)] <- (rt)` | I | `000110` | — |
| **Branch** | Branch Less Than Zero | `bltz rs, L` | `if (rs) < 0 then PC <- L` | B1 | `000111` | — |
| | Branch Zero | `bz rs, L` | `if (rs) == 0 then PC <- L` | B1 | `001000` | — |
| | Branch Not Zero | `bnz rs, L` | `if (rs) != 0 then PC <- L` | B1 | `001001` | — |
| | Branch Register | `br rs` | `PC <- (rs)` | B1 | `001010` | — |
| | Unconditional Branch | `b L` | `PC <- L` | B2 | `001011` | — |
| | Branch and Link | `bl L` | `$ra (reg 31) <- PC + 1; PC <- L` | B2 | `001100` | — |
| | Branch on Carry | `bcy L` | `if (Carry == 1) then PC <- L` | B2 | `001101` | — |
| | Branch on No Carry | `bncy L` | `if (Carry == 0) then PC <- L` | B2 | `001110` | — |
| **Complex** | Bit Difference | `diff rs, rt` | `rs <- LSB bit index where (rs) and (rt) differ` | R | `001111` | `000000` |

---

## Processor Architecture & Solution Design

### 1. Datapath (`datapath.v`)
The single-cycle datapath integrates instruction fetch, decode, register read, ALU execution, data memory access, and write-back:
- **Program Counter (`program_counter.v`, `pc_increment.v`)**: Holds the current instruction address and computes `nextPC = PC + 1` (word-addressed BRAM).
- **Instruction Memory (`bram_instr_memory`)**: Xilinx Single-Port Block ROM initialized with 32-bit machine instructions via `.coe` files.
- **Instruction Decoder (`instruction_decoder.v`)**: Slices the 32-bit instruction into `opcode`, `func`, `rs`, `rt`, `imm` (16-bit), and `jump_addr` (26-bit). Note that for constant shifts (`shll`, `shrl`, `shra`), the shift amount is passed via the immediate/shamt field.
- **Register File (`register_file.v`)**: 32 registers of 32 bits each (`registers[0:31]`), supporting asynchronous dual-port reads (`readData1`, `readData2`) and synchronous single-port writes on the rising clock edge when `RegWrite` is asserted. Register `31` (`$ra`) is dedicated to storing the return address during `bl` (Branch and Link).
- **Data Memory (`bram_data_memory`)**: Xilinx Single-Port Block RAM clocked on `~clk` so that memory reads (`lw`) and writes (`sw`) complete within the same instruction cycle. Includes an interactive button-driven memory inspection counter for reading out array results on FPGA LEDs.

### 2. Arithmetic Logic Unit (`ALU.v`)
The ALU takes two 32-bit signed inputs and a 5-bit `ALUop` control code, producing a 32-bit `result` along with `carry`, `zero_flag`, and `sign_bit` status flags:
- **32-Bit Carry Lookahead Adder (`adder_32_bit.v`, `cla_16_bit_LCU.v`, `cla_4_bit.v`, `lookahead_carry_unit.v`)**: Performs `add`, `addi`, `lw`/`sw` address calculation, and 2's complement (`~input2 + 1` for `comp` and `compi`).
- **Barrel Shifter (`shift_module.v`)**: Handles logical left (`<<`), logical right (`>>`), and arithmetic right (`>>>`) shifts by either a constant immediate or a variable register value.
- **Hardware `diff` Engine (`ALU.v`, `diff_lookup.v`)**:
  - Computes `diff_temp = input1 ^ input2` to identify all differing bit positions.
  - Isolates the least significant set bit using `res_diff_0 = diff_temp & ~(diff_temp - 1)` (computed via a dedicated 32-bit CLA).
  - Decodes the one-hot 32-bit value into its bit index (`0` to `31`, or `32` if identical) via `diff_lookup.v`.
- **Carry Persistence (`dff.v`)**: A D-Flip-Flop latches the ALU `carry` flag across clock cycles so `bcy` and `bncy` can branch based on the carry generated by the preceding arithmetic instruction.

### 3. Control Unit & Branch/Jump Resolution (`control_unit.v`, `branch_control.v`, `jump_control.v`)
- **Main Control Unit (`control_unit.v`)**: Purely combinational decoder mapping the 6-bit `opcode` and 6-bit `func` to 9 control signals:
  - `RegDst [1:0]`: Selects destination register (`00` -> `rs`, `01` -> `rt` for `lw`, `10` -> `$ra` for `bl`).
  - `ALUSrc`: Selects second ALU operand (`0` -> `readData2`, `1` -> sign-extended 16-bit immediate).
  - `MemToReg [1:0]`: Selects register write-back source (`00` -> `PC + 1` for `bl`, `01` -> `mem_read_data` for `lw`, `10` -> ALU `result`).
  - `Jump [1:0]`: Selects next PC source (`00` -> 26-bit jump target, `01` -> PC-relative branch or `PC + 1`, `10` -> register `rs` for `br`).
  - `ALUop [4:0]`, `RegWrite`, `MemRead`, `MemWrite`, `Branch`.
- **Branch & Jump Control (`branch_control.v`, `jump_control.v`)**: Evaluates branch conditions (`bltz`, `bz`, `bnz`, `bcy`, `bncy`, `b`, `bl`, `br`) against `sign`, `zero_flag`, and `prev_carry` to select the next instruction address.

---

## Repository Structure

```text
.
├── ISA22.pdf                           # Lab assignment problem statement & ISA specification
├── COA_Grp_63_RISC_Documentation.pdf   # Detailed design report (encoding, datapath diagram, control truth table)
├── test-instructions/                  # BRAM initialization (.coe) files for verification programs
│   ├── bubble_sort_instr.coe           # Machine code for Bubble Sort algorithm
│   ├── bubble_sort_data.coe            # Initial 10-element signed integer array for Bubble Sort
│   ├── linear_search_instr.coe         # Machine code for Linear Search algorithm
│   ├── linear_search_found_data.coe    # Test array where search key is present
│   └── linear_search_notfound_data.coe # Test array where search key is absent
└── verilog-files/                      # Verilog HDL source modules & testbenches
    ├── KGPminiRISC.v                   # Top-level processor module (Control Unit + Datapath)
    ├── KGP_miniRISC_wrapper.v          # Synthesis wrapper for FPGA deployment
    ├── datapath.v                      # Complete single-cycle processor datapath
    ├── control_unit.v                  # Combinational controller generating 9 control signals
    ├── ALU.v                           # 32-bit Arithmetic Logic Unit
    ├── adder_32_bit.v                  # 32-bit hierarchical Carry Lookahead Adder
    ├── cla_16_bit_LCU.v                # 16-bit CLA block using 4-bit CLAs and LCU
    ├── cla_4_bit.v                     # 4-bit Carry Lookahead Adder
    ├── lookahead_carry_unit.v          # Lookahead Carry Unit (LCU)
    ├── shift_module.v                  # 32-bit logical & arithmetic shifter
    ├── diff_lookup.v                   # One-hot to bit-index decoder for the `diff` instruction
    ├── register_file.v                 # 32 x 32-bit register file
    ├── instruction_decoder.v           # 32-bit instruction field extractor
    ├── immediate_sign_extend.v         # 16-bit to 32-bit sign extender
    ├── branch_control.v                # Condition evaluator for branch instructions
    ├── jump_control.v                  # Next-PC multiplexing logic for jumps and branches
    ├── program_counter.v               # 32-bit Program Counter register
    ├── pc_increment.v                  # PC + 1 incrementer
    ├── dff.v                           # D-Flip-Flop for latching the carry flag
    ├── clock_divider.v                 # Clock divider for FPGA board testing
    ├── mux_5b_3_1.v                    # 5-bit 3:1 multiplexer (write register selection)
    ├── mux_32b_2_1.v                   # 32-bit 2:1 multiplexer
    ├── mux_32b_3_1.v                   # 32-bit 3:1 multiplexer
    └── *_tb.v                          # Unit and full-system Verilog testbenches
```

---

## Testing & Verification

### 1. Algorithmic Benchmarks (`.coe` Programs)
- **Bubble Sort (`bubble_sort_instr.coe`, `bubble_sort_data.coe`)**: Loads a 10-element signed 32-bit array (`[-17, 150, 68, 49, 22, 11, 0, -17, -39, 68]`) from data memory, sorts it in-place in ascending order using nested loops (`lw`, `sw`, `add`, `comp`, `bltz`, `b`, `bl`), and writes the sorted array back to Data BRAM.
- **Linear Search (`linear_search_instr.coe`, `linear_search_found_data.coe`, `linear_search_notfound_data.coe`)**: Searches for a target key across a 10-element array in Data BRAM using the `xor`/`bz` instructions and stores the found index (or `-1` if not found) in register `19`.

### 2. How to Run in Xilinx ISE
1. Create a new project in **Xilinx ISE Design Suite** (targeting e.g. Nexys4 DDR / Artix-7 `xc7a100t-csg324`).
2. Add all Verilog files from `verilog-files/` to the project.
3. Generate two Xilinx **Block Memory Generator (v7.3)** IP cores matching the instantiations in `datapath.v`:
   - `bram_instr_memory`: Single-Port ROM, 32-bit width, 1024 depth (10-bit address), initialized with `test-instructions/bubble_sort_instr.coe` (or `linear_search_instr.coe`).
   - `bram_data_memory`: Single-Port RAM, 32-bit width, 1024 depth (10-bit address), initialized with `test-instructions/bubble_sort_data.coe` (or `linear_search_*_data.coe`).
4. Run behavioral simulation on `KGPminiRISC_tb.v` using **ISim** to observe register states and sorted memory contents.

---

## Authors (Group 63)
- **Shreyas Jena** (20CS30049)
- **Pranav Nyati** (20CS30037)
