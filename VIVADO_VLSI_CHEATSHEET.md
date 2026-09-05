# Vivado & RTL Design — Foundational Guide

> Complete reference for Vivado workflow, SystemVerilog templates, and RTL design best practices.
> Philosophy: **understand every abstraction level (gate → dataflow → behavioral), never trust a waveform you haven't self-checked, document everything in text so it's diffable on GitHub.**

---

## Overview

This guide covers everything you need for Vivado-based RTL design and verification:
- Project creation (GUI & Tcl scripted)
- SystemVerilog vs Verilog
- Three abstraction levels (gate-level, dataflow, behavioral)
- Combinational and sequential building blocks
- FSM design patterns
- Testbench patterns
- Constraints (XDC)
- Debugging in Vivado
- Common pitfalls

**Note**: For repository structure and project organization, see the main [README.md](./README.md).

---

## 1. Creating a Project — GUI Flow

1. `File → New Project` → Name it, choose location (put it *inside* your assignment folder or point Vivado to generate a scratch `build/` dir you `.gitignore`).
2. **Project Type**: RTL Project → check "Do not specify sources at this time" if unsure, or add now.
3. **Add Sources**: design files go under `Design Sources`; the testbench goes under `Simulation Sources` (tick the "Simulation" checkbox when adding, or drag later in the Sources pane).
4. **Add Constraints** (only needed once you target real hardware or check timing): `.xdc` file.
5. **Default Part**: for practice/simulation-only work, part choice barely matters — pick any Artix-7, e.g. `xc7a35tcpg236-1` (Basys3-class), or Zynq `xc7z020clg400-1` if following IIT-G board examples.
6. Finish → you land in the Vivado IDE with the **Flow Navigator** on the left.

**Flow Navigator — the only 5 buttons you need daily:**
| Section | What it does |
|---|---|
| PROJECT MANAGER → Sources | add/remove/organize files |
| SIMULATION → Run Behavioral Simulation | opens XSIM, runs your testbench, shows waveform |
| RTL ANALYSIS → Elaborated Design | schematic view *before* optimization — literal translation of your RTL |
| SYNTHESIS → Run Synthesis → Open Synthesized Design | schematic *after* optimization — what hardware Vivado actually inferred; compare vs your gate-level file here |
| IMPLEMENTATION → Run Implementation | place & route — only needed once targeting real silicon/timing closure |

---

## 2. Creating a Project — Tcl (scripted, reproducible, GitHub-friendly)

This is the version you should actually commit to your repo as `run.tcl`, so anyone (including future-you) can rebuild the project from source with one command.

```tcl
# run.tcl
create_project my_proj ./build -part xc7a35tcpg236-1 -force

add_files -norecurse {gate_level.sv dataflow.sv behavioral.sv}
add_files -fileset sim_1 -norecurse tb.sv
set_property top tb [get_filesets sim_1]
set_property top dataflow_module [current_fileset]

update_compile_order -fileset sources_1
update_compile_order -fileset sim_1

launch_simulation
run all
```

Run it headless from terminal:
```bash
vivado -mode batch -source run.tcl
```
Or drop into an interactive Tcl console:
```bash
vivado -mode tcl
```

**Command-line simulation without opening the GUI at all** (fastest loop while iterating):
```bash
xvlog --sv design.sv tb.sv        # compile (use -sv for SystemVerilog files)
xelab -debug typical tb -s tb_sim # elaborate, create snapshot "tb_sim"
xsim tb_sim -runall               # run to completion, prints $display output
xsim tb_sim -gui                  # or open waveform GUI instead of -runall
```

---

## 3. SystemVerilog vs Verilog — pick SV

Use `.sv` extension, not `.v`, for everything new. Fully supported in Vivado 2026.1, and gives you cleaner constructs that map 1:1 to what you'll see in real job codebases.

| Old (Verilog-2001) | New (SystemVerilog) | Why switch |
|---|---|---|
| `reg` / `wire` | `logic` | one type for almost everything; `reg` was never really "a register," it just confused people |
| `always @(*)` | `always_comb` | tool warns you if you accidentally infer a latch |
| `always @(posedge clk)` | `always_ff @(posedge clk)` | tool warns you if you write it wrong (e.g. mixed blocking/non-blocking) |
| plain `parameter` state values | `typedef enum` | states have names in waveform viewer instead of raw bits |
| `input`/`output` only | `interface` (advanced, later) | bundles related signals — skip for now, learn once designs get bigger |

---

## 4. Data Types & Operators Quick Reference

**Types**
```systemverilog
logic a;              // single bit, 4-state (0,1,x,z)
logic [7:0] b;         // 8-bit vector, MSB first (bit 7 down to 0)
logic [3:0] mem [0:15]; // 16-deep, 4-bit-wide memory array
integer i;              // 32-bit signed, simulation-only (loops)
```

**Operators**
| Category | Operators |
|---|---|
| Bitwise | `& \| ^ ~ ~^ ^~` |
| Logical | `&& \|\| !` |
| Reduction (collapse a vector to 1 bit) | `&vec` (AND all bits), `\|vec`, `^vec` (parity) |
| Relational | `> < >= <=` |
| Equality | `== != === !==` (`===`/`!==` also compare `x`/`z`, use only in testbenches, never in synthesizable RTL) |
| Shift | `<< >> <<< >>>` (last two = arithmetic, sign-extending) |
| Concatenation | `{a, b, c}` |
| Replication | `{4{1'b0}}` → `0000` |
| Ternary (dataflow mux) | `sel ? a : b` |

**Number literals**: `4'b1010`, `8'hFF`, `3'd5`, `16'o17` — `<width>'<base><value>`, base = `b`/`h`/`d`/`o`.

---

## 5. Three Abstraction Levels — Templates

### 5.1 Gate-level (structural)
```systemverilog
module and_or_gate (input a, b, c, output y);
    wire w1;
    and g1 (w1, a, b);   // instantiate primitive: output first, then inputs
    or  g2 (y, w1, c);
endmodule
```
Built-in primitives: `and or not nand nor xor xnor buf`. No `#()` parameter list — just `gate_name (out, in1, in2, ...);`. Multi-input gates like `and(y,a,b,c,d)` are legal.

### 5.2 Dataflow (continuous assignment)
```systemverilog
module and_or_dataflow (input a, b, c, output y);
    assign y = (a & b) | c;
endmodule
```
Use for anything expressible as a single Boolean equation or a mux chain via `?:`. This is where reduced SOP/POS expressions go directly.

### 5.3 Behavioral (procedural)
```systemverilog
module and_or_behavioral (input a, b, c, output logic y);
    always_comb begin
        case ({a,b,c})
            3'b110, 3'b111, 3'b011, 3'b001, 3'b101: y = 1;
            default: y = 0;
        endcase
    end
endmodule
```
Use `always_comb` for combinational, `always_ff @(posedge clk)` for sequential. This is where truth-table-driven or complex control logic naturally lives.

---

## 6. Combinational Building Blocks (memorize these — they recur everywhere)

**2:1 Mux**
```systemverilog
assign y = sel ? b : a;
```
**4:1 Mux (case style)**
```systemverilog
always_comb
    case (sel)
        2'b00: y = i0;
        2'b01: y = i1;
        2'b10: y = i2;
        default: y = i3;
    endcase
```
**Decoder (3-to-8)**
```systemverilog
assign y = 8'b1 << sel;   // one-hot output
```
**Priority Encoder**
```systemverilog
always_comb begin
    casez (in)
        4'b1???: y = 2'd3;
        4'b01??: y = 2'd2;
        4'b001?: y = 2'd1;
        4'b0001: y = 2'd0;
        default: y = 2'd0;
    endcase
end
```

---

## 7. Sequential Building Blocks

**D Flip-Flop (async reset)**
```systemverilog
always_ff @(posedge clk or posedge rst)
    if (rst) q <= 0;
    else     q <= d;
```
**D Flip-Flop (sync reset — generally preferred in modern ASIC/FPGA flows)**
```systemverilog
always_ff @(posedge clk)
    if (rst) q <= 0;
    else     q <= d;
```
**Rule that trips everyone up once**: use **non-blocking `<=`** for all sequential (`always_ff`) logic, **blocking `=`** for all combinational (`always_comb`) logic. Mixing them causes simulation/synthesis mismatches — one of the most common real-world RTL bugs.

**Counter (mod-N)**
```systemverilog
always_ff @(posedge clk or posedge rst)
    if (rst) cnt <= 0;
    else if (cnt == N-1) cnt <= 0;
    else cnt <= cnt + 1;
```
**Shift register**
```systemverilog
always_ff @(posedge clk)
    shreg <= {shreg[6:0], serial_in};
```

---

## 8. FSM Design — the Standard, Reviewable Pattern

Always split into **3 always blocks** (state register / next-state logic / output logic). This separation is what makes an FSM readable in a code review and is close to universal industry style.

```systemverilog
typedef enum logic [1:0] {IDLE, LOAD, RUN, DONE} state_t;
state_t state, next_state;

// 1) State register — the ONLY sequential block
always_ff @(posedge clk or posedge rst)
    if (rst) state <= IDLE;
    else     state <= next_state;

// 2) Next-state logic — purely combinational
always_comb begin
    next_state = state;             // default: hold state (avoids latch inference)
    case (state)
        IDLE: if (start)      next_state = LOAD;
        LOAD:                 next_state = RUN;
        RUN:  if (done_flag)  next_state = DONE;
        DONE:                 next_state = IDLE;
        default:               next_state = IDLE;
    endcase
end

// 3) Output logic — Moore (outputs depend only on state → glitch-free, one cycle latency)
always_comb begin
    out_valid = 1'b0;
    case (state)
        DONE: out_valid = 1'b1;
        default: out_valid = 1'b0;
    endcase
end
```

**Moore vs Mealy**
- Moore: output = f(state) only → stable, no glitches, but one cycle "behind" the triggering input.
- Mealy: output = f(state, input) → reacts same cycle, but can glitch if inputs are combinational/noisy. Use only when you specifically need zero-latency output and inputs are clean.

**FSM checklist before you trust it**
- [ ] Explicit `default:` in every `case` — prevents inferred latches for unreachable states.
- [ ] `next_state = state;` as the first line of the combinational block — same reason.
- [ ] Only ONE `always_ff` block for the state register. Never put next-state logic inside it.
- [ ] State names via `typedef enum` so the waveform viewer shows `IDLE/RUN/DONE` instead of `2'b01`.

---

## 9. Testbench Patterns

**Basic exhaustive check (small input space, e.g. ≤ 3–4 input bits)**
```systemverilog
`timescale 1ns/1ps
module tb;
    logic a, b, c;
    logic y;

    dut_module dut (.a(a), .b(b), .c(c), .y(y));

    integer i;
    initial begin
        for (i = 0; i < 8; i++) begin
            {a,b,c} = i[2:0];
            #10;
        end
        $finish;
    end
endmodule
```

**Self-checking (compares against an expected model — do this, not eyeballing waveforms)**
```systemverilog
initial begin
    for (i = 0; i < 8; i++) begin
        {a,b,c} = i[2:0];
        #10;
        expected = (a&b) | c;              // reference model, computed independently
        if (y !== expected)
            $error("MISMATCH a=%b b=%b c=%b y=%b expected=%b", a,b,c,y,expected);
        else
            $display("PASS a=%b b=%b c=%b y=%b", a,b,c,y);
    end
    $display("ALL TESTS DONE");
    $finish;
end
```

**Clocked DUT testbench skeleton**
```systemverilog
logic clk = 0;
always #5 clk = ~clk;     // 100 MHz-equivalent, 10ns period

initial begin
    rst = 1; #12; rst = 0;   // hold reset across at least one clock edge
    // stimulus here, synced to @(posedge clk)
end
```

---

## 10. Constraints (XDC) — Only Once You Touch Real Timing/Hardware

**Clock definition**
```tcl
create_clock -period 10.000 -name sys_clk [get_ports clk]
```
**I/O pin assignment (board-specific, from your board's master XDC)**
```tcl
set_property PACKAGE_PIN W5 [get_ports clk]
set_property IOSTANDARD LVCMOS33 [get_ports clk]
```
**False path / multicycle (advanced, only when you actually have async or slow paths)**
```tcl
set_false_path -from [get_clocks clkA] -to [get_clocks clkB]
```
**Check timing after implementation:**
```tcl
report_timing_summary -file timing_report.txt
```
Green "Timing Met" in the summary = your design's setup/hold constraints are satisfied at the target frequency. This is the step that separates "it worked in simulation" from "it will actually work on real silicon."

---

## 11. Clock Domain Crossing & Reset — Minimum Foundational Rules

- One clock domain per module wherever possible. Don't casually route a signal across clock domains and hope.
- If you must cross domains, use a **2-flop synchronizer** minimum for single-bit control signals:
```systemverilog
always_ff @(posedge clk_dst) begin
    sync1 <= async_sig;
    sync2 <= sync1;
end
```
- Multi-bit buses crossing domains need a proper FIFO (async FIFO with gray-coded pointers) — never just synchronize each bit independently, they can arrive at different times and corrupt the value.
- Pick sync or async reset **and be consistent across the whole project** — mixing styles module to module is a common review flag.

---

## 12. Debugging Inside Vivado

- **Behavioral sim waveform**: `Simulation → Run Behavioral Simulation`. Right-click any signal → `Add to Wave Window`.
- **`$display` / `$monitor` / `$strobe`**: cheapest debug tool, use liberally in testbenches.
- **ILA (Integrated Logic Analyzer)** — only relevant once you're on real hardware, not simulation: insert a debug core, probe internal signals, capture in real time on the board. Skip until you're doing on-board bring-up.
- **`report_utilization`** — after synthesis, tells you how many LUTs/FFs your design consumed → sanity-check against how complex the design actually is (a 3-input AND-OR shouldn't eat hundreds of LUTs; if it does, you likely have unintended latches or redundant logic).

---

## 13. Common Pitfalls (things that will burn you — read this twice)

| Symptom | Cause | Fix |
|---|---|---|
| Simulation ≠ synthesis behavior | Mixed blocking/non-blocking assignments | `<=` only in `always_ff`, `=` only in `always_comb` |
| Unexpected latch inferred (synthesis warning) | Combinational `always` block doesn't assign output on every path | Add `default:` in every case, assign default value at top of block |
| FSM "stuck" in simulation | Missing default/unreachable-state handling | `next_state = state;` as first line + `default:` in case |
| Glitchy output | Mealy output combined with async/noisy input | Switch to Moore, or register the output |
| Testbench "passes" but design is wrong | Not exhaustively testing all input combos, or comparing against an eyeballed value instead of a computed reference model | Self-checking testbench with independently computed expected value |
| Timing not met after implementation | Combinational path too long between flops | Pipeline (add a register stage) or restructure logic |

---

## 14. Git / GitHub Workflow for HDL Portfolios

- One commit per meaningful step (K-map done → RTL written → testbench passing) — a readable commit history is itself a portfolio signal.
- Each folder's `README.md` should contain: problem statement, K-map/QM work (as a Markdown table, not just an image), final reduced SOP/POS, truth table, and a screenshot/description of the passing simulation.
- Consider a top-level repo `README.md` with a table of contents linking every sub-project — this is what a recruiter actually skims.
- If you want reproducibility bragging rights: include the `run.tcl` per project so `vivado -mode batch -source run.tcl` rebuilds and simulates from a clean checkout.

---

## 15. Roadmap — What "Ultimate Weapon" Actually Points Toward

This assignment level (combinational reduction, 3-style implementation) is the *foundation* layer. The path from here, roughly in order:

1. **Combinational + sequential fundamentals** (where you are now) — muxes, decoders, FFs, counters, simple FSMs.
2. **Standard reusable blocks** — parameterized FIFO (sync and async), UART TX/RX, SPI/I2C controller, simple pipelined ALU. This is the tier that actually resembles interview take-home tasks.
3. **Verification discipline** — self-checking testbenches (you're already doing this), then assertions (`SVA` — `assert property`), then constrained-random stimulus, then (eventually) UVM if you go verification-track.
4. **A small CPU core** — even a minimal RISC-V single-cycle or 5-stage pipeline is one of the strongest portfolio pieces there is; it forces you to combine everything above (FSM control, datapath, hazards, memory interfacing).
5. **Timing closure & physical awareness** — once on real boards, learn to read `report_timing_summary`, understand setup/hold violations, and iterate on pipelining for frequency.

Design vs. Verification fork: design roles want clean, reusable, well-documented RTL like the above; verification roles (currently in *very* high demand relative to design headcount) want you to demonstrate you can break your own designs — self-checking testbenches and SVA assertions are the credential that matters most there. Worth deciding which direction appeals more as you build out the repo, and weighting your later projects accordingly.

---

## 16. One-Page Command Reference

```bash
# CLI simulation (fast loop, no GUI)
xvlog --sv file1.sv file2.sv
xelab -debug typical tb_top -s snapshot_name
xsim snapshot_name -runall
xsim snapshot_name -gui

# Batch mode project build
vivado -mode batch -source run.tcl
vivado -mode tcl                      # interactive Tcl console
```
```tcl
# Tcl — project & sim
create_project name ./dir -part <part> -force
add_files -norecurse {a.sv b.sv}
add_files -fileset sim_1 -norecurse tb.sv
set_property top tb [get_filesets sim_1]
launch_simulation
run all / run 100ns

# Tcl — synth/impl
launch_runs synth_1
wait_on_run synth_1
open_run synth_1 -name synth_1
launch_runs impl_1 -to_step write_bitstream
report_timing_summary -file timing.txt
report_utilization -file util.txt
```
