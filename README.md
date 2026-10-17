# Multicycle RISC-V Processor – Custom Instruction Extension

## Overview
This repository contains a SystemVerilog implementation of a 32-bit Multicycle RISC-V processor[span_32](start_span)[span_32](end_span)[span_33](start_span)[span_33](end_span). The baseline processor architecture supports core RV32I instructions (`LW`, `SW`, R-type ALU operations, `BEQ`, and `JAL`) and was architected and extended to support a custom multi-cycle immediate instruction (`addi` with bitwise inversion)[span_34](start_span)[span_34](end_span)[span_35](start_span)[span_35](end_span)[span_36](start_span)[span_36](end_span).

The custom instruction (`Opcode: 0010011`, `Funct3: 000`) executes the following compound operation directly in hardware across consecutive clock cycles[span_37](start_span)[span_37](end_span)[span_38](start_span)[span_38](end_span):

$$R[rd] = (R[rs1] + \text{SignExt}(imm[11:0])) \oplus \text{0xFFFFFFFF}$$

Implementing this instruction without introducing additional general-purpose registers required structural modifications to the Datapath (`rv_dp.sv`) and expanding the Mealy Control Finite State Machine (`rv_ctl.sv`)[span_39](start_span)[span_39](end_span)[span_40](start_span)[span_40](end_span)[span_41](start_span)[span_41](end_span).

---

## Architectural Modifications

### 1. Datapath Upgrades (`rv_dp.sv` & `params.inc`)
To compute sequential arithmetic (`ADD`) and logical (`XOR`) operations using a single ALU across back-to-back cycles, two hardware routing paths were added[span_42](start_span)[span_42](end_span)[span_43](start_span)[span_43](end_span):
* **ALU Output Feedback Path (`ALUA_ALUOUT`):** Expanded the ALU `A` input multiplexer (`asel`) to accept the latched `aluout` register value from the previous cycle (`2'b10`), feeding the intermediate addition result directly back into the ALU[span_44](start_span)[span_44](end_span)[span_45](start_span)[span_45](end_span)[span_46](start_span)[span_46](end_span).
* **Hardcoded Inversion Constant (`ALUB_1`):** Expanded the ALU `B` input multiplexer (`bsel`) with a dedicated constant input (`32'hffff_ffff`, selected via `2'b10`) to perform the bitwise XOR inversion[span_47](start_span)[span_47](end_span)[span_48](start_span)[span_48](end_span)[span_49](start_span)[span_49](end_span).

![Modified Datapath Architecture](images/datapath.png)

### 2. Control FSM Expansion (`rv_ctl.sv`)
Three dedicated states were integrated into the Mealy Control FSM following the `DECODE` stage whenever `opcode_funct3` matches `I_TYPE` (`10'b0010011_000`)[span_50](start_span)[span_50](end_span)[span_51](start_span)[span_51](end_span)[span_52](start_span)[span_52](end_span):
1. **`ITYPE_ADD_EXEC` (Cycle 3):** Extracts and sign-extends the 12-bit immediate (`immsel = IMM_L`), routes `rs1` (`asel = ALUA_REG`) and the immediate (`bsel = ALUB_IMM`) into the ALU, and executes `ALU_ADD`[span_53](start_span)[span_53](end_span)[span_54](start_span)[span_54](end_span)[span_55](start_span)[span_55](end_span). The intermediate sum is latched into `aluout` while `regwen` remains `0`[span_56](start_span)[span_56](end_span).
2. **`ITYPE_XOR_EXEC` (Cycle 4):** Routes the latched intermediate sum (`asel = ALUA_ALUOUT`) and the all-ones constant (`bsel = ALUB_1`) into the ALU, executing `ALU_XOR`[span_57](start_span)[span_57](end_span)[span_58](start_span)[span_58](end_span)[span_59](start_span)[span_59](end_span). The final result is latched into `aluout`[span_60](start_span)[span_60](end_span).
3. **`ITYPE_WB` (Cycle 5):** Routes `aluout` to the register file write-back bus (`wbsel = WB_ALUOUT`) and asserts `regwen = 1'b1` to store the result in `rd` before transitioning back to `FETCH`[span_61](start_span)[span_61](end_span)[span_62](start_span)[span_62](end_span).

![Control FSM Diagram](images/fsm.png)

---

## Repository Structure

| File | Description |
| :--- | :--- |
| `rv_top.sv` | Top-level processor module connecting the Control FSM and Datapath[span_63](start_span)[span_63](end_span)[span_64](start_span)[span_64](end_span). |
| `rv_dp.sv` | 32-bit Multicycle Datapath (PC, Register File, Immediate Generator, ALU, muxes)[span_65](start_span)[span_65](end_span)[span_66](start_span)[span_66](end_span). |
| `rv_ctl.sv` | Mealy Control Finite State Machine managing datapath control signals and state transitions[span_67](start_span)[span_67](end_span)[span_68](start_span)[span_68](end_span). |
| `params.inc` | Architecture parameters, instruction opcodes, ALU opcodes, and mux select constants[span_69](start_span)[span_69](end_span)[span_70](start_span)[span_70](end_span). |
| `rv_sim.sv` | Top-level SystemVerilog testbench with instruction/data memory models and execution monitor[span_71](start_span)[span_71](end_span)[span_72](start_span)[span_72](end_span). |
| `test.s` | RISC-V assembly verification program testing `lw`, the custom `addi`, and `sw`[span_73](start_span)[span_73](end_span)[span_74](start_span)[span_74](end_span). |
| `imem.hex` | Compiled hexadecimal machine code loaded into instruction memory[span_75](start_span)[span_75](end_span)[span_76](start_span)[span_76](end_span). |
| `dmem_init.hex` | Initial hexadecimal data memory image[span_77](start_span)[span_77](end_span)[span_78](start_span)[span_78](end_span). |

---

## Verification & Simulation

The processor was verified in **ModelSim** using `rv_sim.sv` and the assembly test program `test.s`[span_79](start_span)[span_79](end_span)[span_80](start_span)[span_80](end_span)[span_81](start_span)[span_81](end_span):
1. Loads initial data from memory address `8` (`0x00000BED`) into register `a1` (`lw a1, 8(x0)`)[span_82](start_span)[span_82](end_span)[span_83](start_span)[span_83](end_span)[span_84](start_span)[span_84](end_span).
2. Executes the custom `addi a2, a1, 12` instruction (`0x00C58613`), computing `(0xBED + 12) ^ 0xFFFFFFFF = 0xFFFFF406`[span_85](start_span)[span_85](end_span)[span_86](start_span)[span_86](end_span)[span_87](start_span)[span_87](end_span)[span_88](start_span)[span_88](end_span).
3. Writes the result from `a2` to memory address `16` (`sw a2, 16(x0)`)[span_89](start_span)[span_89](end_span)[span_90](start_span)[span_90](end_span).
4. Triggers the testbench completion sequence by writing `0xDEAD` to address `0xFFFF`, dumping the final memory state to `dmem_out.hex`[span_91](start_span)[span_91](end_span)[span_92](start_span)[span_92](end_span)[span_93](start_span)[span_93](end_span).

![ModelSim Waveform Simulation](images/waveform.png)

### Running the Simulation in ModelSim
```tcl
vlib work
vlog *.sv
vsim rv_sim
add wave -r /*
run -all
