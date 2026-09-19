# CLAUDE.md

**Read [AGENTS.md](./AGENTS.md) first.** It holds the architecture, the forge loop, the
verified reference implementation, and all ten gotchas. This file only adds Claude
Code specifics.

## Before any fastn call

The fastn MCP gateway refuses every platform tool until this session has read its
operating manual, and refuses your first call again if no task skill is loaded:

```
mcp__fastn__skill { "slug": "gateway" }
```

Installed locally in `~/.claude/skills/`: `gateway`, `integration_builder`,
`connector_builder`. Read `connector_builder` before touching a connector, action, auth
method, or webhook — it encodes gotchas G4–G10 in AGENTS.md and will save you hours.

Version-check them at the start of a session; reinstall if the published version moved:

```
mcp__fastn__skill { "slugs": ["gateway", "connector_builder", "integration_builder"] }
```

## Tool loading

Most fastn tools are deferred — only their names are in context, not their schemas.
Load before calling:

```
ToolSearch { "query": "select:mcp__fastn__fastnPlatform__createConnector,mcp__fastn__fastnPlatform__createAction" }
```

Calling a deferred tool without loading its schema fails with `InputValidationError`.

## Large tool results

`listConnectors` with no filter returns ~368KB and gets spilled to a file rather than
returned inline. Don't retry it — query the saved file with `jq`:

```bash
jq -r '.data[] | select(.slug | test("meteo"; "i")) | "\(.slug)\t\(.name)"' <saved-file>
```

Prefer the `search` parameter when you know what you're looking for.

## Gotchas that bite in this harness specifically

- **Bash heredocs get blocked by the auto-mode classifier.** Writing a multi-line script
  via `python3 - <<'PY'` may be denied. Use the `Write` tool for files, or a single-line
  command.
- **`cd` in a Bash call changes the session's working directory** for subsequent calls.
  Prefer absolute paths.

## Don't

- Don't try to register forged tools with the MCP client. Capability is data, not
  schema — see the core insight in AGENTS.md. That path does not exist.
- Don't let the agent use a tool it hasn't proven with a live `200`. The verification
  step *is* the product.
- Don't forge against an API that already has a connector. 424 exist; check first.
