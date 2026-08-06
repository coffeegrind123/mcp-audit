# mcp-audit

A [Claude Code](https://claude.com/claude-code) Agent Skill that audits an MCP server
for **agent usability** — then fixes what it finds and proves the grade moved.

An MCP server can be 100% spec-compliant and still be unusable by a model. The spec
governs transport; nothing in it makes a model pick the right tool, fill arguments
correctly, or decline when no tool fits. This skill drives
[`mcpgrade`](https://www.npmjs.com/package/mcpgrade) against that second axis, maps the
findings back to source, applies the mechanical fixes, and re-scans.

## Install

```bash
git clone https://github.com/coffeegrind123/mcp-audit ~/.claude/skills/mcp-audit
```

The directory name must stay `mcp-audit` — it has to match the `name` in the
frontmatter.

## Use

```
/mcp-audit                          # enumerate installed servers, pick one
/mcp-audit browser                  # a server by name from ~/.claude.json
/mcp-audit ./my-server.js           # a local command
/mcp-audit https://host/mcp         # a remote streamable-HTTP server
```

It runs seven steps: resolve the target, scan, report, locate the source, fix behind a
confirmation, re-scan and diff, then offer a CI gate. Fixes are grouped by risk —
additive text changes are separated from schema changes, and breaking changes
(renames, catalog splits) require their own confirmation plus a reference-update sweep.

## Result on a real server

A 56-tool browser-automation server, `D 66/100` → `A 99/100`:

```
  Descriptions     0 -> 100
  Naming          96 ->  96
  Schema design   89 -> 100
  Token cost     100 ->  98
  Consistency    100 -> 100

  findings  138 -> 5      errors  83 -> 0
```

68 parameter descriptions, 13 expanded tool descriptions, 36 return-value clauses, an
enum, and the `required` arrays. The five remaining findings were deliberate
deferrals — three breaking renames and the catalog-size rules.

## What's in here

| File | Contents |
|---|---|
| `SKILL.md` | The workflow. Loaded when the skill triggers. |
| `references/rules.md` | All 26 mcpgrade rules, what each costs you, and what static analysis cannot see. |
| `references/fix-patterns.md` | Per-framework fix patterns (FastMCP, zod, raw JSON Schema, OpenAPI) and the traps. |

## Traps it encodes

These were paid for in real debugging rounds:

- **`~/.claude.json` structure** — global servers sit at the JSON root under
  `.mcpServers`; project-scoped ones under `.projects["<path>"].mcpServers`.
- **Carry the server's `env`** into the scan, or it hangs and reads as a broken scan.
- **Never `tail` the report** — the grade is at the top.
- **The JSON is nested by category.** There is no top-level `findings` array.
- **FastMCP omits `required` entirely** when every parameter has a default, so S003
  cannot be fixed from the function signature. `fix-patterns.md` has the
  post-registration pass that does clear it.
- **D005 matches the literal word "placeholder"** — describing an input's placeholder
  attribute trips it.
- **N004 wants the tool's name echoed in its description.**
- **Fixing descriptions costs token score**, and can trip T001. That is the correct
  trade, not a regression.

## Credit

`mcpgrade` and its ruleset are by [tengli.dev](https://tengli.dev/posts/mcp-servers-failing-agents.html).
This repo is only the skill that drives it.
