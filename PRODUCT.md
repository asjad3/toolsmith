# Toolsmith — product definition

**Shape: the embedded long-tail integration layer for SaaS AI assistants.**

## The problem

A SaaS company ships an AI assistant. Their customers ask it to connect to systems the
SaaS has never heard of — a regional CRM, an in-house ERP, a warehouse tool with 400
total users. The vendor has 30 connectors. Their customers collectively use thousands.

The tail is effectively infinite, and every request is a sprint. So the answer is always
*"not supported — it's on the roadmap,"* and the AI feature never gets adopted by the
accounts that asked.

Nobody can hand-build 4,000 connectors. That is the unsolvable part, and it is what
makes this worth building.

## Two personas, and they must never see the same thing

| | Vendor (the buyer) | Their customer (the user) |
|---|---|---|
| Wants | control, visibility, a roadmap signal, liability containment | "connect my thing and let the assistant use it" |
| Sees | gap ledger, review queue, promotion ladder, provenance | a connect screen and a working assistant |
| Must never see | — | connector internals, slugs, contracts, the word "connector" |

The fastn `whoami` call distinguishes these directly (`actingAsTenant: true` +
`actor: embed_user` = the customer inside the vendor's product). Branch the entire
experience on it. Platform vocabulary leaking into the customer-facing surface is a
product bug, not a cosmetic one.

## The promotion ladder

This is the core mechanic, and it does two jobs at once.

```
tenant-private        forged for one customer, on their request, their credentials
      │               vendor has not vouched for it
      │  a 2nd tenant needs the same API
      ▼
   candidate          surfaced to the vendor: "2 customers need this. Promote?"
      │
      │  vendor reviews the contract + the proving 200
      ▼
 shared catalog       vendor-owned, maintained, available to every tenant instantly
```

**Job 1 — compounding.** Every forge makes the product better for the next customer. The
second tenant to need an API gets it in zero seconds. The vendor's catalog grows from
real demand instead of guesses, and it grows without the vendor writing code.

**Job 2 — liability firewall.** A tenant-private forged tool is that tenant's own risk,
built on their request with their credentials. A promoted tool is one the vendor
explicitly vouches for. Without this boundary the vendor is shipping connectors they
didn't write to customers who'll blame them — which is the fastest way to lose the deal.

The ladder is why the network effect and the safety story are the same feature.

## The gap ledger is the wedge

Log every capability miss: which tenant, which API, which task, how often.

That is a prioritized integration backlog derived from actual demand. Aggregated across
tenants it becomes a roadmap instrument the vendor cannot get anywhere else:

> *"14 of your customers have asked for Zendesk. 9 for NetSuite. 3 for a Dutch payroll
> API you've never heard of."*

**Ship this first, with zero autonomy.** It is pure observability, carries no risk, is
trivially believable, and it creates the demand signal that justifies forging. Then
forging is the expansion: *"want us to close the top one automatically?"*

Selling observability first is what earns the right to ask a vendor to let an agent
create tools in their product.

## Safety defaults

| Tool shape | Default |
|---|---|
| Read-only (GET) | auto-forge, auto-promote to tenant-private |
| Anything that writes | forge to `test`, human approval required |
| Metered / paid API | approval + spend cap regardless |
| Promotion to shared catalog | always vendor approval |

Read-only auto-forge covers most of what agents get blocked on, so the magic survives
where it's safe and the gate only appears where it isn't.

## The hard constraint: auth bounds the forgeable tail

This determines whether the product works, so state it honestly rather than discovering
it later.

- **Key-based auth (API key, bearer, basic, INPUT) — forgeable.** The customer supplies
  their own credential through a fastn connect link. The vendor never touches it and
  never pre-registers anything. **This is the addressable tail.**
- **OAuth — not forgeable.** OAuth requires registering a client application with each
  provider, which is manual, per-provider vendor work. An agent cannot do it.

So the product claim is *"any API your customer can hand us a key for"* — not "any API."
That is still a very large tail (most long-tail B2B software is key-auth), but
over-claiming here collapses on the first OAuth request.

Mitigation for the OAuth head: those are the high-demand providers anyway, so they're
exactly what the gap ledger should surface for the vendor to build properly once.

## Registry hygiene — invisible in a demo, fatal in production

1. **Dedup before forging.** Evidence from our own org: FX is already covered **4×**
   across 424 connectors. Let every tenant forge freely and you get 40 near-duplicate
   Slack connectors. Requires semantic matching against the existing registry *before*
   `createConnector`.
2. **Lifecycle.** Third-party APIs drift. Forged tools need health checks, break alerts,
   and re-forge on schema change. Silent rot is worse than a missing tool, because the
   assistant confidently returns stale or wrong answers.
3. **Provenance.** Which docs URL, which agent, which task, which tenant, who approved.
   Required for audit, and for debugging a bad inferred contract six weeks later.

## The demo this implies

Drop "name any API" — it proves autonomy, which isn't the product. Prove **compounding**
instead:

1. **Tenant A (Acme)** asks the assistant something needing an uncovered API. Forged
   live, tenant-private, answers the question.
2. **Tenant B (Globex)** asks for the same thing. **It's already there.** Zero forge
   time, instant answer. ← this is the moment
3. **Vendor dashboard:** *"2 customers needed this. Promote to your catalog?"* One click,
   now every tenant has it.
4. **Punchline:** Tenant A asks for something that *writes*. The agent forges the tool,
   then stops itself and emits an approval request instead of firing it.

Step 2 tells the whole product story. Step 4 is what makes a judge who has shipped
something trust the rest.

## Build order

- [ ] **Phase 0 — prove multi-tenant forging.** Create a second tenant; forge under A;
      verify B cannot see it; promote; verify B can. Everything depends on this working.
- [ ] **Phase 1 — gap ledger** aggregated across tenants (`fastn.db`).
- [ ] **Phase 2 — promotion ladder** + vendor review queue with contract-next-to-200.
- [ ] **Phase 3 — customer-facing connect surface**, zero platform vocabulary.

Hackathon scope: Phase 0 + the two-tenant demo + a minimal vendor dashboard showing the
promotion prompt. That is the product story, not a capability showcase.

## What changes in the architecture

The dev prototype is single-tenant (Path A). The product is **Path B** throughout:

- `set_connector_scope` → `MULTI_TENANT` for per-customer connectors
- config **templates** owned by the vendor, **clones** minted per tenant
- workflows read `fastn.config.getByTemplate(templateId)`, never `.get(configId)` —
  `.get` always returns the vendor's template, so every customer would silently read the
  vendor's mapping instead of their own
- credentials only ever via connect/setup links, never collected in conversation

See `references/multi-tenancy.md` in the installed `integration_builder` skill before
touching the manifest, installations, or widgets.
