# 5-Stage Pipelined MIPS CPU

A 32-bit, 5-stage pipelined MIPS processor implemented in schematic form Quartus II `.bdf`. The datapath and control logic are built from primitive gates and a small number of LPM megafunctions (memories, register file, adders), with a gate-level main control unit, ALU control unit and ALU output stage.

## Features

- Classic 5-stage pipeline: IF, ID, EX, MEM, WB
- 18 instructions covering arithmetic, logic, shift, memory access, branch and jump
- Full data forwarding (EX/MEM and MEM/WB to EX)
- Load-use hazard detection with a one-cycle stall
- Control hazard handling: jump resolved in ID, branches resolved in MEM with pipeline flush
- Main control (31 gates) and ALU control (33 gates) built from primitive gates
- Gate-level barrel shifter (`sll`, `srl`) and ALU output multiplexer
- Self-checking assembly programs, verified in ModelSim waveform simulation

## Instruction Set

| Type | Instruction | Opcode | Funct | Operation |
|------|-------------|--------|-------|-----------|
| R | `add`  | 000000 | 100000 | rd = rs + rt |
| R | `sub`  | 000000 | 100010 | rd = rs - rt |
| R | `and`  | 000000 | 100100 | rd = rs & rt |
| R | `or`   | 000000 | 100101 | rd = rs \| rt |
| R | `xor`  | 000000 | 100110 | rd = rs ^ rt |
| R | `nor`  | 000000 | 100111 | rd = ~(rs \| rt) |
| R | `slt`  | 000000 | 101010 | rd = (rs < rt) ? 1 : 0 (signed) |
| R | `sll`  | 000000 | 000000 | rd = rt << shamt |
| R | `srl`  | 000000 | 000010 | rd = rt >> shamt (logical) |
| I | `addi` | 001000 | - | rt = rs + sign_ext(imm) |
| I | `andi` | 001100 | - | rt = rs & zero_ext(imm) |
| I | `ori`  | 001101 | - | rt = rs \| zero_ext(imm) |
| I | `slti` | 001010 | - | rt = (rs < sign_ext(imm)) ? 1 : 0 |
| I | `lw`   | 100011 | - | rt = MEM[rs + sign_ext(imm)] |
| I | `sw`   | 101011 | - | MEM[rs + sign_ext(imm)] = rt |
| I | `beq`  | 000100 | - | if (rs == rt) PC = PC + 4 + (imm << 2) |
| I | `bne`  | 000101 | - | if (rs != rt) PC = PC + 4 + (imm << 2) |
| J | `j`    | 000010 | - | PC = {PC+4[31:28], addr, 2'b00} |

The all-zero word decodes as `sll $0, $0, 0`, which acts as a harmless bubble (NOP).

## Pipeline Organization

| Register | Width | Notes |
|----------|-------|-------|
| IF/ID  | 64  | PC+4, instruction |
| ID/EX  | 153 | operands, immediate, register numbers, control signals (3-bit ALUOp) |
| EX/MEM | 106 | ALU result, store data, branch target, Zero flag, control |
| MEM/WB | 71  | memory data, ALU result, destination register, control |

### ALU control

The main control produces a 3-bit `ALUOp`; the ALU control combines it with `funct` to produce a 4-bit operation code.

| ALUCtrl | Operation | ALUCtrl | Operation |
|---------|-----------|---------|-----------|
| 0000 | AND | 0111 | SLT |
| 0001 | OR  | 1000 | SLL |
| 0010 | ADD | 1001 | SRL |
| 0011 | XOR | 0110 | SUB |
| 0100 | NOR |      |     |

`bne` reuses the subtract operation and inverts the Zero flag (`Zero_EX = Zero ^ InvZero`). `andi` and `ori` use a zero-extended immediate, selected by a `ZeroExt` control signal in the sign extender.

## Hazard Handling

- **Data hazards:** a forwarding unit selects the ALU operands from the ID/EX register, the EX/MEM result or the MEM/WB write-back value.
- **Load-use hazard:** the hazard detection unit stalls IF and ID for one cycle and inserts a bubble into EX.
- **Control hazards:** `j` is resolved in ID; `beq` and `bne` are resolved in MEM, so a taken branch flushes the three younger instructions.

## Repository Layout

```
Component/   reusable blocks: ALU, ALU control, main control, shifter, sign extender,
             pipeline registers, forwarding and hazard units, memories
Stages/      IF, ID, EX, MEM. WB is easy, so I put it in Top.bdf
Top/         top-level design (Top.bdf)
testing/     ModelSim vector waveform files (.vwf)
*.mif        memory initialisation files (programs and data)
```

Main files:

- `MIPS_PipeLine.qpf` / `.qsf`: Quartus II project and settings (revision `MIPS_PipeLine`, top-level entity `Top`)
- `program.mif`: system test program that exercises all instruction groups
- `program_isa18.mif` and `program_isa18_listing.txt`: self-checking test for the 18-instruction ISA, with an assembly listing
- `program_forward.mif`, `program_loaduse.mif`, `program_branch.mif`: focused hazard tests
- `dmem.mif`: data memory initial contents

## Getting Started

**Requirements:** Quartus II 13.0.1 SP1 (Web Edition) with ModelSim-Altera.

1. Open `MIPS_PipeLine.qpf` in Quartus II.
2. Run **Processing > Start Compilation**.
3. To run a different program, copy it over the instruction memory initialisation file (`program.mif`) or point the ROM at it, then recompile.
4. Open a `.vwf` file from `testing/` and run **Simulation > Run Functional Simulation**.

## Verification

Each test program is self-checking: every result is compared with the expected value by a `beq` / `j ERR` pair, and the program ends by writing a known marker value to the register file when all checks pass. A failed check jumps to an error loop and the marker is never written.

| Test | Program | Result |
|------|---------|--------|
| System test (10-instruction baseline and regression) | `program.mif` | marker at about 7.05 us |
| 18-instruction ISA test | `program_isa18.mif` | marker at about 3.85 us |

The 18-instruction test covers logic, immediate, `slti`, shift, `bne` and load-use cases. It was also checked against a cycle-accurate software model, which detected 21 of 22 injected design faults (mutants).

## Tools

- Quartus II 13.0.1 SP1
- ModelSim-Altera for functional simulation
- Git and GitHub for version control
