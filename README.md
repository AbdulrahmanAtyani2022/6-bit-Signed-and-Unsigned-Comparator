# 6-bit Comparator with Registered I/O (Verilog)

A gate-level (structural) 6-bit magnitude comparator written in Verilog, with a signed/unsigned select input, registered inputs and outputs, and a self-checking testbench that verifies the design against a behavioral model.

> Course project for **ENCS3310 – Advanced Digital Design**, Electrical & Computer Engineering Department, Birzeit University (First Semester 2024/2025).

---

## Features

- **Structural design** built only from primitive gates (`xnor`, `not`, `and`, `or`) with explicit propagation delays
- Three outputs: **LT** (A < B), **GT** (A > B), **EQ** (A = B)
- **Selection input `S`** to choose between unsigned and signed mode
- **Registered inputs and outputs**: two 6-bit registers and three D flip-flops with asynchronous active-low reset
- **Behavioral reference model** running in parallel for verification
- **Self-checking testbench** that prints `Pass` or the number of mismatches
- Critical path timing analysis (31 ns)

---

## How It Works

### Comparator logic (MSB to LSB)

| Output | Logic |
|--------|-------|
| `EQ`   | `E[i] = XNOR(A[i], B[i])`, and `EQ = AND(E[5..0])` |
| `GT`   | A bit is 1 and B bit is 0 at the first (most significant) differing bit |
| `LT`   | A bit is 0 and B bit is 1 at the first (most significant) differing bit |

For each bit `i`, `Q[i] = A[i] & ~B[i]` and `U[i] = ~A[i] & B[i]`. A bit position only decides the result if all more-significant bits are equal, so each term is ANDed with the equality signals above it, and the terms are ORed together:

```
GT = Q5 + E5·Q4 + E5·E4·Q3 + ... + E5·E4·E3·E2·E1·Q0
LT = U5 + E5·U4 + E5·E4·U3 + ... + E5·E4·E3·E2·E1·U0
```

### Signed / unsigned select

| `S` | Behavior |
|-----|----------|
| `0` | Inputs are compared as unsigned numbers |
| `1` | Inputs are converted with two's complement (`~X + 1`) before the same comparison is applied |

### Full system (pipeline)

```
A ──► [Register 6-bit] ──┐                         ┌──► [D-FF] ──► LT
                         ├──► [Comparator (S)] ────┼──► [D-FF] ──► GT
B ──► [Register 6-bit] ──┘                         └──► [D-FF] ──► EQ
                  ▲                                         ▲
                  └────────────── Clk, Reset ───────────────┘
```

On each clock edge the inputs are latched into the registers while the previous comparison result is latched into the output flip-flops, so a new input pair can be applied every clock cycle.

---

## Module Overview

| Module | Description |
|--------|-------------|
| `Unsigned` | Gate-level 6-bit comparator producing LT, GT, EQ |
| `final_comparter` | Adds the signed/unsigned select (`S`) in front of `Unsigned` |
| `DFF` | D flip-flop with asynchronous active-low reset |
| `register #(n)` | Parameterized n-bit register (default 6) with asynchronous active-low reset |
| `Full_comparter` | Complete structural system: input registers + comparator + output flip-flops |
| `behavioral` | Behavioral reference model using `<`, `>`, `==` with the same registers |
| `test_circuit` | Testbench instantiating both designs and comparing their outputs |

---

## Gate Delays and Timing

| Gate | Delay |
|------|-------|
| `not` | 4 ns |
| `xnor` | 10 ns |
| `and` | 9 ns |
| `or` | 9 ns |

Critical path: `not (4) → and (9) → and (9) → or (9)` = **31 ns**

The clock period is set to 31 ns in the testbench, which gives a maximum frequency of about **32.26 MHz** (`f = 1 / T`).

---

## Testbench and Simulation

The testbench:

1. Generates a clock with a 31 ns period
2. Applies `{A, B}` as a 12-bit counter, covering **all 4096 combinations in the sweep** (counter-based)
3. Compares the structural outputs (`LT`, `GT`, `EQ`) with the behavioral outputs on every step
4. Increments an error counter `T` on any mismatch
5. Prints `Pass` if `T == 0`, otherwise `Fail Number of errors <T>`

To check that the testbench really detects errors, a single gate can be modified on purpose (for example, breaking one `xnor`). In the report this produced `Fail Number of errors 127`.

### Running the simulation

Using Icarus Verilog:

```bash
iverilog -o comparator_sim comparator.v
vvp comparator_sim
```

Or load the file in any Verilog simulator (ModelSim, QuestaSim, Vivado, EDA Playground) and run the `test_circuit` module as the top level.

---

## Repository Structure

```
.
├── comparator.v        # All modules + testbench (rename to match your file)
├── report.pdf          # Full project report
└── README.md
```

---

## Known Limitations and Future Work

- **Signed mode**: when `S = 1`, the current design negates both inputs with two's complement and then compares them as unsigned values. This is not the same as a true signed comparison, where the MSB acts as the sign bit. A proper implementation would treat the MSB as the sign bit (for example, by inverting the MSB of both inputs before the unsigned comparison).
- **Test coverage**: the testbench runs with `S = 0`; extending the sweep to also cover `S = 1` would validate signed mode.
- **Hard-coded delays** (`#4`, `#9`, `#10`) could be turned into parameters to make timing easier to adjust.
- The testbench does not cover reset behavior during operation or unknown (`X`/`Z`) inputs.
- The design could be extended to a parameterized N-bit comparator and used as a building block inside a simple ALU.

---

## Author

**Abdulrahman Atyani**
Birzeit University – Electrical & Computer Engineering
Instructor: Dr. Abdullatef Abu Issa
