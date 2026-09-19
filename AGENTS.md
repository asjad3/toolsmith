# AGENTS.md — Toolsmith

Instructions for any coding agent working in this repo.

## What this project is

**An agent that builds its own tools at runtime and immediately uses them.**

When the agent hits a task it has no capability for, it does not fail. It researches the
API, forges a governed fastn connector, proves it works with a real `200`, then completes
the original task using a tool that did not exist when the user asked.

Built on the fastn MCP gateway.

## The core insight — read this before changing anything

**Capability is data, not schema.**

An MCP server's tool list is fixed for the session, so fastn never adds new MCP tool
entries. Instead `executeAction(connectorId, actionId, input)` is a *generic executor* —
the tool's identity is a runtime **parameter**. The instant `createAction` returns an id,
that id is callable through the executor already in the tool list. No reconnect, no
client restart.

This holds at three layers:

| Layer | Mechanism |
|---|---|
| MCP | `executeAction` takes `connectorId` / `actionId` as params |
| Workflow sandbox | `fastn.connector.<slug>.<action>()` — dynamic proxy, resolved at execution time |
| Registry | the connector persists org-wide, so the capability outlives the session |

Do not try to "register" new tools with the MCP client. That is not how this works and
that path does not exist.

## The forge loop

```
task ──▶ listConnectors ──▶ gap confirmed?  ── no ──▶ just use the existing connector
              │ yes
              ▼
         searchWeb / fetchWebPage          find docs or an OpenAPI spec
              │
              ▼
         createConnector                   REST + auth method
         createAction × 1..3               ONLY the endpoints the task needs
         storeConnections                  mint a connection (required — see G2)
              │
              ▼
         executeAction ──▶ real 200?       ── no ──▶ diagnose, patch, retry (max 3)
              │ yes
              ▼
         call the new tool ──▶ finish the original task
```

**The verification step is not optional.** The agent may not use a tool it has not proven
with a live `200`. This is the entire credibility story — "verified, not hallucinated."

## Verified reference implementation

Proven end to end on 2026-09-19 against the USGS earthquake catalog (NO_AUTH, a genuine
gap — 0 hits for earthquake/usgs/seismic across 424 connectors).

```
createConnector  { name: "USGS Earthquake Catalog", slug: "usgsEarthquake",
                   protocol: "REST", domain: "earthquake.usgs.gov",
                   authMethods: [{ type: "NO_AUTH", title: "No authentication",
                                   isDefault: true }] }
  → connectorId  e5b4d2aa-1298-4387-bba4-7cd986f7e188
  → authMethodId 27907c91-5f93-4471-9239-5e103beb9089

createAction     { type: "HTTP", slug: "queryEarthquakes",
                   httpConfig: { url: "https://earthquake.usgs.gov",
                                 path: "/fdsnws/event/1/query", method: "GET" },
                   inputContract: { ...becomes query params... } }
  → actionId     bacf5014-7ec2-45be-b384-b148986019a6

storeConnections { authType: "NO_AUTH", name: "default",
                   credentials: { noop: "public-api-no-auth-required" } }
  → ACTIVE, verifyStatus "skipped"

executeAction    { connectorId, actionId, connectionName: "default",
                   input: { format: "geojson", limit: 3, minmagnitude: 6,
                            starttime: "2026-08-20", orderby: "magnitude" } }
  → success: true, status: 200, durationMs: 132
```

## Gotchas

### Found empirically — NOT in the fastn skill docs

- **G1. `executeAction` requires `connectionName` explicitly.** Its schema claims it
  "defaults to `'default'`". It does not — omitting it returns
  `400 connectionName is required`. Always pass it.
- **G2. NO_AUTH connectors still need a connection, and credentials must be non-empty.**
  `credentials: {}` → `400 cred carries no usable material (empty fields and no
  oauth.refreshToken)`. Pass a dummy key. So even the auth-free path has an auth-shaped
  step the forge loop must handle — do not assume NO_AUTH means zero setup.
- **G3. Cosmetic:** a connector whose only auth method is NO_AUTH still reports
  `authType: "api_key"`. Ignore it; trust the `authMethods` array.

### From the `connector_builder` skill — each one is a demo killer

- **G4. Multi-field credentials must use `INPUT` auth type.** `API_KEY` renders exactly
  one fixed field and *silently drops* extras (instance URL, account id, store hash).
  `BASIC` drops everything but username/password.
- **G5. Never name a credential field `token`.** fastn auto-injects
  `Authorization: Bearer {token}` based on the *field name*, overriding any per-action
  `Authorization` header → 401. Use `apiKey` / `accessToken`.
- **G6. Every `create_actions_batch` call rotates the auth method** and orphans existing
  connections ("Connection not found" while the UI still shows it). **Build every action
  BEFORE connecting.** If the demo connects and then forges one more action, the
  connection dies on stage.
- **G7. Validate one action before mass-building.** One simple GET confirms auth
  injection and the response envelope before you build on a bad assumption.
- **G8. Responses >~4KB truncate** with a misleading `Unterminated string in JSON`.
  *The execution still succeeded.* Confirm via `getAction(id).lastTestEvidence`, never by
  retrying — critical for writes.
- **G9. URL params are double-brace.** `{{input.x}}` substitutes; `{x}` is a literal. In
  URL *paths* use `{{auth.accessToken}}` — `{{auth.token}}` renders empty there (it works
  fine in headers).
- **G10. GET query params go in `inputContract` only.** fastn auto-appends them. Do not
  also set `httpConfig.queryParams` — optional empty ones break the request.

## Conventions

- Connector slugs: `^[a-z][a-zA-Z0-9]*$` (camelCase, lowercase first char).
- Label every forged connector `toolsmith-forged` so it can be found and cleaned up.
- `stage: "test"` and `visibility: "private"` while developing.
- Keep `limit` small on probe calls — G8 truncation starts around 4KB.
- Build only the endpoints the task needs. Forging 100 actions is not the demo; forging
  the *right one* in 60 seconds is.

## Demo tiers

Auth is where every demo killer lives (G2, G4, G5, G6), so the guaranteed path is
auth-free.

| Tier | What | Risk |
|---|---|---|
| 1 | NO_AUTH public API, forged live in ~60s (USGS is proven) | low — promise this one |
| 2 | A judge names an API from a pre-vetted shortlist | medium — electric if it lands |
| 3 | API-key API showing the connect-link flow | high — only if time holds |

Confirmed genuine gaps across the 424-connector registry: `meteo`, `usgs`/`earthquake`,
`openlibrary`, `nasa`, `pollen`, `tide`, `carbon`. FX is **already covered 4×** — do not
use it as a gap example.

## Environment

- fastn org: `personal_8efbb0503fdbe7cb848e` (personal), env `test`, role `owner`.
- The fastn MCP gateway requires reading its `gateway` skill before any platform call.
  Installed skills: `gateway`, `integration_builder`, `connector_builder` in
  `~/.claude/skills/`.
