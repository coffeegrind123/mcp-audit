# mcp-audit

A [Claude Code](https://claude.com/claude-code) Agent Skill that audits an MCP server
for **agent usability** — then fixes what it finds and proves the grade moved.

An MCP server can be 100% spec-compliant and still be unusable by a model. The spec
governs transport; nothing in it makes a model pick the right tool, fill arguments
correctly, or decline when no tool fits. This skill drives two pinned tools against the
remaining axes, maps the findings back to source, applies the mechanical fixes, and
re-scans:

- a fork of [`mcpgrade`](https://github.com/tenglidev/mcpgrade),
  [`coffeegrind123/mcpgrade`](https://github.com/coffeegrind123/mcpgrade) — agent
  usability, with per-tool exceptions, server instructions/resources/prompts, and a
  code-mode profile for servers built on a code tool, a batch tool and a searchable API;
- [`mcp-lint`](https://www.npmjs.com/package/mcp-lint) — which clients accept the schemas.

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
| `references/rules.md` | The mcpgrade fork's rules, profiles and exceptions, what each costs you, and what static analysis cannot see. |
| `references/mcp-lint.md` | Running mcp-lint safely, its output, triage, and its overlap with mcpgrade. |
| `references/fix-patterns.md` | Per-framework fix patterns (FastMCP, zod, raw JSON Schema, OpenAPI), compatibility fixes, and the traps. |

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
- **zod v4 omits `required` too** when every field is optional, and `meta({required})`
  replaces the computed list. `fix-patterns.md` has the guarded helper.
- **A server's design outranks the grader.** Splitting tools or renaming a code tool to
  satisfy a size or naming rule undoes a code-first design; record an exception instead.
- **Check the detected roles first** in code-mode: a wrong role means the wrong rules.
- **mcp-lint imports plugins from any `.mcplintrc.json` above the current directory.**
  Run it from a scratch directory with an explicit empty config.
- **Triage mcp-lint, never quote its raw count** — most findings are client warnings and
  output schemas.
- **A first `npx` run of a stdio server can time out** on the download; fetch it once and
  re-run before suspecting the grader.

## Credit

`mcpgrade` and its ruleset are by [tengli.dev](https://tengli.dev/posts/mcp-servers-failing-agents.html);
`mcp-lint` is by robert19001-cmyk. This repo is only the skill that drives them.
