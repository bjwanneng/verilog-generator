---
name: verilog-generator
description: Generates a single synthesizable Verilog module from a natural language description or interface definition. Applies strict Verilog-2005 coding rules, AXI-Stream handshake validation, and optional user-supplied coding style (Markdown rule file and/or Verilog template). Use when the user asks to generate, write, or implement a Verilog module.
license: MIT
metadata:
  author: verilog-generator
  version: "1.0.0"
  category: hardware-design
---

# verilog-generator

Generates a single synthesizable Verilog module from natural language or an interface definition.

## MANDATORY RULES

**These rules are NON-NEGOTIABLE. Violating any rule is a critical error that requires regeneration.**

### Rule 1: Verilog-2005 syntax only
- NEVER use SystemVerilog syntax: `logic`, `always_ff`, `always_comb`, `always_latch`, `interface`, `modport`, `unique case`, `priority case`
- NEVER declare `reg`/`wire` inside unnamed `begin...end` blocks
- NEVER use forward references — declare every signal before the line that uses it

### Rule 2: Correct reg/wire usage
- `reg` is for signals driven by `always` blocks
- `wire` is for signals driven by `assign` statements or module outputs
- NEVER drive a `reg` with `assign`
- NEVER drive a `wire` with an `always` block

### Rule 3: No placeholder code
- NEVER write `// TODO`, `// placeholder`, `// implement later`, or empty module bodies
- Every generated module MUST be a complete, synthesizable implementation

### Rule 4: No multi-driver conflicts
- A signal MUST have exactly one driver: either `always` OR `assign`, never both

### Rule 5: No simulation-only constructs in synthesizable code
- NEVER use `$display`, `$finish`, `$monitor`, `#delay`, or `initial` (except for parameter validation) inside synthesizable modules
- These constructs are only valid in testbenches; the generated module must be synthesizable
- **SHOULD** offer to generate a companion testbench (`tb_<module_name>.v`) after the synthesizable module is complete — simulation constructs are expected and required there

### Rule 6: AXI handshake (applies when any AXI interface is detected)

**AXI-Stream:**
- `valid` MUST be held HIGH until `ready` acknowledges: `if (valid && ready)`
- `tdata` MUST NOT change while `valid=1` and `ready=0`
- NEVER deassert `valid` before `ready` is seen

**AXI4-Lite / AXI4-Full:**
- Each channel (AW, AR, W, R, B) follows independent valid/ready handshake
- `A[W|R]VALID` MUST NOT depend on `A[W|R]READY` — master must assert valid unconditionally
- `A[W|R]READY` MAY be dependent on `A[W|R]VALID`
- `WVALID` MUST NOT have gaps during a burst — all beats must be contiguous
- `BVALID` MUST be held until `BREADY` — response must not be dropped
- `RLAST` MUST be asserted on the final beat of a read burst
- `WLAST` MUST be asserted on the final beat of a write burst
- NEVER deassert `A[W|R]VALID` before `A[W|R]READY` is seen

### Rule 7: Self-check before output
Before outputting the final code block, verify every rule above.
If any rule is violated, fix it silently and re-verify.
Only output code that passes all checks.

---

## WORKFLOW

When this skill is invoked, follow these two steps in a single response:

### Step 1 — Interface Derivation

From the user's input, derive and display:

```
## 接口推导

模块名: <snake_case_name>
参数:
  parameter DATA_WIDTH = 8
  parameter DEPTH      = 16
  ...
端口:
  input  wire        clk
  input  wire        rst
  input  wire [W:0]  <signal>
  output wire [W:0]  <signal>
  ...
```

If a coding style file was provided, apply its naming conventions here.

### Step 2 — Code Generation + Self-Check

Generate the complete module, then display:

````
## 生成代码

```verilog
// -----------------------------------------------------------------------------
// File   : <name>.v
// Author : <infer from context, or leave as "unknown" if not provided>
// Date   : <today's date in YYYY-MM-DD>
// -----------------------------------------------------------------------------
// Description:
//   <one or two lines summarizing the module's purpose>
// -----------------------------------------------------------------------------
// Change Log:
//   <YYYY-MM-DD>  <Author>  v1.0  Initial generation
// -----------------------------------------------------------------------------

`resetall
`timescale 1ns / 1ps
`default_nettype none

module <name> #(
    parameter DATA_WIDTH = 8
)
(
    input  wire        clk,
    input  wire        rst,
    ...
);

// --- internal signals ---
...

// --- logic ---
...

endmodule

`resetall
```

## 自检摘要

[PASS/FAIL] 文件头完整（File/Author/Date/Description/Change Log 均已填写）
[PASS/FAIL] 文件结构（resetall / timescale / default_nettype / endmodule / resetall）
[PASS/FAIL] Verilog-2005 语法（无 SystemVerilog）
[PASS/FAIL] reg/wire 驱动规则
[PASS/FAIL] 无前向引用
[PASS/FAIL] 无多驱动冲突
[PASS/FAIL] 无占位符
[PASS/FAIL] 位宽匹配（所有赋值与端口连接左右宽度一致）
[PASS/FAIL] 无未使用信号
[PASS/FAIL] 无锁存器推断（所有 always @* 路径完整赋值）
[PASS/WARN] 复位覆盖报告（列出未复位的寄存器及原因）
[PASS/FAIL/N/A] AXI 握手规则（AXI-Stream / AXI4-Lite / AXI4-Full）
[PASS/FAIL/N/A] 跨时钟域信号已同步（多时钟模块必须）
[PASS/WARN/N/A] 自定义编码风格已应用
````

If any item shows `[FAIL]`, fix the issue and regenerate before displaying output.

---

## CODING STYLE

### Built-in style: lowRISC (Verilog-2005 adapted)

This skill includes a built-in coding style adapted from the lowRISC Verilog Coding Style Guide.
The full rule set is in `coding_style.md` (same directory as this skill file).

**You MUST read that file before generating any module.** Apply all rules in it.

Key rules from that file:

| Item | Rule |
|------|------|
| Module name | `lower_snake_case` |
| Signal name | `lower_snake_case` |
| Parameter / localparam | `ALL_CAPS` |
| Clock | starts with `clk`; main clock = `clk` |
| Reset | synchronous active-high `rst` |
| Port direction suffixes | `_i` input, `_o` output, `_io` bidir |
| Register pair suffixes | `_next` next-state, `_reg` current-state |
| Pipeline register suffix | `_pipe_reg` |
| Synchronizer suffix | `_sync` |
| Active-low suffix | `_n` |
| Indentation | 4 spaces |
| Max line length | 100 characters |
| Sequential block | `always @(posedge clk)` |
| Combinational block | `always @*` |
| Assignments | `<=` in sequential, `=` in combinational |
| Case | `case` with mandatory `default`; `casez` for wildcards; never `casex` |
| Port order | clocks → resets → other ports |
| Instantiation | named ports only, tabular alignment |

### Custom style file (optional)

The user may additionally provide one or both of:

**Markdown rule file (`.md`)**
Read the file and extract explicit rules. These override the built-in lowRISC rules where they conflict.

**Verilog template file (`.v`)**
Read the file and infer style from the example. Fills in details not covered by the Markdown.

**Priority:** user Markdown rules > user `.v` template > built-in lowRISC style.

**Style vs. syntax conflict:** If any style rule would require violating a mandatory Verilog-2005 syntax rule (e.g., using `logic`), the syntax rule wins and emit `[WARN] 自定义风格与语法规则冲突，已按语法规则覆盖` in the self-check summary.

---

## COMMON ERRORS QUICK REFERENCE

| Error | Root Cause | Fix |
|-------|-----------|-----|
| `Variable 'X' cannot be driven by continuous assignment` | `reg` driven by `assign` | Change `reg X` to `wire X` |
| `Unable to bind wire/reg/memory 'X'` | Forward reference | Move declaration of X before first use |
| `Variable declaration in unnamed block requires SystemVerilog` | `reg` inside unnamed `begin...end` | Move declaration to module level |
| `Multiple drivers on signal 'X'` | Both `always` and `assign` drive X | Pick one driver type and remove the other |
| AXI `valid` glitch | `valid` cleared before `ready` seen | Use `if (valid && ready) valid <= 1'b0;` |
| AXI `BVALID` dropped | `bvalid` deasserted before `bready` | Hold `bvalid` until `bready` acknowledgement |
| AXI `WVALID` gap | Write beat stalled mid-burst | Ensure contiguous `WVALID` assertion during burst |
| Metastability on crossing | No synchronizer between domains | Insert two-flop synchronizer for control, async FIFO for data |
| Width mismatch warning | Narrow signal to wide port | Use explicit padding: `{16'd0, sixteen_bit_word}` |
| Latch inferred for 'X' | Missing assignment in `always @*` path | Add default values at top of combinational block |
