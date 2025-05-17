# RV64I Execution Unit — VHDL Project Report

## Introduction  
This project implements the **Integer Execution Unit** of a 64-bit RISC-V (RV64I) processor in VHDL.  
Two 64-bit operands and a small control word enter; a single-cycle result and status flags exit.  
Three purely combinational blocks—an **Arithmetic Logic Unit (ALU)**, a **Shift-Logic Unit (SLU)**, and a **Logic Unit (LU)**—feed a final multiplexer selected by the decode logic.  
Functional and timing testbenches in *ModelSim* validate correctness and ensure the design beats a 23 ns worst-case delay budget.

---

## 1 Arithmetic Logic Unit (ALU)  
* **Coding approach** – A generic ripple-carry adder is instantiated once.  Subtraction is obtained by inverting **B** and asserting the adder’s carry-in, so the same hardware handles `ADD`, `SUB`, and 32-bit `ADDUW`.  
* **Flags generated** –  
  * `Zero` (result = 0)  
  * `Carry-out` and `Overflow` from the adder  
  * **`ALTB`** (A < B, signed) computed as *Overflow ⊕ MSB(result)*  
  * **`ALTBU`** (A < B, unsigned) given by the inverse of carry-out  
* **Why ripple-carry?** – For 64 bits it meets timing on Intel Cyclone IV while using about 10 % fewer LUTs than look-ahead options because the FPGA’s hard add-carry chain keeps the path short.

---

## 2 Shift-Logic Unit (SLU)  
* **Coding approach** – Relies on `numeric_std.shift_left/shift_right`, allowing synthesis to infer a log₂(N)-stage barrel shifter that completes in one clock.  
* **Operations** – `SLL`, `SRL`, `SRA`; arithmetic right shift is achieved by casting the operand to **signed** before shifting.  
* **Word-extension mode** – An `ExtWord` bit forces 32-bit behaviour for “-W” instructions and masks the shift amount so only legal distances (0 … 31) are used in that mode.  
* **Safe shifts** – High bits of the shift count are zeroed, preventing undefined distances on the hardware shifter.

---

## 3 Logic Unit (LU)  
A single combinational `case` statement implements four Boolean functions:  

| Selector | Function |
|----------|----------|
| `00` | **LUI-style upper-immediate** load (upper 20 bits of **B**) |
| `01` | **XOR** |
| `10` | **OR** |
| `11` | **AND** |

Only the selected branch drives the output, ensuring the logic path is just one gate deep.

---

## 4  Execution Unit (ExecUnit)

The **ExecUnit** is a thin VHDL wrapper that drops the three blocks (ALU, Shifter, Logic) side-by-side, drives them with the same operands, then chooses which result to send out.

* **Sub-units used**
  * **AUnit** – adds or subtracts (`AddnSub`=0/1) and produces the flags `Zero`, `ALTB`, `ALTBU`.
  * **shifter** – performs `SLL`, `SRL`, or `SRA` as selected by `ShiftFN`; an `ExtWord` bit forces correct 32-bit “-W” behaviour.
  * **logic** – executes one of four Boolean ops set by `LogicFN`.

* **Selection logic**
  * A tiny mux first decides between the ALU value and the shifter value (based on the low bit of `ShiftFN`).
  * A second, top-level 2-bit field **`FuncClass`** picks the final 64-bit output `Y`:
    * `"00"` → arithmetic or shift result  
    * `"01"` → logic result  
    * `"10"` → `ALTB` zero-extended to 64 bits  
    * `"11"` → `ALTBU` zero-extended to 64 bits  

* **Flag routing**
  * `Zero`, `ALTB`, and `ALTBU` leave the ALU and go straight to ExecUnit outputs, ready for branch or compare logic.

Because each block is pure combinational logic and the adder’s carry chain is mapped to FPGA carry cells, the assembled unit still meets the 23 ns timing target (measured 21.02 ns on Cyclone IV).

---

## 5 Verification Strategy  
* **Functional testbench** – Runs > 1 000 random and corner-case vectors across every opcode; finishes only if no assertions trip.  
* **Timing testbench** – Loads the post-layout SDF file, applies the path-worst 64-bit left-shift, and asserts total propagation delay < 23 ns.  
Both benches are scripted and run automatically in *ModelSim-Intel Edition 2023.1*.

---

## Conclusion  
| Metric | Result | Target |
|--------|--------|--------|
| Worst-case combinational delay | **21.02 ns** | < 23 ns |
| FPGA area | **≈ 1 500 logic elements** + 15 carry chains | — |
| Functional failures | **0 / > 1 000 vectors** | 0 |
| Critical-path slack (post-SDF) | **Positive** | ≥ 0 |

Waveforms show clean single-cycle behaviour across arithmetic, shift, and logic instructions, with flags (`Zero`, `Carry-out`, `Overflow`, `ALTB`, `ALTBU`) matching the reference model cycle-for-cycle.  
**RTL screenshots of the ALU, SLU, and LU, plus ModelSim waveforms for both testbenches, are included in the repository.**
