# A 32-bit, 5-stage pipelined MIPS processor implementing a selected subset of the MIPS instruction set.

The processor currently supports 10 instructions and focuses on demonstrating fundamental pipelining, data forwarding, hazard detection, and control hazard handling.

## Features

- Classic 5-stage pipeline: Instruction Fetch, Decode, Execute, Memory, Write-Back
- 32-bit datapath with a 32-entry register file (write-through, so a same-cycle read sees the new value)
- Data forwarding from the EX/MEM and MEM/WB stages to the ALU inputs
- Hazard detection unit that inserts a one-cycle stall for load-use dependencies
- Branch and jump handling with selective pipeline flushes
- On-chip memories built from M4K blocks: 128-word instruction memory and 64-word data memory
- Self-checking assembly test program with a pass/fail signature

## Instruction Set

| Instruction | Format | Opcode | Funct |
|---|---|---|---|
| `add`  | R | `000000` | `100000` |
| `sub`  | R | `000000` | `100010` |
| `and`  | R | `000000` | `100100` |
| `or`   | R | `000000` | `100101` |
| `slt`  | R | `000000` | `101010` |
| `lw`   | I | `100011` | - |
| `sw`   | I | `101011` | - |
| `beq`  | I | `000100` | - |
| `addi` | I | `001000` | - |
| `j`    | J | `000010` | - |

## Pipeline Organization

| Stage | Function |
|---|---|
| IF  | Fetch instruction from instruction memory, compute PC + 4 |
| ID  | Decode, read registers, sign-extend immediate, generate control signals, resolve jumps |
| EX  | ALU operation, forwarding muxes, branch target calculation |
| MEM | Data memory access, branch decision |
| WB  | Select ALU result or loaded data, write the register file |

## Hazard Handling

| Hazard | Mechanism |
|---|---|
| RAW, producer in EX/MEM or MEM/WB | Forwarding unit selects the bypassed value for each ALU operand |
| RAW, producer three instructions ahead | Register file write-through |
| Load-use | Hazard detection unit freezes PC and IF/ID and inserts a bubble into ID/EX for one cycle |
| Taken `beq` (resolved in MEM) | Flush IF/ID, ID/EX and EX/MEM; branch has priority over a concurrent jump |
| `j` (resolved in ID) | Flush IF/ID; suppressed while the pipeline is stalled |

## Repository Layout
Component/ Reusable building blocks: ALU, register file, control units,
Forwarding and Hazard units, Pipeline registers
Stages/ Per-stage datapath schematics (IF, ID, EX, MEM, WB)
Top/ Top-level design integrating all stages
testing/ Simulation waveforms and test setups
MIPS_PipeLine.qpf Quartus project file
MIPS_PipeLine.qsf Quartus project settings
program.mif Instruction memory image (self-checking system test)
program_listing.txt Annotated listing of program.mif
program_forward.mif Focused test: data forwarding
program_loaduse.mif Focused test: load-use stall
program_branch.mif Focused test: branch and jump flush
dmem.mif Data memory initialization

## Getting Started

**Requirements:** Quartus II 13.0 SP1 
1. Open `MIPS_PipeLine.qpf` in Quartus II.
2. Select the `MIPS_PipeLine` revision and run **Processing > Start Compilation**.
3. Open a waveform file from `testing/` and run the simulation. The testbench uses a 20 ns clock with reset asserted for the first 20 ns.

To run a focused test instead of the full system test, copy the desired `.mif` file over `program.mif`, recompile, and simulate again.

## Verification

`program.mif` is a self-checking program of 126 instructions that exercises every instruction, all forwarding paths and priorities, load-use stalls, branch and jump flushes, a backward loop, and a signed-overflow `slt` case. Each check branches to a failure handler on mismatch.

- **Pass:** the program writes `0x0000600D` to `$2` and settles into the `j` loop at word `0x7D`.
- **Fail:** the failure handler loads `0x00000BAD` into `$2` and loops at word `0x7F`.

In simulation the pass signature appears on `WriteData_WB` at about 7.05 us.

## Tools

Quartus II 13.0 SP1.
