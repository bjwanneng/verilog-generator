# verilog-generator

A Claude Code skill that generates single, synthesizable Verilog modules from natural language descriptions or interface definitions.

## Features

- **Natural language to RTL** — describe the module in plain text, get a complete Verilog implementation
- **Verilog-2005 strict mode** — enforces pre-SystemVerilog syntax, no `logic`, `always_ff`, or `interface`
- **Full AXI protocol support** — correct handshake validation for AXI-Stream, AXI4-Lite, and AXI4-Full
- **14-point self-check pipeline** — every generated module passes a comprehensive checklist before output
- **Customizable coding style** — supply a Markdown rule file and/or a Verilog template to override defaults

## File Structure

```
verilog-generator/
├── SKILL.md          # Skill definition: mandatory rules, workflow, output format
├── coding_style.md   # Built-in coding style guide (27 sections, lowRISC adapted for Verilog-2005)
├── README.md         # This file
└── .gitignore
```

## How It Works

When invoked, the skill follows a two-step workflow:

1. **Interface Derivation** — parses the user's description and displays the inferred module name, parameters, and port list
2. **Code Generation + Self-Check** — generates the complete module and runs a 14-point checklist

### Self-Check Items

| # | Check | Scope |
|---|-------|-------|
| 1 | File header completeness | File, Author, Date, Description, Change Log |
| 2 | File structure | `resetall` / `timescale` / `default_nettype` / `endmodule` / `resetall` |
| 3 | Verilog-2005 syntax | No SystemVerilog constructs |
| 4 | reg/wire driving rules | `reg` for `always`, `wire` for `assign` |
| 5 | No forward references | All signals declared before use |
| 6 | No multi-driver conflicts | One driver per signal |
| 7 | No placeholders | Complete, synthesizable implementation |
| 8 | Width matching | All assignments and port connections match widths |
| 9 | No unused signals | Every declared signal is read |
| 10 | No latch inference | All `always @*` paths fully assigned |
| 11 | Reset coverage report | Lists unreset registers with justification |
| 12 | AXI handshake rules | AXI-Stream / AXI4-Lite / AXI4-Full |
| 13 | CDC synchronizer check | Multi-clock modules require synchronization |
| 14 | Custom style compliance | User-provided rules applied |

## Coding Style

The built-in style (defined in `coding_style.md`, 27 sections) is adapted from the lowRISC Verilog Coding Style Guide.

### Key Conventions

| Item | Convention |
|------|-----------|
| Module / signal names | `lower_snake_case` |
| Parameters / localparam | `ALL_CAPS` |
| Clock | `clk` (main), `clk_<domain>` (additional) |
| Reset | Synchronous active-high `rst` |
| Register pair | `_reg` (current) / `_next` (next-state) |
| Pipeline register | `_pipe_reg` |
| Synchronizer output | `_sync` |
| Port suffixes | `_i` input, `_o` output, `_io` bidir |
| Indentation | 4 spaces |
| Max line length | 100 characters |

### Style Sections

| Section | Topic |
|---------|-------|
| 1–9 | File structure, formatting, naming, signals, clocks, reset, module declaration, parameters, declarations |
| 10–15 | Output driving, two-block logic separation, latch elimination, case statements, FSM, instantiation |
| 16–19 | Generate constructs, memory arrays, number literals, signed arithmetic |
| 20–22 | Clock domain crossing, pipeline insertion, FSM encoding selection |
| 23–24 | Resource utilization, module partitioning |
| 25–27 | AXI protocol (Stream / Lite / Full), comments, prohibited constructs |

Users can override these rules by providing their own Markdown rule file and/or Verilog template. Priority: user Markdown > user `.v` template > built-in style.

## Mandatory Rules (Non-Negotiable)

1. Verilog-2005 syntax only — no SystemVerilog constructs
2. Correct `reg`/`wire` usage — `reg` for `always`, `wire` for `assign`
3. No placeholder code — every module must be complete and synthesizable
4. No multi-driver conflicts — one driver per signal
5. No simulation-only constructs in synthesizable code (testbench generation encouraged separately)
6. AXI handshake correctness for AXI-Stream, AXI4-Lite, and AXI4-Full
7. Self-check before output — all 14 rules must pass

## Usage

In Claude Code, invoke the skill directly:

```
/verilog-generator
```

Then describe the module you need, for example:

> Generate a parameterized FIFO with AXI-Stream interfaces, data width 32, depth 64.

## License

MIT
