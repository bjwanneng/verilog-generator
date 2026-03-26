---
title: verilog-generator Skill Design
date: 2026-03-26
status: approved
---

# verilog-generator Skill Design

## Overview

A simplified Claude Code skill that generates a single synthesizable Verilog module from a natural language description or interface definition. No project management, no external tools, no multi-stage pipeline.

## Scope

**In scope:**
- Single Verilog module generation
- Natural language → complete `.v` file
- Interface definition → complete `.v` file
- LLM self-check against coding rules

**Out of scope:**
- Project directory management
- Multi-module / project-level design
- External tool invocation (iverilog, yosys, verilator)
- Simulation / synthesis
- Testbench generation
- Built-in vendor presets (Xilinx/Intel)
- Stage gate approvals
- Experience DB / post-run analysis

## Workflow

Two implicit steps, single user interaction:

```
User input (natural language or interface spec)
    └─> Step 1 (internal): Interface derivation
           Output: module name, parameters, port table
    └─> Step 2 (internal): Code generation + self-check
           Output: complete .v file + self-check summary
```

The user sees both steps in one response. No confirmation required between steps.

## Coding Rules (Enforced)

### Syntax Rules (blocking — regenerate if violated)
- Verilog-2005 syntax only; no SystemVerilog (`logic`, `always_ff`, `always_comb`, `interface`)
- `reg` must not be driven by `assign` — use `wire` instead
- No forward references — declare before use
- No `reg` declarations inside unnamed `begin...end` blocks
- No placeholder code (`// TODO`, `// placeholder`, empty module bodies)
- No multi-driver conflicts (a signal may not be driven by both `always` and `assign`)
- No `$display` or `$finish` in synthesizable code (testbench only)

### AXI-Stream Rules (applied when AXI-Stream interface is detected)
- `valid` must be held HIGH until `ready` acknowledges (`valid && ready`)
- `tdata` must not change while `valid=1` and `ready=0`
- `valid` must not be deasserted before `ready` is seen

### Coding Style

**Default (no style file provided):**
- Module names: `snake_case`
- Signal names: `snake_case`
- Parameter names: `UPPER_CASE`
- Reset: asynchronous active-low (`rst_n`)
- Indentation: 4 spaces
- All generated modules must be complete and synthesizable

**Custom style file (optional):**

Users may provide one or both of:
- A **Markdown rule file** (`.md`) — describes naming conventions, reset style, indentation, comment style, etc. Skill reads and applies the rules described.
- A **Verilog template file** (`.v`) — a reference module demonstrating the desired style. Skill infers style from the example (signal naming, spacing, always block structure, comment patterns) and applies it.

When both are provided, Markdown rules take precedence for explicit rules; the `.v` template fills in style details not covered by the Markdown.

If a custom style conflicts with a syntax rule (e.g., template uses SystemVerilog syntax), the syntax rule wins and a `[WARN]` is emitted in the self-check summary.

## Output Format

```
## 接口推导

模块名: <name>
参数: <PARAM = value, ...>
端口:
  input  clk
  input  rst_n
  ...
  output ...

---

## 生成代码

```verilog
module <name> (...);
...
endmodule
```

---

## 自检摘要

[PASS] Verilog-2005 语法
[PASS] reg/wire 驱动规则
[PASS] 无前向引用
[PASS] 无多驱动冲突
[PASS] 无占位符
[PASS/WARN] AXI-Stream 握手规则（如适用）
```

## Skill File Location

```
verilog-generator/
└── verilog-generator/
    └── SKILL.md
```
