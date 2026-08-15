# 13-Bit-CPU — A Gate-Level Computer Built in Logisim

# 📖 Overview

The **13-Bit CPU** is a complete, gate-level, single-accumulator computer designed and simulated entirely in **Logisim Evolution**. It implements a full fetch–decode–execute cycle, a genuinely **microprogrammed control unit**, a custom **13-bit ALU**, and an integrated **hardware Booth's-algorithm multiplier** — all built from primitive logic gates, registers, counters, comparators, multiplexers, and ROMs, with no built-in CPU blocks used anywhere.

Unlike a minimal 8-instruction textbook CPU, this system implements a **16-slot instruction set** (14 populated), a **real microcode ROM** driving every control signal, a **shared arithmetic core** reused across five different instructions, and a **multi-cycle hardware multiplier** that is made to fit inside a control scheme originally built only for single-cycle instructions — through a dedicated stall mechanism built from a free-running counter and a pair of comparators.

Every architectural fact in this README — every ROM byte, every register width, every claim about how the multiply instruction actually works — was confirmed by directly parsing the project's own `.circ` source files and cross-checking that analysis against a live, single-stepped simulation trace.

📄 For the complete signal-by-signal breakdown, see the [Technical Report](docs/13bit_CPU_Technical_Report_.pdf).

---

# ✨ Key Features

## 🧠 CPU Core

- Full Fetch–Decode–Execute Cycle
- Single 13-bit Accumulator Architecture
- 9-bit Program Counter (512-word address space)
- Memory Address Register (MAR)
- Memory Buffer Register (MBR)
- 4-bit Instruction Register (opcode-only)
- 13-bit Internal Data Bus
- Reset and Clock-Driven Synchronous Operation

---

## 🎛️ Microprogrammed Control Unit

- 64-Word × 8-Bit Microcode ROM
- 6-Bit Self-Addressing Microprogram Counter
- Self-Redirecting Opcode Dispatch (zero dedicated decode logic)
- Decoder + Priority Encoder Signal Expansion
- 14 Independent Control Signal Outputs
- Fixed 4-Microstate Instruction Cycle (T0–T3)

---

## 🧮 Arithmetic Logic Unit (ALU)

- Custom 13-Bit ALU (AND / OR / ADD / SUB)
- Single Shared Ripple-Carry Adder Reused for 5 Operations
- Two's-Complement Subtraction via XOR Trick
- Signed Overflow, Zero, Negative, and Carry Flags
- Three Dedicated Decode ROMs for Opcode-to-Hardware Routing
- Clean Multiplexer-Based Output Selection

---

## ✖️ Hardware Multiplier (Booth's Algorithm)

- True Iterative 13-Cycle Hardware Multiplication
- Dedicated `M`, `A`, `Q`, `Q-1` Shift Registers
- Independent 8-State Multiplier Control Unit (`cu_mul`)
- 12-Bit Microcoded Control Word
- 26-Bit Full-Precision Internal Product
- CPU-Level Stall Mechanism for Multi-Cycle Integration

---

## 💾 Memory System

- 512 × 13-Bit Unified RAM
- Von Neumann Architecture (shared program/data space)
- Separate Read/Write Data Bus Lines
- Persistent Program Loading via Logisim RAM Image

---

## 🔍 Verification System

- Static XML-Level Source Verification
- Dynamic Live Simulation Trace Verification
- Cross-Checked ROM Content Analysis
- Register-Level Execution Trace Logging
- Documented, Corrected Errors From Earlier Analysis Drafts

---

# 🏗️ System Architecture

```
                        +------------------+
                        |   Clock / Reset  |
                        +--------+---------+
                                 |
                        +--------v---------+
                        |    CPU Datapath  |
                        +--------+---------+
                                 |
        +---------+---------+---+---+---------+---------+
        |         |         |       |         |         |
        v         v         v       v         v         v
     +----+    +-----+   +-----+  +----+   +------+  +--------+
     | PC |    | MAR |   | IR  |  |MBR |   |  Ac  |  |  RAM   |
     +----+    +-----+   +-----+  +----+   +------+  | 512x13 |
                                                       +--------+
                                 |
                        +--------v---------+
                        |  ALU + Booth's   |
                        |    Multiplier    |
                        +--------+---------+
                                 |
                        +--------v---------+
                        |   ControlUnit    |
                        |  (Microcode ROM) |
                        +------------------+
```

---

# 🔄 Instruction Execution Workflow

```
                    Power On / Reset
                            │
                            ▼
                    Fetch Instruction
                    (MAR<-PC, RAM->MBR)
                            │
                            ▼
                    Decode Opcode
              (IR<-MBR[12:9], ROM addr jumps)
                            │
                            ▼
                    Execute (T1)
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
        Opcode = MUL?                Normal Opcode
              │                           │
              ▼                           │
     Booth Multiplier Stall               │
     (46-tick counter, 13                 │
      internal Booth iterations)          │
              │                           │
              └─────────────┬─────────────┘
                            ▼
                     Write Back (T2/T3)
                  (LoadAc, PC++, fetch next)
                            │
                            ▼
                  Repeat Until HLT Executed
```

---

# 📂 Project Structure

```
13-Bit-CPU-With-Booth-Multiplier/
│
├── 13_bit_cpu.circ          # CPU datapath + ControlUnit + test bench
├── 13-bit-ALU.circ          # ALU, decode ROMs, Booth multiplier, cu_mul
│
├── docs/
│   └── Technical_Report.pdf # Full architecture, ROM traces, timing diagrams
│
├── assets/
│   └── (diagrams / screenshots)
│
└── README.md
```

---

# 🌟 Highlights

* Complete Gate-Level CPU Design
* Genuine Microprogrammed Control Unit
* Hardware Booth's Multiplier (not a software loop)
* Single Shared Adder Reused Across Five Operations
* Self-Redirecting Microcode Address Scheme
* CPU-Level Multi-Cycle Stall Mechanism
* Fully Verified ROM Contents (source-checked, not assumed)
* Live-Traced Execution Proof
* Modular, Extensible 16-Slot Instruction Set
* Two Reserved Opcodes for Future Expansion
* Clean Separation Between CPU Project and ALU Project
* Reusable ALU Test Bench Embedded as External Library

---

# 🧠 CPU Datapath Module

The datapath is the physical backbone of the machine — every register, bus, and connection that data actually moves through during execution.

## 🔑 Register Set

| Register | Width | Role |
|---|---|---|
| `PC` | 9-bit | Program Counter, max address `0x1FF` |
| `MAR` | 9-bit | Memory Address Register — drives the RAM address bus |
| `IR` | 4-bit | Instruction Register — holds the opcode only |
| `MBR` | 13-bit | Memory Buffer Register — holds the full fetched word |
| `Ac` | 13-bit | Accumulator — sole destination of every ALU/multiplier result |

## 📐 Instruction Word Format

```
 12  11  10   9    8   7   6   5   4   3   2   1   0
+---+---+---+---+-----------------------------------+
|      OPCODE       |            ADDRESS / OPERAND   |
|     (4 bits)       |            (9 bits)            |
+---+---+---+---+-----------------------------------+
```

## 💾 Memory Configuration

- Type: 512 × 13-bit RAM
- Bus Mode: Separate Read/Write Data Lines
- Address Width: 9 bits
- Shared Program and Data Space (Von Neumann model)

---

# 📋 Instruction Set

The CPU supports 16 possible opcodes, of which 14 are implemented.

| Opcode | Mnemonic | Type | Operation |
|---|---|---|---|
| `0x0` | AND | Logic | `Ac <- Ac AND M[addr]` |
| `0x1` | OR | Logic | `Ac <- Ac OR M[addr]` |
| `0x2` | ADD | Arithmetic | `Ac <- Ac + M[addr]` |
| `0x3` | SUB | Arithmetic | `Ac <- Ac - M[addr]` |
| `0x4` | STO | Memory | `M[addr] <- Ac` |
| `0x5` | LDA | Memory | `Ac <- M[addr]` |
| `0x6` | BUN | Branch | `PC <- addr` (unconditional) |
| `0x7` | JZ | Branch | `if Ac = 0 then PC <- addr` |
| `0x8` | HLT | Control | Halts the clock sequencer |
| `0x9` | JNZ | Branch | `if Ac != 0 then PC <- addr` |
| `0xA` | INC | Arithmetic | `Ac <- Ac + 1` |
| `0xB` | DEC | Arithmetic | `Ac <- Ac - 1` |
| `0xC` | 2's-Complement | Arithmetic | `Ac <- -Ac` |
| `0xD` | **MUL** | Arithmetic | `Ac <- (Ac x M[addr])[12:0]` |
| `0xE` | — | Reserved | Unused, free for expansion |
| `0xF` | — | Reserved | Unused, free for expansion |

---

# 🎛️ Microprogrammed Control Unit

## 🔧 Architecture

- 6-bit self-addressing microprogram counter (max `0x3F` = 63)
- 64-word × 8-bit microcode ROM
- 5-to-32 Decoder
- 2-bit Priority Encoder
- 14 independent output control signals

## 🔁 Self-Redirecting Address Scheme

Every instruction is allocated exactly four consecutive ROM words:

```
ROM Address = (Opcode << 2) + Microstep
```

The ROM address is wired **directly from the live Instruction Register**, not from a separately latched or dispatched value. The instant the opcode nibble of a freshly fetched word lands in `IR`, the ROM address automatically jumps to that opcode's own four-word microcode block — with no dedicated instruction-dispatch hardware anywhere in the design.

## 📜 Full Verified Microcode ROM

```
09 19 14 29 19 31 02 29
19 41 02 29 19 39 02 29
19 49 02 29 51 59 02 29
19 71 02 21 02 02 02 21
02 00 00 7A 00 00 00 61
02 00 00 69 39 02 00 69
49 02 00 81 19 A9 02 29
19 41 02 B9 00 00 00 00
```

| Opcode | T0 | T1 | T2 | T3 |
|---|---|---|---|---|
| AND (0x0) | 09 | 19 | 14 | 29 |
| OR (0x1) | 19 | 31 | 02 | 29 |
| ADD (0x2) | 19 | 41 | 02 | 29 |
| SUB (0x3) | 19 | 39 | 02 | 29 |
| STO (0x4) | 19 | 49 | 02 | 29 |
| LDA (0x5) | 51 | 59 | 02 | 29 |
| BUN (0x6) | 19 | 71 | 02 | 21 |
| JZ (0x7) | 02 | 02 | 02 | 21 |
| HLT (0x8) | 02 | 00 | 00 | 7A |
| JNZ (0x9) | 00 | 00 | 00 | 61 |
| INC (0xA) | 02 | 00 | 00 | 69 |
| DEC (0xB) | 39 | 02 | 00 | 69 |
| 2sC (0xC) | 49 | 02 | 00 | 81 |
| **MUL (0xD)** | 19 | **A9** | 02 | 29 |
| (0xE unused) | 19 | 41 | 02 | B9 |
| (0xF unused) | 00 | 00 | 00 | 00 |

> Full opcode-by-opcode reasoning for every byte, plus the live-trace proof that MUL's `0xA9` byte does genuinely different work than ADD's `0x41` byte, is documented in the [Technical Report](docs/Technical_Report.pdf).

---

# 🧮 ALU Module

## 🔹 Core Arithmetic Design

The entire arithmetic capability of the machine funnels through **one physical adder**.

```
B' = B XOR (Sub broadcast to 13 bits)
Result, Carry = 13-bit Ripple-Carry-Adder(A, B', Cin = Sub)
```

| Sub | Result |
|---|---|
| 0 | `A + B` |
| 1 | `A - B` (two's-complement, via XOR + carry-in trick) |

This single adder is reused for: **ADD, SUB, INC, DEC, 2's-Complement**, and the internal accumulate step of the Booth multiplier.

## 🔹 Control1:Control0 Select Table

| Control1 | Control0 | Operation |
|---|---|---|
| 0 | 0 | AND |
| 0 | 1 | OR |
| 1 | 0 | ADD |
| 1 | 1 | SUB |

## 🔹 Status Flags

- **Zero** — 13-input reduce-OR, inverted
- **Negative** — sign bit (bit 12) of the result
- **Overflow** — standard signed-overflow equation from A/B/Result sign bits
- **Carry** — raw adder carry-out

## 🔹 ALU Decode ROM System

Three small ROMs route sixteen opcodes down to the correct physical hardware block:

| ROM | Address Space | Purpose |
|---|---|---|
| ROM 1 | 4-bit (opcode) | Dispatch: raw Control1:0 for AND/OR/ADD/SUB, one-hot enable bits for INC/DEC/2sC/MUL |
| ROM 2 | 3-bit | Supplies Control1:0 for the adder-reuse path (INC/DEC/2's-Complement) |
| ROM 3 | 4-bit (opcode) | Final output-multiplexer select — decides which block's result reaches `alu_output` |

---

# ✖️ Booth's Multiplier Module

## 🔹 Registers

| Register | Width | Role |
|---|---|---|
| `M` | 13-bit shift register | Multiplicand |
| `A` | 13-bit shift register | Running accumulator |
| `Q` | 13-bit shift register | Multiplier |
| `Q-1` | 1-bit shift register | Previous-bit tracker |
| Iteration Counter | 4-bit, max 13 | Controls loop length |

## 🔹 Algorithm

```
Initialize: A = 0, Q = multiplier, Q-1 = 0, M = multiplicand, counter = 13

Repeat 13 times:
    if (Q0, Q-1) == (0,1): A = A + M
    if (Q0, Q-1) == (1,0): A = A - M
    Arithmetic-shift A:Q:Q-1 right by 1
    counter = counter - 1

Result = A:Q  (26-bit signed product)
```

## 🔹 Multiplier Control Unit (`cu_mul`)

- 3-bit Counter Address Register (`CAR`), 8-state Moore FSM
- 8-word x 12-bit dedicated microcode ROM
- Control Word Format (documented directly in the circuit):
  `Clear(A,Q-1) | Load(M,Q,Cnt) | LoadA | C0 | C1 | Shr | DecCnt | Addr(3) | Sel(2)`

```
ROM: 1  C01  2  394  281  61  B  1C
```

| State | Word | Role |
|---|---|---|
| 0 | 001 | idle |
| 1 | C01 | Clear A/Q-1, Load M/Q/Counter |
| 2 | 002 | dispatch / test Q0,Q-1 |
| 3 | 394 | add branch |
| 4 | 281 | subtract branch |
| 5 | 061 | arithmetic shift |
| 6 | 00B | decrement counter |
| 7 | 01C | loop-test / branch |

---

# ⏱️ MUL Stall Mechanism

Booth's algorithm needs 13 real clock iterations, but every CPU instruction is architected around a fixed 4-microstate window. This conflict is resolved with a dedicated stall subsystem bolted onto the CPU:

```
Comparator: IR == 0xD (MUL)?
        │
        ▼
Free-Running Counter (max 0x2E = 46)
        │
        ├── Checkpoint @ 4  -> Booth loop released to start
        │
        ▼
cu_mul runs its own 8-state FSM
for 13 full shift/add-subtract iterations
(independent clock domain from the CPU)
        │
        ▼
Checkpoint @ 46 -> CPU sequencer resumes
        │
        ▼
T2/T3 fire: LoadAc, PC++, fetch next instruction
```

This is a genuine, if minimal, clock-domain handoff — not a shortcut. It was verified directly: a test program loading `Ac <- 2` then executing `MUL` against a memory operand of `3` correctly produces `Ac = 6`, confirmed both inside the full CPU and independently on the standalone ALU test bench.

---

# 🧪 Verification & Testing

Every technical claim in this project was checked:

## 🔹 Dynamic Verification

- Live, single-stepped simulation trace of a real test program (`LDA 2`, `MUL 3`)
- Register-level trace of `MAR`, `IR`, `MBR`, `Ac` across every microstep
- Cross-checked ALU decode ROM outputs against the standalone ALU test bench


Full testing log and trace tables are in the [Technical Report](docs/Technical_Report.pdf).

---

# 💻 Technical Implementation

This project is implemented entirely as a **gate-level digital logic design** in Logisim Evolution — there is no software, firmware, or HDL involved. Every behavior described above is the emergent result of wiring primitive components together.

## 🔹 Design Principles Applied

- **Hardware Reuse** — one adder circuit serves five different instructions
- **Microprogrammed Control** — instruction behavior lives in ROM content, not gate wiring
- **Separation of Concerns** — the ALU project is a fully independent, separately testable circuit, embedded into the CPU as a black box
- **Minimal Gate-Count Design** — INC/DEC/2's-Complement are implemented as adder-reuse tricks rather than dedicated hardware
- **Self-Describing Circuits** — the multiplier's control-word format is documented directly inside the circuit via a `Text` annotation

## 🔹 Component Libraries Used

| Logisim Library | Components Used |
|---|---|
| Wiring | Pin, Tunnel, Splitter, Constant, Clock |
| Gates | AND, OR, NOT, XOR, NOR |
| Plexers | Multiplexer, Decoder, Priority Encoder |
| Arithmetic | Comparator |
| Memory | Register, Counter, RAM, ROM |
| I/O | Button |

---

# 📊 System Statistics

| Metric | Value |
|---|---|
| Word width | 13 bits |
| Address space | 512 words |
| Instruction opcodes implemented | 14 / 16 |
| ControlUnit ROM size | 64 x 8 bits |
| Multiplier control ROM size | 8 x 12 bits |
| ALU decode ROM count | 3 |
| Booth iterations per MUL | 13 |
| MUL CPU stall window | 46 clock ticks |
| Circuit files | 2 (`13_bit_cpu.circ`, `13-bit-ALU.circ`) |
| Total sub-circuits across both files | 17 |

---

# 🛠️ Technologies Used

| Technology | Purpose |
|---|---|
| Logisim Evolution 2.7.1 | Gate-level circuit design and simulation |
| Digital Logic Design | Combinational and sequential circuit construction |
| Microprogramming | Control unit implementation |
| Booth's Algorithm | Signed hardware multiplication |
| XML | Underlying `.circ` project file format (used for verification) |

---

# ▶️ Installation & Execution

## Prerequisites

- [Logisim Evolution](https://github.com/logisim-evolution/logisim-evolution) (built and verified on version `2.7.1`)

## Running the Project

### Step 1

Install & Open the Logisim Evolution

### Step 2

Keep `13_bit_cpu.circ` and `13-bit-ALU.circ` in the same folder — the CPU project loads the ALU project as an external library and will not run correctly if separated.

### Step 3

Open `13_bit_cpu.circ` in Logisim Evolution.

### Step 4

Load a program into the `main` circuit's RAM (right-click -> *Edit Contents*, or *Load Image*), packing each instruction word as:

```
word = (opcode << 9) | address
```

### Step 5

Toggle `Reset`, then run or single-step the `Clock`.

### Step 6

Observe `Ac`, `MAR`, `IR`, and `MBR` to verify execution. For `MUL`, allow the full 46-tick stall window before checking the result.

---

# 📸 Sample Modules

```
Main Test Bench
│
├── Clock
├── Reset
├── RAM (512 x 13)
└── CPU
```

```
CPU
│
├── PC / MAR / IR / MBR / Ac
├── ALU + Booth Multiplier (embedded external circuit)
├── MUL-Stall Logic (Comparator + Counter)
└── ControlUnit
```

```
ControlUnit
│
├── 6-bit Microprogram Counter
├── 64x8 Microcode ROM
├── Decoder
└── Priority Encoder
```

---

# 🚀 Future Enhancements

This project can be extended with several architectural improvements.

### Arithmetic

- Capture the full 26-bit multiplication product instead of truncating to 13 bits
- Route the Booth multiplier's result through the existing flag logic (Zero/Negative/Overflow/Carry)
- Add a hardware divider using one of the two reserved opcodes

### Control

- Replace the fixed 46-tick stall counter with a genuine "done" signal fed back from `cu_mul`'s own state machine
- Add a basic interrupt/trap mechanism using the remaining reserved opcode slot


# 🎓 Learning Outcomes

This project demonstrates practical, gate-level implementation of:

- Fetch-Decode-Execute CPU Architecture
- Microprogrammed Control Unit Design
- Ripple-Carry Adder and Two's-Complement Subtraction
- Signed Overflow / Zero / Carry Flag Logic
- Booth's Multiplication Algorithm
- Finite State Machine Design (Moore-style)
- Hardware Reuse and Minimal Gate-Count Design
- Multi-Cycle Instruction Integration into a Fixed-Cycle Pipeline
- Clock-Domain Handoff via Counter/Comparator Stalling
- Reverse-Engineering and Verifying a Digital Design from Its Own Source Files

---

# ⚠️ Important Notes

- Both `.circ` files must remain in the same folder — the CPU project references the ALU project as an external library by relative path.
- `MUL` requires the full 46-tick stall window to complete; reading `Ac` before this window elapses will show a stale value.
- The multiplication result stored in `Ac` is truncated to the low 13 bits of the true 26-bit product; large operand pairs will wrap around with no overflow warning.
- Opcodes `0xE` and `0xF` are reserved and currently unpopulated — free space for future instructions.

---

# 🤝 Contributing

Contributions are welcome.

1. Fork the repository.
2. Create a new feature branch.
3. Make your changes to the `.circ` files (Logisim Evolution required).
4. Update or re-verify the relevant ROM tables if microcode changes.
5. Open a Pull Request describing the change.

---

# 👨‍💻 Author

**Tasmin Rubaiat Rimve**

**Department of Computer Science & Engineering (CSE)**

**Khulna University of Engineering & Technology (KUET)**

---

# 📄 License

This project is developed for **educational and academic purposes**.

You are welcome to study, modify, and improve the project with proper attribution.

---

# ⭐ Support the Project

If you found this project useful or informative:

⭐ Star this repository

🍴 Fork the repository

📢 Share it with others

---

# 🙏 Acknowledgements

This project was developed as part of an academic exploration of **computer architecture and digital logic design**, focusing on building a genuinely microprogrammed CPU with an integrated hardware multiplier from first principles in Logisim Evolution — verified against its own source files rather than documented from assumption.
