# SAP-01

An 8-bit computer based on the SAP-1 (Simple-As-Possible) architecture, built and simulated in Proteus 8.12. All registers share one 8-bit bus, tri-state buffers prevent bus contention, and a ring-counter controller/sequencer runs each instruction through five T-states. A 7-segment display shows the current T-state.

---

## Architecture

![SAP-01 Architecture](SAP%2001%20Architecture.png)

Each register loads from the bus only when its active-low load pin goes low, and drives the bus only when its active-high enable is on.

---

## Instruction Set

Upper nibble is the opcode, lower nibble is the operand address.

| Mnemonic | Opcode | Operation |
|---|---|---|
| `LDA addr` | `0000` | A ← RAM[addr] |
| `ADD addr` | `0001` | A ← A + RAM[addr] |
| `SUB addr` | `0010` | A ← A − RAM[addr] |
| `OUT` | `0011` | OUT ← A |
| `HLT` | `1111` | Stop the clock |

The opcode decoder uses one 4-input AND gate per instruction. Each gate takes an opcode bit directly where the opcode has a 1 and inverted where it has a 0, so only one instruction line goes high.

---

## Instruction Cycle

**Fetch**

| T-state | Active signals | Transfer |
|---|---|---|
| T1 (Address) | `CO`, `MI'` | PC → MAR |
| T2 (Increment + Memory) | `CE`, `RO'`, `II'` | PC + 1, RAM → IR |

**Execute**

| Instruction | T3 | T4 | T5 |
|---|---|---|---|
| `LDA` | `IO'`, `MI'` | `RO'`, `AI'` | — |
| `ADD` | `IO'`, `MI'` | `RO'`, `BI'` | `EO`, `AI'` |
| `SUB` | `IO'`, `MI'` | `RO'`, `BI'` | `EO`, `AI'`, `SU` |
| `OUT` | `AO`, `OI'` | — | — |

When the IR holds `1111 XXXX`, a NAND gate drives `HLT'` low. That signal goes into an AND gate in the main clock, which forces the clock to 0 and stops the whole computer.

### Microinstructions

| Macro | State | CO CE MI' | RO' II' IO' | AI' AO EO | SU BI' OI' |
|---|---|---|---|---|---|
| LDA | T3 | 0 0 0 | 1 1 0 | 1 0 0 | 0 1 1 |
| | T4 | 0 0 1 | 0 1 1 | 0 0 0 | 0 1 1 |
| ADD | T3 | 0 0 0 | 1 1 0 | 1 0 0 | 0 1 1 |
| | T4 | 0 0 1 | 0 1 1 | 1 0 0 | 0 0 1 |
| | T5 | 0 0 1 | 1 1 1 | 0 0 1 | 0 1 1 |
| SUB | T3 | 0 0 0 | 1 1 0 | 1 0 0 | 0 1 1 |
| | T4 | 0 0 1 | 0 1 1 | 1 0 0 | 0 0 1 |
| | T5 | 0 0 1 | 1 1 1 | 0 0 1 | 1 1 1 |
| OUT | T3 | 0 0 1 | 1 1 1 | 1 1 0 | 0 0 0 |

---

## Modules

### 8-bit Register (74LS173)
Every 8-bit register uses two 74LS173 4-bit D-type registers.

| Pin | Name | Active | Function |
|---|---|---|---|
| 11, 12, 13, 14 | D0–D3 | High | Inputs |
| 3, 4, 5, 6 | Q0–Q3 | High | Outputs |
| 7 | CLK | +ve edge | Clock |
| 1, 2 | OE1', OE2' | Low | Output enable (grounded) |
| 9, 10 | E1, E2 | Low (as used) | Load |
| 15 | MR | High | Master reset |

Pins 1 and 2 are grounded so the outputs are always valid. Bus access is controlled by separate tri-state buffers.

### Accumulator
| Inputs | Outputs (bus) | Outputs (ALU) | Control |
|---|---|---|---|
| J7–J0 | j7–j0 | A7–A0 | `AI'`, `AO`, CLK, RESET |

Two 74LS173 chips with shared CLK and RESET. The outputs go directly to the ALU, and through tri-state buffers (enabled by `AO`) to the bus.

### Register B
| Inputs | Outputs (ALU) | Control |
|---|---|---|
| N7–N0 | B7–B0 | `BI'`, CLK, RESET |

Same wiring as the accumulator, but the outputs go only to the ALU.

### Output Register
| Inputs | Outputs | Control |
|---|---|---|
| OI7–OI0 | OO7–OO0 | `OI'`, CLK, RESET |

Same wiring as Register B.

### Instruction Register
| Inputs | Outputs (bus) | Outputs (sequencer) | Control |
|---|---|---|---|
| IN7–IN0 | IR3–IR0 | IC7–IC4 | `II'`, `IO'`, CLK, RESET |

- **Upper nibble (opcode):** this chip has pins 1 and 2 grounded, so it always feeds the sequencer.
- **Lower nibble (operand):** this chip uses pins 1 and 2 as the `IO'` enable, so it drives the bus only when `IO'` is active.

### Program Counter
| | |
|---|---|
| IC | 74LS107 |
| Inputs | CE, CO, CLK, RESET |
| Outputs (buffer) | C3–C0 |
| Outputs (display) | P3–P0 |

- **Counter type:** a 4-bit ripple counter that counts 0000 → 1111.
- **First stage:** the rightmost flip-flop toggles on the clock.
- **Ripple:** each Q output drives the active-low clock input of the next flip-flop.
- **Enable:** all J and K inputs are tied to `CE`.
- **Bus output:** the outputs reach the bus through tri-state buffers controlled by `CO`.

### Controller / Sequencer
| Inputs | Outputs (active low) | Outputs (active high) | Control | T-states |
|---|---|---|---|---|
| IC7–IC4 | `HLT'`, `MI'`, `II'`, `IO'`, `AI'`, `BI'`, `OI'` | `CO`, `CE`, `AO`, `SU`, `EO` | CLK, RESET | 5 |

The ring counter uses five master-slave JK flip-flops, one per T-state. On reset, the complemented output of the rightmost flip-flop gives T1. Each clock pulse then shifts the single 1 one flip-flop to the left.

| RESET | CLK | Q0 | Q1 | Q2 | Q3 | Q4 |
|---|---|---|---|---|---|---|
| 0 | 0 | 1 | 0 | 0 | 0 | 0 |
| | ↑ | 0 | 1 | 0 | 0 | 0 |
| | ↑ | 0 | 0 | 1 | 0 | 0 |
| | ↑ | 0 | 0 | 0 | 1 | 0 |
| | ↑ | 0 | 0 | 0 | 0 | 1 |
| | ↑ | 1 | 0 | 0 | 0 | 0 |

Combinational logic combines the active T-state with the decoded instruction to drive the control outputs.

### T-State Display
A BCD 7-segment display shows the current T-state (1–5). The fourth BCD input is grounded.

```
t3 = (T1 + T2 + T3)' · (T4 ⊕ T5)
t2 = (T1 + T4 + T5)' · (T2 ⊕ T3)
t1 = T2' · T4' · (T1·T3'·T5' + T1'·T3·T5' + T1'·T3'·T5)
```

---

## Example Program

| Address | Instruction | Binary |
|---|---|---|
| `0H` | `LDA FH` | `0000 1111` |
| `1H` | `ADD EH` | `0001 1110` |
| `2H` | `HLT` | `1111 XXXX` |

Program the RAM with the address and data switches, reset, then run the clock.

---

## Problems Faced

| Module | Problem | Resolution |
|---|---|---|
| Clock | The push button and potentiometer were internal to the clock subcircuit and could not be reached from outside | Added terminals and fed them as inputs to a "Mode Switching" module |
| Clock | Astable mode threw a Proteus error because the SPDT switch had no initial value | Known simulation issue; give the switch an initial position before running |
| Accumulator | Pin names that differed only in case (e.g. `J0` / `j0`) shorted together | Used distinct net names |
| Sequencer | Worked alone, but on the bus it showed logic contention on 20+ wires, reached only 4 T-states, and only LDA gave output | Reverted to a different opcode decoding, which also fixed `HLT'` |
| T-State Display | Worked alone but showed nothing once integrated | Reworked the Boolean expressions |

---

## Documentation

Theory, block diagrams and module notes are in `SAP 01 Documentation.docx`.
