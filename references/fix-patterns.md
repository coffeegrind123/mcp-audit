# Fix patterns per framework

Load this before editing any server source. Confirm which mechanism the target
actually uses (grep an exact description string from a D002 finding) rather than
assuming from the language.

## Writing the text itself

This is the part that carries the score, and it is a writing problem more than an
engineering one. Both templates are short on purpose — verbosity costs tokens on
every request.

**Tool description — three questions, one to three sentences:**

> *what it does* — Load a URL in the active tab and wait for the load event.
> *when to use it* — Call before any element query; element tools fail with no page loaded.
> *what it returns* — Returns the final URL after redirects.

**Parameter description — meaning, format, one example:**

> Absolute URL including scheme; relative paths are rejected. Example: `https://example.com/pricing`

Do not restate the name. `"url": "The url."` clears no rule worth clearing and adds
tokens for nothing. If a parameter has a fixed value set, it belongs in `enum`, not in
the prose (S005).

### Three traps in the text itself, all hit on a real run

- **D005 matches the literal word "placeholder".** Describing an input field's
  placeholder attribute, or "returns a red placeholder image", is flagged as
  dev-leftover text. Reword to "inline hint text" / "a plain red image". The rule
  cannot tell your prose from a `TODO`.
- **N004 wants the tool's name echoed in its description.** A description that opens
  "Load a URL in the active tab" on a tool named `navigate` scores as
  name/description disjoint. Open with the name's own vocabulary — "Navigate the
  active tab to a URL" — then say the rest.
- **Fixing descriptions will cost you token score, and may trip T001.** One 56-tool
  server went from a clean 100 on tokens to ~10.5k per request against an 8k budget.
  That is the correct trade, not a regression: state both numbers rather than quietly
  trimming the descriptions back.

## Python — FastMCP (`mcp.server.fastmcp`)

The dominant Python stack. Schemas are derived from the function signature through
pydantic, and the docstring becomes the tool description.

```python
from typing import Annotated, Literal
from pydantic import Field

async def navigate(
    self,
    url: Annotated[str, Field(description="Absolute URL including scheme; relative paths are rejected. Example: https://example.com/pricing")],
    wait_until: Annotated[Literal["load", "domcontentloaded", "networkidle"], Field(description="Which load event to wait for before returning. Default 'load'.")] = "load",
) -> str:
    """Load a URL in the active tab and wait for the load event.

    Call before any element query — element tools fail with no page loaded.
    Returns the final URL after redirects.
    """
```

Verified against the installed SDK:

- **`Annotated[T, Field(description=...)]` propagates** into `inputSchema.properties.<p>.description`. This is the D004 fix.
- **The docstring becomes the tool description** verbatim, including the extra
  paragraphs. This is the D002/D007 fix.
- **`Literal[...]` emits a JSON Schema `enum`.** This is the S005 fix.
- **Parameters without a default land in `required` automatically.**

### The S003 trap — and the fix that works

When **every** parameter has a default, pydantic omits `required` from the schema
entirely rather than emitting `"required": []`. S003 then fires and **cannot be
cleared by editing the signature.** Giving a parameter a fake non-default to force it
into `required` is worse than the finding — it breaks callers.

The fix is a post-registration pass. `Tool.parameters` is a plain mutable dict, and
`_tool_manager.list_tools()` hands back the live objects, so injecting the empty array
sticks. Run it once, after every tool module has registered:

```python
def _declare_explicit_required(server: FastMCP) -> None:
    """Emit an explicit "required" array on every tool schema."""
    for tool in server._tool_manager.list_tools():
        params = tool.parameters
        if isinstance(params, dict) and params.get("type") == "object" and "required" not in params:
            params["required"] = []

_declare_explicit_required(mcp)
```

Verified: this cleared all 16 S003 findings on a 56-tool server and took Schema design
from 89 to 100, while tools with genuinely mandatory arguments kept their real
`required` list. It touches a private attribute (`_tool_manager`), so leave a docstring
saying why — it looks like dead code to anyone who does not know the trap.

Fall back to disabling only if the SDK version in front of you does not expose that
manager:

```json
{ "disable": ["S003"] }
```

Say so explicitly in the report. A disabled rule you announced is fine; a disabled
rule that silently raised a score is not.

### Registration-time override

`FastMCP.tool()` accepts a `description=` keyword that overrides the docstring. Useful
when the docstring must stay developer-facing, but it splits the text from the
function — prefer the docstring where you can.

## TypeScript — MCP SDK with zod

`.describe()` on each field is the D004 fix; the description field in the registration
call is the D002/D007 fix.

```ts
inputSchema: {
  url: z.string().url().describe("Absolute URL including scheme. Example: https://example.com/pricing"),
  waitUntil: z.enum(["load", "domcontentloaded", "networkidle"])
    .default("load")
    .describe("Which load event to wait for before returning."),
}
```

The registration signature differs across SDK versions (`server.tool(...)` vs
`server.registerTool(...)`, and where `description` sits). **Read the existing
registrations in the repo and match them** rather than importing a remembered shape.

`.optional()` / `.default()` control `required`. **But zod v4 omits `required` entirely
when every field is optional or defaulted** — the same S003 trap as FastMCP. The fix is
`.meta({ required: [] })` on the object, and it has its own trap: `meta()` **replaces** the
computed list, so on an object with a mandatory field it silently drops that field from
`required`. Wrap it so it refuses that case at startup:

```ts
function optionalArgs(shape) {
  const required = Object.keys(shape).filter((k) => !shape[k].safeParse(undefined).success);
  if (required.length) throw new Error(`optionalArgs: ${required.join(", ")} is required; use z.object()`);
  return z.object(shape).meta({ required: [] });
}
```

Verified on the TS SDK v2 (`@modelcontextprotocol/server`): validation is unchanged
(`meta` only affects the JSON Schema).

### Token padding zod adds (T001 / T002)

zod's JSON Schema puts `"$schema"` on every schema and
`"minimum": -9007199254740991, "maximum": 9007199254740991` on every `.int()`. Neither
carries information (2020-12 is MCP's default dialect; zod still enforces the bounds),
and on a mid-size catalog they cost ~900 tokens per request. The SDK accepts any Standard
Schema, so wrap each schema and clean only the advertised JSON:

```ts
const SAFE = Number.MAX_SAFE_INTEGER;
const strip = (n) => Array.isArray(n) ? n.map(strip) : n && typeof n === "object"
  ? Object.fromEntries(Object.entries(n)
      .filter(([k, v]) => k !== "$schema" && !(k === "minimum" && v === -SAFE) && !(k === "maximum" && v === SAFE))
      .map(([k, v]) => [k, strip(v)]))
  : n;
function lean(schema) {
  if (!schema) return schema;
  const std = schema["~standard"];
  return { "~standard": { ...std, jsonSchema: {
    input: (o) => strip(std.jsonSchema.input(o)), output: (o) => strip(std.jsonSchema.output(o)) } } };
}
```

Apply it at registration (`inputSchema: lean(...)`, `outputSchema: lean(...)`). Confirm
with a real `tools/list` through the SDK, not with your own dump: count
`9007199254740991` in the output (should be 0).

## Raw JSON Schema

If the server hand-writes schemas, edit them directly. Every property gets a
`description`; `required` is always present even when empty.

```json
{
  "type": "object",
  "properties": {
    "url": { "type": "string", "description": "Absolute URL including scheme. Example: https://example.com/pricing" }
  },
  "required": ["url"],
  "additionalProperties": false
}
```

`additionalProperties: false` also addresses S004.

## Generated from OpenAPI

Descriptions are lost at generation time, so editing the generated output is thrown
away on the next build. Fix the OpenAPI source (`description` on each parameter and
schema property) and regenerate. If the spec genuinely has none, the generator config
is the place to inject them — not the artifact.

## Client-compatibility fixes (mcp-lint)

Verify the claim for the client first (`mcp-lint.md`). Then, on the **input** schema:

- **`required: false` on a property** — delete it; `required` is an array on the parent.
- **Empty `enum`** — remove the enum or give it values; nothing can satisfy it.
- **Recursive `$ref`** — inline a bounded depth, or take the recursive part as a JSON string
  the tool parses.
- **`anyOf`/`oneOf` from nullable fields** — if a target client really rejects them, make
  the field optional instead of nullable (absent rather than `null`).
- **OpenAI strict tools** (`openai/*`) — every property listed in `required` (optional ones
  become nullable) and `additionalProperties: false`. Only for servers that target strict
  mode; it conflicts with the advice above, so pick per target.
- **`description-exists` on output schemas** — describe output fields the model has to
  interpret (codes, units, ids to pass back); plain echoes can stay bare.

`mcp-lint fix` rewrites a JSON tools file. Use it only on files that are the source of
truth, never on a snapshot of a server whose schemas come from code.

## After editing

- **Python:** `python3 -c "import src.server"` from the server root, or launch it — a
  syntax error in a docstring edit surfaces immediately.
- **TypeScript:** run the build. A stale `dist/` is a top cause of a re-scan showing
  no movement.
- **Clear `__pycache__`** if the re-scan grade is suspiciously unchanged.
- Re-scan with the **same pinned mcpgrade version** as the baseline, or the comparison
  is meaningless.
