# Toolsmith

**An agent that builds its own tools, at runtime, and immediately uses them.**

When the agent hits a task it has no capability for, it doesn't fail. It researches the
API, forges a governed connector, proves it works with a real `200`, and then completes
the original task using a tool that did not exist when you asked.

Built on the [fastn](https://fastn.ai) MCP gateway — the only control plane where
authoring an integration is a *runtime* action rather than a build-time one.

---

## How it works

An MCP server's tool list is fixed for the session, so you cannot hand a running agent a
new tool entry. Toolsmith sidesteps that entirely, because in fastn:

> **Capability is data, not schema.**

`executeAction(connectorId, actionId, input)` is a **generic executor** — the tool's
identity is a runtime *parameter*, not a schema entry. The instant `createAction` returns
an id, that id is callable through the executor the agent already had. No reconnect, no
client restart, no tool-list refresh.

Here is that happening, verbatim, from the proving run:

```
createConnector  → e5b4d2aa…      capability does not exist yet
createAction     → bacf5014…      now it does
storeConnections → ACTIVE
executeAction(e5b4d2aa…, bacf5014…, {…}) → 200 in 132ms, real data
```

Four calls. The last two used ids that did not exist when the turn started.

It holds at three layers:

| Layer | Mechanism | Why it matters |
|---|---|---|
| MCP | `executeAction` takes ids as parameters | new tools callable with zero client churn |
| Workflow sandbox | `fastn.connector.<slug>.<action>()` is a dynamic proxy resolved at execution time | a workflow can call a connector forged seconds earlier |
| Registry | the connector persists org-wide | the capability outlives the session and is **shared** |

That last row is the point. The agent isn't writing throwaway code in `/tmp` — it is
permanently expanding the organisation's capability surface.

## Why this isn't codegen

1. **Governed.** The tool lands in the org's connector registry with real auth handling,
   rate limiting, observability, and audit.
2. **Persistent and shared.** The next agent — and every other user in the org — now has
   that tool. The capability surface only grows.
3. **Verified, not hallucinated.** The agent is *required* to prove a live `200` via
   `executeAction` before it may use its own tool.

Point 3 is the credibility answer to "how do you know it actually works?"

**The framing: agent capability compound interest.** Every gap becomes a permanent org
capability.

## The forge loop

```
task ──▶ listConnectors ──▶ gap confirmed
              │
              ▼
         searchWeb / fetchWebPage        find docs or an OpenAPI spec
              │
              ▼
         createConnector
         createAction × 1..3             only the endpoints the task needs
         storeConnections
              │
              ▼
         executeAction ──▶ real 200?     ── no ──▶ diagnose, patch, retry (max 3)
              │ yes
              ▼
         call the new tool ──▶ finish the original task
```

## Proven

Verified end to end on 2026-09-19 against the USGS earthquake catalog — a genuine gap
(zero hits for `earthquake` / `usgs` / `seismic` across 424 registered connectors),
NO_AUTH, and dramatic output.

Task: *"Were there any significant earthquakes this month?"* → no connector exists →
forge one → `M 6.7 — 38 km NE of Tambo, Peru`.

Full call-by-call reproduction, including ids, in [AGENTS.md](./AGENTS.md).

## Status

- [x] Authorize Fastn Workspace
- [x] Baseline connector inventory (424 registered; genuine gaps identified)
- [x] Forge a connector by hand end to end, recording every failure
- [ ] Wrap the loop as an agent that does it unattended
- [ ] Pre-vet the tier-2 shortlist
- [ ] Capability counter + demo UI
- [ ] Rehearse twice end to end

## Docs

- [AGENTS.md](./AGENTS.md) — architecture, forge loop, verified reference calls, all
  ten gotchas, demo tiers
- [CLAUDE.md](./CLAUDE.md) — Claude Code specifics
