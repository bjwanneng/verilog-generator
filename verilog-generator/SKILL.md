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
- NEVER use `$display`, `$finish`, `$monitor` outside of testbenches
- These constructs are only valid in simulation; the generated module must be synthesizable

### Rule 6: AXI-Stream handshake (applies when AXI-Stream interface is detected)
- `valid` MUST be held HIGH until `ready` acknowledges: `if (valid && ready)`
- `tdata` MUST NOT change while `valid=1` and `ready=0`
- NEVER deassert `valid` before `ready` is seen

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
  input  wire        rst_n
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
[PASS/FAIL/N/A] AXI-Stream 握手规则
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
| Tunable parameter | `UpperCamelCase` |
| Localparam constant | `ALL_CAPS` |
| Clock | starts with `clk`; main clock = `clk` |
| Reset | active-low asynchronous `rst_n` |
| Port direction suffixes | `_i` input, `_o` output, `_io` bidir |
| Register pair suffixes | `_d` next-state, `_q` current-state |
| Active-low suffix | `_n` |
| Indentation | 2 spaces |
| Max line length | 100 characters |
| Sequential block | `always @(posedge clk or negedge rst_n)` |
| Combinational block | `always @(*)` |
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
