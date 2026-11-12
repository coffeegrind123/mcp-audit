---
name: mcp-audit
description: "Audit an MCP server for agent usability — not spec compliance — with mcpgrade (a fork with a code-mode profile), plus client compatibility with mcp-lint, then fix what they find and prove the grade moved. Grades tool and parameter descriptions, name collisions, schema design, token cost, server instructions, and for code-first servers whether a model can write a working program or batch; checks which clients (Claude, Cursor, Gemini, OpenAI, ...) accept the schemas; maps findings back to source, applies fixes, and re-scans. Use when the user wants to audit, grade, lint, or score an MCP server, asks why a model picks the wrong tool or hallucinates arguments or ignores a tool, or mentions mcpgrade, missing tool descriptions, or MCP catalog bloat. Examples: 'audit my MCP server', 'run mcpgrade', 'grade the browser server', 'why does it call the wrong tool', 'my tools have no parameter descriptions', 'is my MCP server usable by an agent'."
allowed-tools: Read Write Edit Glob Grep AskUserQuestion Bash(npx:*) Bash(python3:*) Bash(jq:*) Bash(timeout:*) Bash(grep:*) Bash(diff:*) Bash(git status:*) Bash(git diff:*) Bash(git restore:*)
when_to_use: "Use when auditing or improving how usable an MCP server is TO A MODEL, or when diagnosing wrong-tool / bad-argument / no-refusal behaviour that traces back to a tool catalog. Examples: 'audit my MCP server', 'run mcpgrade on X', 'why is the agent calling the wrong tool', 'score my tool descriptions'. Do NOT use for MCP protocol/transport debugging (handshake, JSON-RPC framing, capability negotiation) — that is compliance, a different axis."
argument-hint: "[server name | command | http url | snapshot.json]"
---

# MCP Agent-Usability Audit

An MCP server can be 100% spec-compliant and still unusable by a model. The spec
governs transport; nothing in it makes a model pick the right tool, fill arguments
correctly, or decline when no tool fits. Two pinned tools cover the two remaining axes:

- **mcpgrade fork — `github:coffeegrind123/mcpgrade#v0.5.0-fork.4`**: agent usability.
  Upstream 0.4.0 plus per-tool exceptions, server instructions/resources/prompts, and a
  **code-mode profile** that grades a code-first server (code tool + batch tool +
  searchable API) on what it needs instead of pushing it toward many small tools. Installs
  from a release tag that carries its built `dist/`; nothing compiles locally.
- **`mcp-lint@1.0.4`**: client compatibility — which clients accept the schemas.

Grades and rule IDs shift between versions, so the pins are what make a before/after
comparison meaningful. On a bump, re-run `rules` (mcpgrade) and re-read mcp-lint's rules,
then reconcile `references/rules.md` and `references/mcp-lint.md`.

Below, `MCPGRADE` means `npx -y github:coffeegrind123/mcpgrade#v0.5.0-fork.4` and
`MCPLINT` means `npx -y mcp-lint@1.0.4`.

## Inputs

- Target (optional), taken whole: `$ARGUMENTS`. A server name from the Claude config, a literal command,
  an `http(s)://` URL, or a saved snapshot `.json`. Omit to enumerate everything installed.

## Goal

A named server has a measured grade, the findings are explained in terms of what
they do to a model, the mechanical fixes are applied to the real source, and a
re-scan proves the score moved.

## Gotchas

Each is explained where it bites:

- `~/.claude.json` keeps global servers at the root `.mcpServers`, not under `.projects` (step 1).
- A server's configured `env` must reach the scan, or it hangs (step 2).
- npx's first-run download reads as `-32001 Request timed out` (step 2).
- The grade is at the top of the report: never `tail` it (step 2).
- The report JSON has no top-level `findings` (step 3).
- An installed copy and a checkout can be identical; edit the one the config launches (step 4).
- FastMCP S003 can't be fixed from the signature (step 5).

## Steps

### 1. Resolve the target

With no target, enumerate installed servers from `~/.claude.json`. **The structure
trips people up:** globally-installed servers live at the JSON *root* under
`.mcpServers`; project-scoped ones live under `.projects["<abs path>"].mcpServers`,
and nearly all of those are `{}`.

```bash
python3 -c "
import json, os
d=json.load(open(os.path.expanduser('~/.claude.json')))
for name,cfg in (d.get('mcpServers') or {}).items():
    print('global', name, cfg.get('type'), cfg.get('command') or cfg.get('url'), cfg.get('args'), cfg.get('env'))
for proj,pc in (d.get('projects') or {}).items():
    for name,cfg in (pc.get('mcpServers') or {}).items():
        print(proj, name, cfg.get('type'), cfg.get('command') or cfg.get('url'), cfg.get('args'), cfg.get('env'))
"
```

Present what you found and confirm which to scan. `type: stdio` → `--stdio "<command> <args...>"`.
`type: http` → pass the URL positionally, with `--header "Authorization: Bearer $TOKEN"` for auth.

**Success criteria**: One concrete target with its transport, full argv, and its
configured `env`.
**Artifacts**: target argv, env map, source path.

### 2. Scan

**Carry the server's `env` from the config into the scan command.** A server needing
`DISPLAY=:99` or an API key will hang or die without it, and that reads as a broken
scan rather than a missing variable.

```bash
cd <scratchpad>
DISPLAY=:99 timeout 300 $MCPGRADE --stdio "python /path/run.py" > report.txt 2>&1
DISPLAY=:99 timeout 300 $MCPGRADE --stdio "python /path/run.py" --json --save-snapshot snapshot.json > report.json 2>/dev/null
```

`--save-snapshot` keeps exactly what was graded (tools, instructions, resources,
templates, prompts). mcp-lint lints that file — one connection, and it works for
authenticated servers, which mcp-lint cannot authenticate to itself. Run it from the
scratchpad with an explicit empty config, never inside the server's repo: it imports the
`plugins`/`extends` of any `.mcplintrc.json` it finds walking up from the current
directory (`references/mcp-lint.md`). Write `{}` to `clean-mcplintrc.json` first.

```bash
jq '.tools' snapshot.json > tools.json
timeout 300 $MCPLINT check tools.json --config clean-mcplintrc.json --format json > lint.json 2>lint.err
```

mcp-lint exits 1 when it finds any error; that is a result, not a failure.

**First run of a stdio server through `npx`**: the package download counts against the
client's 60 s initialize timeout and reads as `MCP error -32001: Request timed out`.
Fetch the server package once before the scan (`timeout 200 npx -y <pkg> --version
</dev/null`), then re-run — never conclude "this version regressed" from that error.

Always redirect to a file and read it with `head`/`offset`. **Do not pipe to `tail`** —
the grade and the category bars are at the *top*, and `tail` silently discards the
only numbers that matter.

The server's own logging lands in `report.txt` too (FastMCP prints
`INFO Processing request of type ListToolsRequest`). That is the server talking, not
mcpgrade. Harmless, but never quote it as a finding.

**Success criteria**: `report.txt` shows a grade and its category scores (six in
code-mode); `report.json`, `snapshot.json` and `lint.json` parse.
**Artifacts**: `report.txt`, `report.json`, `snapshot.json`, `lint.json` (keep — step 6
diffs against these).

### 3. Report

The JSON is **nested by category — there is no top-level `findings` array.** Assuming
one is the fastest way to a `TypeError`:

```
{ snapshot: {serverName, toolCount, instructionsChars, resourceCount, promptCount, notes?},
  profile: "classic" | "code-mode", roles: {code, batch, search, docs},
  totalScore, grade,
  categories: [ {category, score, findings: [{ruleId, severity, toolName, param?, message, fix}] } ],
  accepted: [ finding + reason ], unusedExceptions: [...], reach: {catalogTokens, ...} }
```

**Check `profile` and `roles` before anything else.** A tool detected in the wrong role
gets the wrong rules; correct it with `"roles"` in `.mcpgraderc.json` and re-scan.
`snapshot.notes` lists anything the scan could not read (a failed resources/prompts list).

```bash
python3 -c "
import json,collections
d=json.load(open('report.json'))
f=[x for c in d['categories'] for x in c['findings']]
print(d['snapshot']['serverName'], d['grade'], d['totalScore'], d['snapshot']['toolCount'],'tools')
for c in d['categories']: print(f\"{c['category']:8} {c['score']:3}\")
for (r,s),n in sorted(collections.Counter((x['ruleId'],x['severity']) for x in f).items()):
    print(f'{r:6} {s:8} {n:4}')
print('worst tools:', collections.Counter(x['toolName'] for x in f if x.get('toolName')).most_common(8))
"
```

Then tally `lint.json` by rule, severity and schema root as in `references/mcp-lint.md`
— never quote its raw count, which is mostly client warnings and output schemas.

Report the profile and roles, the grade, the category scores, a rule tally, the tools
carrying the most findings, and the client-compatibility tally triaged (universal input
errors first; client-specific findings only for clients the user targets — ask which).
Collapse the overlaps listed in `mcp-lint.md` into one finding each. Present client
claims as claims with the client named; verify one before calling it a defect.
**Explain findings by consequence, not by rule ID** — a name collision means the model
picks the wrong one; an undocumented parameter means it invents a value. Read
`references/rules.md` for any rule you need to explain precisely.

**Success criteria**: User has the grade, the category breakdown, and a tally
ordered by leverage.

### 4. Locate the source

Map findings to files before touching anything. Descriptions come from docstrings
(FastMCP), `.describe()` (zod), or literal schema JSON — grep the exact text from a
D002 finding to land on the definition:

```bash
grep -rn "Navigate to a URL" <server-src>/
```

**Beware duplicate trees.** An installed copy (`/opt/foo-mcp`) and a working checkout
(`~/Foo-MCP`) can both exist with identical contents; editing the wrong one produces
a re-scan that shows no improvement and looks like the fix failed. `diff -rq` them and
confirm which one the config actually launches — the config's `command` path is the
authority. If they are separate copies, say so and agree which to edit before starting.

Check `git status --porcelain` on the tree. A clean tree makes the whole sweep
revertable with `git restore`; if it is dirty or untracked, tell the user before editing.

**Read the server's own design docs** (README, architecture/plan docs, comments at the
tool registrations) before step 5. A server built around few tools and a code or batch
tool will trip size and naming rules on purpose; the design outranks the grader.

**Success criteria**: Confirmed authoritative source tree, the mechanism that produces
the schemas, and known revert path.

### 5. Fix

Read `references/fix-patterns.md` before editing — verified per-framework patterns
and the traps that waste time.

Confirm before editing, and keep the classes separate, because they carry very
different risk:

**Additive (safe — text only, no behaviour change):** D001, D002, D004, D005, D007.
This is the bulk of the score. Every parameter gets a description stating meaning,
format, and one example value; every tool description answers what it does, when to
use it, and what it returns. Write real semantics, not the parameter name restated in
a sentence — `"url"` → *"Absolute URL including scheme; relative paths are rejected.
Example: https://example.com/pricing"*, never *"The url."*

**Schema (low risk):** S002, S003, S004, S005, S006, S007, S008. Note the verified
FastMCP trap: pydantic omits `required` when every parameter has a default, so
**S003 cannot be fixed by editing the signature** — it needs the post-registration
pass in `references/fix-patterns.md`, which is proven to clear it.

**Expect the token score to fall.** Real descriptions cost real tokens, and a server
can pick up a fresh T001 as `desc` goes to 100. Report both numbers; do not trim good
descriptions back to protect a token score. On zod stacks, strip zod's schema padding
first (`fix-patterns.md`) — it is pure noise and often pays for the descriptions.

**Compatibility (schema class):** fixes from `lint.json`, only for clients the user
targets and only after the claim is verified. `references/fix-patterns.md` has them.

**A rule that conflicts with the stated design is an exception, not a finding.** Record
it in `.mcpgraderc.json` with a reason — `{"exceptions": [{"rule": "S007", "tool":
"<tool>", "reason": "..."}]}` — rather than splitting tools, renaming, or using `disable`
(which hides the same problem in every other tool). Chasing the grade into more, smaller
tools or renamed entry points undoes a code-first design and breaks its users; propose
that only when the user wants the design changed.

**Breaking (requires its own explicit confirmation):** N002, N003, C001, T004, T005, and
any split or merge of tools (S007/T002 "split the tool").
Renaming `get_cookies`→`list_cookies` or splitting a 56-tool catalog invalidates
existing configs, saved skills, and prior transcripts. State the blast radius first,
then grep for every reference — `~/.claude.json`, other skills, docs, README, tests,
client code — and update them in the same pass.

Prefer targeted `Edit` calls over rewriting files.

**Human checkpoint**: Explicit approval per class. Breaking changes need a second
confirmation naming what will break.
**Rules**: Never edit a tree you have not confirmed is authoritative. Never let a
rename land without updating its references. Never disable a rule to raise a score
without saying so in the report.
**Success criteria**: Edits applied; the server still imports/builds (import the
module or run the build — do not assume).

### 6. Re-scan and diff

Re-run step 2 verbatim, same pinned versions, same env, and report before/after: total,
grade, per-category deltas, and the triaged compatibility tally.

```bash
python3 -c "
import json
a=json.load(open('report.json')); b=json.load(open('report2.json'))
print(f\"{a['grade']} {a['totalScore']} -> {b['grade']} {b['totalScore']}\")
sb={c['category']:c['score'] for c in b['categories']}
for c in a['categories']: print(f\"{c['category']:8} {c['score']:3} -> {sb[c['category']]:3}\")
"
```

If the score did not move, **say so plainly and diagnose** — the usual causes are
editing the wrong tree, a schema built from something other than what you edited, or
a stale build/`__pycache__` artifact. Do not report a fix as successful without the
second number.

**Success criteria**: A second measured grade, stated next to the first.

### 7. Offer the gate and the deeper checks

Offer, do not run:

- **CI gate** — `$MCPGRADE <target> --fail-on error` with `.mcpgraderc.json` carrying
  the per-tool exceptions, and `$MCPLINT check tools.json --clients <targeted> --severity
  error` (exit 1 on errors). Commit the exceptions with their reasons.
- **`--probe`** (stdio only) — calls tools with invalid arguments to grade error
  quality (C003, C004). **It really calls them**, so on a server that drives a
  browser, filesystem, or database this has real side effects. Get explicit consent.
- **`--eval`** — live model test of tool selection and refusal. Needs
  `ANTHROPIC_API_KEY`, or `--eval-base-url` plus `MCPLINT_EVAL_API_KEY`/`OPENAI_API_KEY`
  for an OpenAI-compatible endpoint. Costs pennies. This is the only way to measure the
  **refusal rate** — whether the model declines out-of-scope tasks instead of
  reaching for a plausible-looking tool. Static rules cannot see it, and it is the
  most dangerous failure mode: an agent acting when it should do nothing.

`mcpgrade serve` also runs mcpgrade itself as an MCP server, if grading other servers
should become an always-available tool rather than a skill invocation.

**Success criteria**: User knows the three options and their costs.
