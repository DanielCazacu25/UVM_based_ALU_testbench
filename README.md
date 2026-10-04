# ALU Verification Environment (UVM Testbench)

This project implements a UVM-based, coverage-driven verification environment for a 4-bit ALU supporting 12 operations (arithmetic, comparison, gray-code conversion, logarithmic shifting, and bitwise logic), following the standard UVM layered architecture (sequence, sequencer, driver, monitor, scoreboard, reference model, functional coverage).

## Design Under Test (DUT)

A combinational ALU with:

- 4-bit operands (`A`, `B`), 4-bit opcode, 8-bit result
- 12 supported operations, selected via a `typedef enum` mirrored on the verification side (`ADD`, `SUB`, `MUL`, `COMPARE`, `BIN_TO_GRAY`, `GRAY_TO_BIN`, `SHIFT_L`, `SHIFT_R`, `LOGIC_NAND`, `LOGIC_NOR`, `LOGIC_XNOR`, `LOGIC_NOT`)
- Built from independently designed and tested sub-modules (adder/subtractor with carry-lookahead, array multiplier, 4-bit comparator, binary↔gray converters, logarithmic barrel shifter, 4-bit logic unit)

```verilog
module ALU_Design(input [3:0] A,
                  input [3:0] B,
                  input [3:0] Opcode,
                  output reg [7:0] Result);
```

Each sub-module (adder/subtractor, multiplier, comparator, gray-code converters, shifter, logic unit) was designed by me and tested in isolation before integration into the top-level ALU — see the `Design/Operations_Design` folder for the individual RTL modules. Those standalone tests were plain Verilog testbenches that applied every input combination through nested loops, but the outputs were checked by hand rather than by a self-checking model. That gap — exhaustive stimulus, manual checking — is exactly what this UVM environment closes: every result is now compared automatically against a reference model.

## Verification architecture

The testbench follows the standard UVM layered architecture, with all class files grouped into a single package (`ALU_pkg.sv`) to guarantee correct compile order regardless of the simulator's automatic dependency resolution.

Data flow, at a glance:

`Sequence → Sequencer → Driver → DUT (ALU) → Monitor → { Scoreboard (with the reference model inside), Coverage }`, connected through a single `uvm_analysis_port` broadcasting each observed transaction to both the scoreboard and the coverage collector.

- **Sequence** – generates constrained-random `ALU_seq_item` transactions (opcode constrained to the 12 valid values via `inside`, with a weighted distribution on `A`/`B`: 20% min, 20% max, 60% mid-range).
- **Driver** – drives each transaction onto the interface, with a small propagation delay before signaling completion, to let the combinational logic settle before the next transaction.
- **Monitor** – passively observes the interface (triggered on any change of `A`, `B`, or `Opcode`), reconstructs the completed transaction (inputs + result), and broadcasts it via `analysis_port`.
- **Reference model** – a plain SystemVerilog class (independent of the UVM library) implementing a behavioral "golden" model of all 12 operations, built by tracing each DUT sub-module's RTL rather than assumed from the high-level operation name.
- **Scoreboard** – compares the DUT's actual result against the reference model's prediction and reports PASS/FAIL per transaction, plus a final PASS/FAIL summary in `report_phase`.
- **Coverage** – a `uvm_subscriber`-based collector tracking opcode coverage (with illegal-bin protection on the 4 unused opcode values), boundary-value bins (zero/mid/max) on `A` and `B`, and cross coverage between opcode and each operand (excluding the irrelevant `Op × B` cross for `LOGIC_NOT`, which ignores `B` entirely).

Transaction count per run: 1000 (adjustable via `repeat()` in the sequence).

## A real bug found during development: implicit bit-width extension in the reference model

While bringing up the reference model, the scoreboard reported failures on every `LOGIC_NAND`/`LOGIC_NOR`/`LOGIC_XNOR`/`LOGIC_NOT` transaction, and on every `SHIFT_L` transaction that shifted a `1` out of the 4-bit range, even though the DUT itself was correct.

**Root cause:** the reference model computed these operations directly into the 8-bit result variable, e.g. `res = ~(a & b);`, where `a`/`b` are 4-bit operands. SystemVerilog extends operands to the width of the assignment target _before_ applying the operator — so `~(a & b)` was evaluated as a bitwise NOT over all 8 bits, not just the lower 4, producing an inverted upper nibble that the DUT (which computes the operation strictly on 4 bits before zero-extending the result) never produces. The same issue affected `SHIFT_L`: bits that should overflow out of the 4-bit shifter were instead retained by the wider 8-bit context.

**Fix:** compute each of these operations into an explicit 4-bit intermediate variable first, forcing SystemVerilog to evaluate the operator in the correct (narrow) context, then zero-extend the result — mirroring exactly what the DUT's RTL does:

```systemverilog
LOGIC_NAND : begin
  logic_res = ~(a & b);      // evaluated strictly on 4 bits
  res = {4'd0, logic_res};   // zero-extended afterwards, same as the DUT
end
```

This is a good general lesson for reference models: an operation that looks trivially correct in isolation can silently pick up the wrong bit width once it's written directly into a wider result variable — always match the DUT's internal bit width explicitly, rather than relying on the surrounding context to get it right.

A second, related lesson came from the `SUB` operation: instead of using SystemVerilog's `-` operator directly (which computes a mathematically correct but differently-structured result), the reference model reproduces the DUT's actual implementation — addition with the two's complement of `B` (`a + ~b + 1`) — so that the carry/borrow bit lines up exactly with the DUT's `{carry_out, sum_dif_result}` structure.

## Testbench validation via mutation testing (bug injection)

Following the same methodology used in the [RAM project](https://github.com/DanielCazacu25/Class-Based-RAM-Testbench-SV), each deliberate RTL fault lives on its own `bug-injection/*` branch, starting from a clean copy of `main`, and is never merged back — the branch exists only as proof that the UVM testbench actually detects the fault, not as part of the shipped design. All results below were produced by running the exact same `ALU_test` (1000 constrained-random transactions) against the faulty branch, with no changes to the testbench itself.

| Branch                                | Injected fault                                                                                                                                                                                                                                                                                 | Result                                                                                                                                                                                                                                                                                                                  |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bug-injection/mux-opcode-swap`       | In the top-level `case (Opcode)` mux: `MUL` (`0010`) ↔ `COMPARE` (`0011`) swapped, and `SHIFT_R` (`0111`) ↔ `LOGIC_NAND` (`1000`) swapped — each pair routes a _different_ internal signal to `Result`, so the mutation is guaranteed to be observable (see note below on equivalent mutants). | **676 / 1000 PASS, 324 FAIL** — all failures isolated to the four affected opcodes (`MUL`, `COMPARE`, `SHIFT_R`, `LOGIC_NAND`); the other 8 operations kept passing at 100%, confirming the fault was correctly localized rather than corrupting the whole environment.                                                 |
| `bug-injection/logic-stuck-at-bit0`   | In `Logic_4bit`: `Y[0]` forced to `1'b0` after the `case` block, regardless of the selected operation — a classic stuck-at-0 fault, applied uniformly across all four logic operations (`NAND`, `NOR`, `XNOR`, `NOT`).                                                                         | **845 / 1000 PASS, 155 FAIL** — failures confined to the four logic operations, but only on the subset of transactions where the _correct_ LSB happened to be `1` (see note below on partial detectability). The other 8 operations kept passing at 100%.                                                               |
| `bug-injection/comparator-lt-gt-swap` | In `comparator_1bit`: `lt`/`gt` logic swapped (`assign lt = A & ~B;` / `assign gt = ~A & B;`, inverted from the correct definitions) — a pure logic-inversion fault inside a base sub-module, distinct from both the routing fault (mux swap) and the partial stuck-at fault above.            | **924 / 1000 PASS, 76 FAIL** — failures confined to `COMPARE` transactions where `A ≠ B` (~76, matching the expected ~1/16 chance of `A == B` among the ~83 `COMPARE` transactions generated); `COMPARE` transactions with `A == B` still passed, since `eq` was untouched and both inverted bits are `0` in that case. |

**Note on partial detectability of stuck-at faults:** unlike the mux swap above, a stuck-at fault on a single bit is only observable on inputs where that bit's correct value differs from the stuck value. For example, `LOGIC_XNOR` with `A=15, B=8` correctly produces `Res=8` (`4'b1000`) — the LSB is already `0`, so forcing it to `0` changes nothing, and the transaction passes despite the fault being present. This is expected, not a testbench gap: it's a concrete illustration of why mutation testing reports a _pass rate_, not a binary detected/not-detected result, and why a single directed test is not enough to guarantee a stuck-at fault is caught — only broad, randomized coverage across many input combinations makes detection reliable.

**Note on equivalent mutants:** a mutation is only useful if it actually changes the design's behaviour. Swapping the `ADD` (`0000`) and `SUB` (`0001`) lines of the top-level mux, for example, changes nothing: both lines route the same signal (`{3'b000, carry_out, sum_dif_result}`), and the add/subtract choice is made inside `add_sub_4bit` from `Opcode[0]`. The same holds for `SHIFT_L`/`SHIFT_R`, whose direction comes from a separate `shift_dir` signal. Such an *equivalent mutant* cannot be detected by any test, so it says nothing about the testbench. That is why every pair swapped in `mux-opcode-swap` routes a *different* internal signal to `Result`.

## Stimulus-level error injection via UVM callbacks

Mutation testing (above) validates the testbench against **RTL faults**. As a separate, complementary technique, this project also demonstrates **stimulus-level negative testing** using the `uvm_callback` mechanism — injecting invalid or corrupted data at the driver, without touching the DUT or the base testbench at all.

**Why callbacks instead of just writing another sequence:** a callback lets error-injection behavior be attached to (or removed from) a specific driver instance at runtime, without creating a new driver subclass or modifying `ALU_driver`'s source. This mirrors how verification IP is extended in practice, where the base component is often not meant to be edited directly.

**Implementation:**

- `ALU_driver_cb` — a `uvm_callback` base class declaring an empty `modify_pkt(ALU_seq_item seq_itm)` hook.
- `ALU_driver` registers the callback type (`` `uvm_register_cb(ALU_driver, ALU_driver_cb)``) and invokes the hook (`` `uvm_do_callbacks(ALU_driver, ALU_driver_cb, modify_pkt(sq_itm))``) **after** receiving a transaction from the sequencer but **before** driving it onto the interface — so any modification made by an attached callback actually reaches the DUT.
- `ALU_error_inject_cb extends ALU_driver_cb` overrides the hook to, with low probability, replace operand `A` with a new random value (~10% of transactions) or force an out-of-range, undefined opcode (`4'b1100`, ~9% of transactions). The out-of-range opcode is the actual negative test. Replacing `A` is not an error on its own, since every 4-bit value is a legal operand; what it does show is that the scoreboard checks what the monitor observed on the interface, not what the sequence intended, so a value changed after the sequence still produces a correct comparison.
- `ALU_error_inject_test extends ALU_test` attaches the callback to the running driver instance (`uvm_callbacks#(ALU_driver, ALU_driver_cb)::add(envir.agn.drv, err_cb)`) in `connect_phase` — not `build_phase`, since the driver instance doesn't exist yet at that point in the phase hierarchy — leaving `ALU_test` itself completely untouched.

**Result: 1000 / 1000 PASS, 0 FAIL** — even with the callback active. Two things make this a meaningful pass rather than a no-op test:

1. **Undefined-opcode transactions still pass, for a documented reason.** Both the DUT (`default: Result = 8'b0;` in the top-level mux) and the reference model (`default: res = 8'd0;`) handle an out-of-range opcode identically, so the scoreboard correctly reports a match. This is itself a useful confirmation that the DUT's default behavior and the reference model's default behavior are consistent — not something to be taken for granted.
2. **The functional coverage's `illegal_bins` correctly fired.** Every time the callback forced the invalid opcode, the coverage engine raised an error for hitting `Op_cp`'s `illegal_bins`, exactly as designed — proof that the coverage model actively flags out-of-spec inputs rather than silently ignoring them. (The scoreboard log shows these transactions with an empty operation name, e.g. `PASS [] : A=13 | B=15 -> Res=0` — `alu_op_e'(4'b1100)` casts to a value with no associated enumerator, so `.name()` returns an empty string, which is expected SystemVerilog behavior for unnamed enum values.)

## Known limitations

**The monitor detects a transaction only when `A`, `B` or `Opcode` changes** (`@(inf.A or inf.B or inf.Opcode)`). If the sequence generates two identical transactions in a row, no signal changes, so the second one is never observed: it is neither checked by the scoreboard nor sampled by coverage, and the only symptom is a transaction total below 1000. The standard fix is an explicit sampling event, such as a `valid` strobe in the interface or a testbench clock; the later [FIFO](https://github.com/DanielCazacu25/UVM_Based_Async_FIFO_Testbench) and [AXI4-Lite](https://github.com/DanielCazacu25/UVM_Based_AXI4_Lite_Testbench) projects sample on clock edges and on the `VALID`/`READY` handshake for this reason. The ALU monitor is deliberately left unchanged, as a record of how the approach evolved across projects.

## Project structure

| Folder/File                             | Description                                                                                                     |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `Design/ALU_Design/ALU_Design.v`        | The ALU DUT top-level module (Verilog)                                                                          |
| `Design/Operations_Design/`             | Individual sub-module RTL (adder/subtractor, multiplier, comparator, gray-code converters, shifter, logic unit) |
| `Testbench/ALU_pkg.sv`                  | Package including all UVM class files, in dependency order                                                      |
| `Testbench/ALU_inf.sv`                  | Virtual interface (`A`, `B`, `Opcode`, `Result`)                                                                |
| `Testbench/ALU_seq_item.sv`             | Transaction class (`ALU_seq_item`), with opcode/value constraints                                               |
| `Testbench/ALU_sequencer.sv`            | Sequencer                                                                                                       |
| `Testbench/ALU_sequence.sv`             | Sequence generating constrained-random transactions                                                             |
| `Testbench/ALU_driver.sv`               | Driver                                                                                                          |
| `Testbench/ALU_driver_cb.sv`            | `uvm_callback` base class for the driver (empty `modify_pkt()` hook)                                            |
| `Testbench/ALU_monitor.sv`              | Monitor                                                                                                         |
| `Testbench/ALU_agent.sv`                | Agent (driver + monitor + sequencer)                                                                            |
| `Testbench/ALU_ref_model.sv`            | Reference (golden) model, traced from DUT sub-module RTL                                                        |
| `Testbench/ALU_scoreboard.sv`           | Scoreboard                                                                                                      |
| `Testbench/ALU_coverage.sv`             | Functional coverage collector                                                                                   |
| `Testbench/ALU_env.sv`                  | Top-level environment class, connects agent, scoreboard, and coverage                                           |
| `Testbench/ALU_test.sv`                 | Test class, starts the sequence on the environment's sequencer                                                  |
| `Testbench/ALU_error_inject_cb.sv`      | Callback overriding `modify_pkt()` to corrupt an operand or force an invalid opcode                             |
| `Testbench/ALU_error_inject_test.sv` | Test extending `ALU_test`, attaching `ALU_error_inject_cb` to the driver instance                               |
| `Testbench/ALU_top.sv`                  | Testbench top: interface instantiation, DUT connection, `run_test()`                                            |

## How to run

1. Open Vivado and create a new project.
2. Add all files under `Design/` (including `Design/ALU_Design/` and `Design/Operations_Design/`) as design sources.
3. Add `Testbench/ALU_pkg.sv`, `Testbench/ALU_inf.sv`, and `Testbench/ALU_top.sv` as simulation sources.
   - **Important:** do not add the individual class files (`ALU_driver.sv`, `ALU_monitor.sv`, etc.) as separate simulation sources — they are pulled in through `` `include`` inside `ALU_pkg.sv`. Adding them both individually and through the package causes redefinition errors. Grouping all UVM classes into a package this way also avoids compile-order issues, since the simulator otherwise resolves file order from static instantiation hierarchy, which doesn't apply to classes selected dynamically via `run_test()`.
4. Set the simulation top module to `ALU_top`.
5. Run behavioral simulation (`launch_simulation` / Run All). Vivado launches with a default runtime of 1000 ns, which is not enough to complete all 1000 randomized transactions plus their propagation delays and the sequence's drain time.
6. **After** the initial launch, type `run -all` in the Tcl console and press Enter, so the simulation runs until UVM itself calls `$finish` (after `report_phase`/`final_phase`), instead of stopping at a fixed, guessed time value:

  ![Tcl run -all](results/main/RunAll.png)

7. Check the Tcl console for the scoreboard summary (PASS/FAIL counts) and the functional coverage percentage.

To reproduce the mutation-testing results, check out the relevant branch (e.g. `git checkout bug-injection/<fault-name>`) before running the simulation, and compare against `main`.

- Note: `ALU_top.sv` contains two `run_test()` calls: `run_test("ALU_test")` (the main regression, 1000/1000 PASS) and `run_test("ALU_error_inject_test")` (the callback-based error injection demo). Only one should be uncommented at a time — comment out the other before running.

## Git workflow

This project uses branches to isolate experiments from the main, verified codebase:

- `main` — correct, working DUT and UVM testbench.
- `bug-injection/*` — each branch introduces exactly one deliberate design fault, starting from a clean `main`, to validate that the testbench detects it. These branches are not merged into `main`.

## Results

### Main

#### Summary

![Summary](results/main/Summary.png)

#### Example transactions per operation

![Operations](results/main/Operations.png)

### Branch: mux-opcode-swap

#### Summary

![Summary](results/opcode_swap/Summary.png)

#### Example transactions per operation

![Operations](results/opcode_swap/Operations.png)

### Branch: logic-stuck-at-bit0

#### Summary

![Summary](results/stuck_at_bit/Summary.png)

#### Example transactions per operation

![Operations](results/stuck_at_bit/Operations.png)

### Branch: comparator-lt-gt-swap

#### Summary

![Summary](results/comparator-lt-gt-swap/Summary.png)

#### Example transactions per operation

![Operations](results/comparator-lt-gt-swap/Operations.png)

### Stimulus-level error injection (ALU_error_inject_test)

#### Summary

![Summary](results/error_injection/Summary.png)

_(1000/1000 PASS — see the "Stimulus-level error injection via UVM callbacks" section above for why this is a meaningful result, not a no-op test.)_

#### Example transactions, including invalid-opcode injections

![Operations](results/error_injection/Operations.png)

_(Note the transactions with an empty operation name, e.g. `PASS [] : A=13 | B=14 -> Res=0` — these are the injected out-of-range opcode transactions; the empty name is expected, since `ALU_op_e'(4'b1100)` has no associated named enumerator.)_

#### Example of illegal bins error

![Operations](results/error_injection/il_bins.png)