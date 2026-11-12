# mcpgrade rule reference (fork v0.5.0-fork.4)

Load this when you need to explain a specific finding precisely, or decide whether a
rule is worth fixing on a given server. Client-compatibility rules (mcp-lint) are in
`mcp-lint.md`.

The skill pins the fork `github:coffeegrind123/mcpgrade#v0.5.0-fork.4` (upstream
`mcpgrade@0.4.0` plus the rules below marked *fork*). Regenerate after any version bump —
rule IDs and severities are not stable across versions:

```bash
npx -y github:coffeegrind123/mcpgrade#<tag> rules
```

Scored categories: `desc`, `naming`, `schema`, `tokens`, `consistency`, and in the
code-mode profile `prog` (Programmability). There is no D003 — a gap in the numbering,
not a missing rule.

## Profiles (fork)

`--profile auto` (default) detects the tools' roles and picks:

- **classic** — the upstream grade: weights desc 30 / naming 15 / schema 30 / tokens 15 /
  consistency 10.
- **code-mode** — when the server has a **code tool** (a free-text string parameter
  named `code`, `script`, `source`, `program`, a language name; an enum is a mode switch,
  not a program) or a **batch tool** (an array-of-objects parameter named `ops`,
  `operations`, `actions`, `commands`, `steps`, ...). Adds Programmability at 20% and
  scales the rest: desc 25 / naming 10 / schema 25 / tokens 10 / consistency 10 / prog 20.
  N003 does not apply to the code tool.

The report prints the detected roles (`code`, `batch`, `search`, and API `docs`
resources). Check them before reading the grade: a wrong role means the wrong rules.
`"roles": {"<tool>": "code" | "batch" | "search" | "none"}` in `.mcpgraderc.json`
corrects them.

Score anchors observed in the wild: A ≈ 95+, D ≈ 66, F ≈ 57. A single category at 0
(usually `desc`) is enough to drag an otherwise-clean server to D.

## Descriptions (`desc`)

The category that decides most grades. Additive to fix, zero behaviour risk, highest
leverage per hour — the whole point of the ruleset.

| Rule | Sev | What it means | What it costs you |
|---|---|---|---|
| **D001** | error | Tool has no description at all | The model has only a name to go on. Worst case in the ruleset. |
| **D002** | error | Description too short to be useful | `"""Navigate to a URL."""` tells the model what, never when or what it gets back. |
| **D004** | error | **Parameter has no description** | The single most common failure in the ecosystem. Type says `url: string`; the model must invent which URL, in what form, with what constraints — so it hallucinates. |
| **D005** | warn | Placeholder or dev-leftover text | `"TODO"`, `"test tool"` shipped to production. |
| **D006** | warn | Two descriptions nearly identical | The model cannot tell the tools apart, so it picks by coin-flip. |
| **D007** | info | Description doesn't state the return value | Breaks multi-step planning: the model can't tell what it will hold after the call. |
| **I001** *fork* | warn | No server instructions on a server with 5+ tools, or any resources/prompts | Nothing tells the model where to start or how the parts fit; most clients never surface resources or prompts on their own. |
| **I002** *fork* | warn | Instructions over 4,096 characters | Claude Code keeps only the first 4,096; the rest never reaches the model. |

I001/I002 are scored under `desc`. They never fire on a snapshot file that did not
capture instructions (unknown is not the same as none).

**Fix shape.** Tool description answers three questions — what it does, when to use
it, what it returns. Parameter description gives meaning, format, and one concrete
example value.

The root cause is almost always a schema generated from types (pydantic, zod,
OpenAPI) with no descriptions attached. The type system knows the shape; the model
needs the semantics. Your generator is silently stripping the most important signal
your tools carry.

## Naming (`naming`)

| Rule | Sev | What it means | What it costs you |
|---|---|---|---|
| **N001** | warn | Inconsistent convention across tools | Mixing `getUser` and `list_items` makes the catalog read as several APIs. |
| **N002** | error | Names too similar (≤2 chars apart) | Measured cause of live mis-selection: `get_cookies`↔`set_cookie`, `extract`↔`scrape`. |
| **N003** | warn | Generic verb, no object | `execute`, `run`, `query` say nothing about what is operated on. Not applied to the code tool in code-mode, where `execute` is the convention. |
| **N004** | info | Name and description semantically disjoint | The name promises one thing, the text another. |

**Fix shape.** `verb_object`, one convention across the whole catalog, distinguishing
concept inside the name: `get_user_by_id` vs `list_users`, not `get_user` vs `get_users`.

**Renames are breaking.** They invalidate existing configs, saved skills, stored
transcripts, and client code. Never apply one without grepping for and updating every
reference in the same pass.

## Schema design (`schema`)

| Rule | Sev | What it means |
|---|---|---|
| **S001** | error | Tool has no input schema |
| **S002** | error | Parameter declared with no type |
| **S003** | warn | No `required` array declared — the model must guess what is optional |
| **S004** | warn | Overly permissive (bare `object`, no properties, `additionalProperties` wide open) |
| **S005** | warn | Fixed value set described in prose instead of `enum` |
| **S006** | warn | Schema nested too deep |
| **S007** | warn | Too many parameters on one tool |
| **S008** | info | Complex parameter carries no example (not reported where D004 already flags the parameter: one defect, one charge) |

**Fix shape.** Declare `required` explicitly — an empty array is valid and is the
point. Move any fixed value set out of prose into `enum`, where the model cannot
misread it. Flatten deep nesting; a model fills a flat object far more reliably.

See `fix-patterns.md` for the FastMCP `required` trap — on that stack S003 is not
fixable from the function signature.

## Consistency (`consistency`)

| Rule | Sev | What it means |
|---|---|---|
| **C001** | warn | Two tools with heavily overlapping functionality |
| **C002** | info | Mixed natural languages across descriptions |

C001 is an architecture finding, not a text one — the fix is merging or clearly
separating responsibilities, so treat it as breaking.

## Token cost (`tokens`)

The entire catalog is serialized into **every single request**. This is a recurring
tax, not a one-time cost.

| Rule | Sev | What it means |
|---|---|---|
| **T001** | error | Catalog total token cost over budget |
| **T002** | warn | One tool's schema over budget |
| **T003** | warn | Repeated boilerplate across descriptions |
| **T004** | info | Too many tools in one server (suggested max 25) |
| **T005** *fork* | info | 15+ tools and no code or batch tool; replaces T004 when both fire |

**Watch for the inversion:** an undocumented server scores *well* on tokens precisely
because it is undocumented. A 100 here next to a 0 in `desc` is not a strength — it is
the same defect counted twice. Fixing `desc` will and should cost tokens.

T004 is real but not automatically fatal: a well-documented 15-tool catalog can score
A. Size does not force a bad grade; it makes discipline harder to sustain, and every
added tool is another description to write and another name that can collide.

## Programmability (`prog`, code-mode only) — fork

Can a model write a working program or batch against the server on the first try? Read
from the tool's description and its parameters' descriptions; normalised by the number of
code, batch and search tools, not the whole catalog.

| Rule | Sev | What it means |
|---|---|---|
| **P001** | error | Code tool never names its language |
| **P002** | warn | Code tool doesn't say how a result comes back (`return`, last expression, stdout) |
| **P003** | warn | Code tool states no limits (time, memory, output), as text or parameters |
| **P004** | warn | A writing code/batch tool doesn't say what a failure halfway leaves (rollback or partial) |
| **P005** | warn | Neither the code/batch tool nor the instructions point at the API (search tool or doc URI) |
| **P006** | warn | A documentation URI in a description or the instructions matches no listed resource |
| **P007** | warn | Batch tool doesn't state its size limit (`maxItems`) or how to refer to earlier results |
| **P008** | info | A writing code/batch tool takes no idempotency key |
| **P009** | info | A writing code tool has no read-only mode (and no read-only sibling) |

## Exceptions and reach (fork)

A rule that conflicts with a server's stated design is an exception, not a finding:

```json
{ "exceptions": [{ "rule": "S007", "tool": "<tool>", "reason": "one view tool by design" }] }
```

`reason` is required; `tool` is optional (omit for server-wide). Accepted findings are
listed with the reason and not scored; an exception that matches nothing is reported as
stale. Prefer this to `disable`, which hides the same problem in every other tool.

Every report shows the catalog's tokens per request. With `"apiOperations": <n>` declared
it also shows operations reachable per 1k catalog tokens — not scored.

## Probe-only rules (`--probe`)

Only fire under `--probe`, which **calls your tools with deliberately invalid
arguments**. Real side effects on a server that drives a browser, filesystem, or
database — get explicit consent first.

| Rule | Sev | What it means |
|---|---|---|
| **C003** | warn | Error messages non-actionable |
| **C004** | error | Schema declared but not enforced at runtime |

**Fix shape.** An error must name the offending parameter and what was wrong with it,
so the model can self-correct in one turn instead of retrying blindly or giving up.

## What static analysis cannot see

`--eval` runs a live model against the catalog and measures two things no static rule
reaches:

- **Tool-selection accuracy** — ~100% on well-documented catalogs, dropping sharply
  on servers with N002 collisions. The static findings predict where the misses land.
- **Refusal rate** — given a deliberately out-of-scope task, does the model decline?
  Near 100% on small well-documented catalogs, but roughly a coin flip on large fuzzy
  ones. Half the time it "finds" a plausible tool and calls it.

That second number is the one to worry about. An agent that acts when it should do
nothing is the most dangerous failure mode available, and a clean static report does
not rule it out.
