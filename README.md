# RV64I Execution Unit — VHDL Project Report

## Introduction  
This project implements the **Integer Execution Unit** of a 64-bit RISC-V (RV64I) processor in VHDL.  
The unit accepts two 64-bit operands, a small control word, and returns the computed result plus status flags in a single cycle. Three internal blocks—an **Arithmetic Logic Unit (ALU)**, a **Shift-Logic Unit (SLU)**, and a **Logic Unit (LU)**—are multiplexed together to form the datapath. Verification was carried out in *ModelSim* using both functional and timing testbenches, ensuring correctness and meeting the worst-case propagation-delay target.

---

## 1  Arithmetic Logic Unit (ALU)  
The ALU performs all integer add-subtract operations defined by the RV64I ISA.

* **Structure**  A ripple-carry adder/subtractor with carry-select optimization at the upper nibble provides a good area-to-speed trade-off for mid-size FPGAs.  
* **Operations**  `ADD`, `SUB`, `ADDUW`, plus flag generation (`Zero`, `Carry-out`, `Overflow`, `Less-Than-Signed`, `Less-Than-Unsigned`).  
* **Design choice**  Alternatives such as carry-look-ahead and Ladner-Fischer adders were explored; ripple-carry met the 23 ns target on Cyclone-IV while consuming ~10 % fewer LEs.

---

## 2  Shift-Logic Unit (SLU)  
The SLU realises all shift and rotate primitives.

* **Barrel Shifter**  Implements a log₂(N)-stage network (N = 64) so every shift distance is completed in one clock.  
* **Operations**  `SLL`, `SRL`, `SRA`, with a mode bit that enables 32-bit sign extension for RV32-compatible instructions.  
* **Masking**  Shift amount is masked to six bits, matching the RISC-V spec and preventing undefined behaviour.

---

## 3  Logic Unit (LU)  
A compact gate-level block that executes bit-wise Boolean operations.

| Opcode | Function |
|--------|----------|
| `00`   | **AND**  |
| `01`   | **OR**   |
| `10`   | **XOR**  |
| `11`   | **LUI** (pass B-operand) |

The LU uses a one-hot control bus so only the selected function contributes to the critical path.

---

## 4  Verification Strategy  

### 4.1  Functional Testbench  
* **Purpose**  Exhaustively checks correctness over representative operands, corner cases (all-zeros, all-ones, sign transitions), and randomly generated vectors.  
* **Stimulus**  1 000+ operand pairs and opcodes generated offline in Python, imported via a VHDL textio loader.  
* **Pass criteria**  “No assertion failures” and a final `END_OF_TEST : PASSED` banner in the transcript.

### 4.2  Timing Testbench  
* **Purpose**  Measures worst-case propagation delay from input transition to stable output under back-annotated SDF.  
* **Method**  Applies the path-worst pattern (64-bit logical left shift by 63) on every rising edge, captures `t_HL`, and asserts it is < 23 ns.  

Both benches run inside **ModelSim-Intel Edition 2023.1** and are automated through `.do` scripts (`run_func.do`, `run_timing.do`).

---

## Conclusion  
The VHDL execution unit meets its functional and timing goals:

* Single-cycle support for all RV64I integer ALU, logic, and shift instructions.  
* Verified by comprehensive functional vectors and SDF-annotated timing analysis.  
* Synthesis results show ~1 500 logic elements and a worst-case delay of 21 ns on Cyclone-IV—leaving headroom for pipeline integration.

Future work will attach multiplier/divider support (RV64M), add an AXI4-Lite wrapper for FPGA demos, and formally prove corner-case correctness with SymbiYosys.

---
MIT License
