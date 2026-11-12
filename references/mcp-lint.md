# mcp-lint: client-compatibility checks

Load this when running or reporting mcp-lint. mcpgrade asks "can a model use these tools
well?"; mcp-lint asks "will this schema load and behave in client X?" — Claude, Cursor,
Gemini, VS Code, Windsurf, Cline, the OpenAI Agents SDK, Continue. Different axis, so run
both. Pinned: **`mcp-lint@1.0.4`** (MIT). Re-read its README rules section on any bump.

## Running it safely

mcp-lint looks for `.mcplintrc.json` in the current directory and every parent up to the
home directory, and **imports whatever that file lists under `plugins` or `extends`**.
Run inside an untrusted repo, that executes the repo's code. Therefore, always:

```bash
cd <scratchpad>
echo '{}' > clean-mcplintrc.json
jq '.tools' snapshot.json > tools.json          # mcp-lint reads a bare tools array
npx -y mcp-lint@1.0.4 check tools.json --config clean-mcplintrc.json --format json > lint.json
```

- Lint the snapshot mcpgrade saved (`--save-snapshot`): same catalog, one connection,
  and it covers authenticated servers (mcp-lint cannot send auth headers).
- Never `check --installed` or `--remote` without asking: they launch or connect to every
  server in the user's client configs.
- Never `recover --sandbox --invoke` without consent: it **calls tools**.
- Never `fix` on a source file. `fix` rewrites JSON tool files; source fixes go through
  the server's own schema code (see `fix-patterns.md`).
- Exit code 1 means "at least one error", not a crash. Read `lint.json`.

## Output

A flat JSON array, one object per finding:

```
[{ ruleId, severity: "error"|"warning"|"info", message, toolName, path, clients: [...] }]
```

`ruleId` is either universal (`valid-json-schema-subset`, `description-exists`,
`max-depth`, `no-required-false`, `no-empty-enum`, `no-recursive-refs`,
`no-content-encoding`, `no-unsupported-formats`, `tool-name`, `destructive-hint`, ...) or
client-prefixed (`claude/...`, `cursor/...`, `gemini/...`, `openai/...`, ...). `path`
starts with `inputSchema`, `outputSchema` or `description`.

## Triage — never report the raw count

A well-built server can produce hundreds of findings. Tally by
`(ruleId, severity, path root)` and work in this order:

1. **Universal errors on `inputSchema`** — the schema the model fills. Real defects.
2. **Client-prefixed findings on `inputSchema` for clients the user targets.** Ask which
   clients matter; `openai/*` strict-mode findings only matter for OpenAI strict tools.
3. **`outputSchema` findings** — clients validate results against it but do not show it
   to the model. Lower priority; often generator artefacts.
4. **Client warnings for clients nobody targets** — list, do not fix.

```bash
python3 -c "
import json,collections
d=json.load(open('lint.json'))
for (r,s,io),n in sorted(collections.Counter((x['ruleId'],x['severity'],x.get('path','').split('.')[0]) for x in d).items()):
    print(f'{r:34} {s:8} {io:13} {n}')
"
```

**Its compatibility claims are claims.** Before calling a finding a defect, check the
client's own documentation or reproduce it. Example: `valid-json-schema-subset` reports
every `anyOf` as an error for all clients, while zod emits `anyOf` for ordinary nullable
fields; a blanket "unsupported" needs a source.

## Overlap with mcpgrade

Report one finding per defect:

| mcp-lint | mcpgrade | Note |
|---|---|---|
| `description-exists` | D001 / D004 | mcp-lint also checks outputSchema properties |
| `max-depth` (> 5) | S006 (> 3) | different thresholds; quote mcpgrade's |
| `tool-name` | N001–N003 | |
| `destructive-hint` | — | annotations; mcpgrade does not check them |
