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

## 4 Bringing It All Together — `ExecUnit`  
The top-level entity instantiates each block once:

* **Control word** –  
  * `FuncClass` (2 bits) decides what the final multiplexer outputs:  
    * `"00"` → arithmetic/shift result  
    * `"01"` → logic result  
    * `"10"` → `ALTB` zero-extended to 64 bits  
    * `"11"` → `ALTBU` zero-extended to 64 bits  
  * `AddnSub`, `LogicFn`, `ShiftFn`, and `ExtWord` feed the sub-units directly.  
* **Intermediate muxing** – The SLU and ALU share a small inside mux so a shift can replace an arithmetic result without extra top-level delay.  Sign-extension logic widens 32-bit “-W” results before they hit the outer mux.  
* **Flag output** – `Zero`, `ALTB`, and `ALTBU` are driven straight from the ALU; other ALU flags are available to future pipeline stages if needed.  
This structure isolates each sub-unit’s delay, leaving the ALU carry chain as the critical path—optimised automatically by FPGA hard carry cells.

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
