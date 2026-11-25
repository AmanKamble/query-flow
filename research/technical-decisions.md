# Technical Decisions

> ADR-style record. Each decision states the context, the choice, and what we accept by
> choosing it. Revisit triggers are listed so these do not become permanent by accident.

## Summary table

| # | Decision | Choice | Status |
|---|---|---|---|
| 1 | AI's role | Spec generator only; no DB access, no tools | Accepted |
| 2 | AI interface | One endpoint, `POST /api/v1/assistant/query` | Accepted |
| 3 | Output format | JSON Schema-constrained `ReportSpec` + Zod | Accepted |
| 4 | Model | Qwen3.6/3.5-35B-A3B via local Ollama | Accepted |
| 5 | Hosted fallback | Groq free tier behind the same adapter | Accepted |
| 6 | App architecture | Single Next.js full-stack app | Accepted |
| 7 | Database | MongoDB, shared collections, `tenantId` | Accepted |
| 8 | Multi-tenancy | Session-derived `orgId`, injected at execution | Accepted |
| 9 | Module metadata | Code-first TypeScript registry | Accepted |
| 10 | Report execution | Deterministic compiler, no string queries | Accepted |
| 11 | Cross-module | Declared relations, ≤ 2 tenant-filtered `$lookup` | Accepted |
| 12 | Conversation state | Full spec per turn, last 6 turns, no rows stored | Accepted |
| 13 | Auth | Better Auth + organization plugin | Accepted |
| 14 | Validation | Zod 4 at every boundary | Accepted |
| 15 | Exports | ExcelJS / csv-stringify / PDFKit, streaming | Accepted |
| 16 | Package manager | Yarn 4 via Corepack | Accepted |
| 17 | Deployment | Next.js Node host + Atlas; Ollama on a GPU box | Accepted |
| 18 | Background jobs | None in v1; add a queue only for large exports | Accepted |

---

## ADR-1: AI generates specs, never queries

**Context.** The obvious shortcut is "let the model write MongoDB queries" or "give it
tools to inspect the database". Both are fast to build and impossible to secure: the
model becomes an untrusted principal with database credentials.

**Decision.** The AI returns a structured `ReportSpec`. A deterministic compiler turns
that spec into a query. The AI process has no driver, no connection string, no tools.

**Consequences.**

- ✅ Prompt injection cannot reach data; the worst outcome is a rejected spec.
- ✅ Every query is reviewable, testable, and indexable.
- ✅ Results are correct by construction (metric semantics live in code).
- ❌ The model cannot answer questions outside the registry. That is the product
  boundary, and it is a feature.
- ❌ New report shapes require a registry/compiler change, not just a prompt tweak.

**Revisit if:** never for this product. A separate, explicitly designed "summarize these
aggregates" path may exist later, with aggregation-only inputs.

---

## ADR-2: One AI endpoint

**Context.** Multiple AI routes (`/chat`, `/insights`, `/suggest`) multiply the security
surface and drift in validation.

**Decision.** `POST /api/v1/assistant/query` is the only route that calls a model.
Everything else accepts an already-validated spec.

**Consequences.**

- ✅ One place to rate limit, validate, log, and audit.
- ✅ The UI can edit a spec and re-run it without the model (cheap, fast).
- ❌ Feature requests for "AI everywhere" need a design conversation, not a new route.

---

## ADR-3: Constrained JSON + server-side Zod

**Context.** Prompt-only "return JSON" fails a few percent of the time; every failure is
a user-visible error.

**Decision.** Use the provider's JSON-Schema mode (Ollama `format`, Groq `strict`) **and**
validate with Zod on the server. Both, not either.

**Consequences.**

- ✅ Near-100% parseability; failures are semantic, not syntactic.
- ✅ Constrained decoding guarantees shape, not meaning — Zod and semantic checks still
  catch forbidden fields and bad operators.
- ❌ Two schema definitions (JSON Schema for the model, Zod for the server) must stay in
  sync. Mitigation: generate the JSON Schema from Zod (`z.toJSONSchema()`) where the
  provider accepts it.
- ❌ Some providers do not support strict mode; the adapter must degrade gracefully.

---

## ADR-4: Local Qwen as the primary model

**Context.** ERP data is sensitive; per-token pricing adds up; free hosted tiers have
rate limits and data-processing terms.

**Decision.** Qwen3.6-35B-A3B (fallback Qwen3.5-35B-A3B) on local Ollama, Apache 2.0,
`temperature: 0`, fixed seed, 8K context.

**Consequences.**

- ✅ No data leaves the network; no marginal cost; no rate limits.
- ✅ Strong JSON adherence and multilingual input.
- ❌ Requires a 32 GB machine (or 24 GB VRAM) for the 35B MoE; 16 GB machines fall back
  to Gemma 4 12B.
- ❌ You own uptime, model updates, and GPU capacity.
- ❌ Throughput is hardware-bound; concurrency needs queueing.

**Revisit if:** accuracy on the golden set is below target and a hosted model clearly
wins, or volume outgrows a single box.

---

## ADR-5: Groq free tier as fallback, not primary

**Context.** Demos and development machines may not have a capable GPU.

**Decision.** Implement a `GroqProvider` behind the same interface; enable it via
`AI_PROVIDER`/`GROQ_API_KEY`. Only catalog metadata and the question are sent.

**Consequences.**

- ✅ Fast, free for development, zero infra.
- ✅ Proves the adapter abstraction is real.
- ❌ 30 RPM / 1,000 RPD / 8K TPM / 200K TPD — fine for dev, not for a busy team.
- ❌ Data leaves the network. Acceptable for synthetic/demo data, **not** for production
  tenant data without a signed agreement.
- ❌ Free-tier availability and model lineup can change without notice.

**Revisit if:** a paid hosted tier becomes cheaper than maintaining a GPU box, or a
customer requires cloud inference.

---

## ADR-6: Single Next.js app, not client/server split

**Context.** The scaffold has empty `client/` and `server/` folders. A separate Express
API is the "MERN" default.

**Decision.** One Next.js App Router app. Route Handlers are the API. `src/server/*`
holds backend code; it is never imported by Client Components.

**Consequences.**

- ✅ One deploy, one auth context, shared TypeScript types and Zod schemas.
- ✅ Server Components reduce client JS; streaming responses work for exports.
- ✅ Fewer moving parts for a small team.
- ❌ Long-running work (large exports, batch jobs) does not fit request/response well;
  needs a worker later.
- ❌ Next.js upgrades are app-wide.
- ❌ A separate GPU service may still be needed for Ollama (different host, same API
  contract).

**Revisit if:** the backend needs independent scaling, or a second client (mobile)
appears. The API contract is already versioned (`/api/v1`), so splitting later is
mechanical.

---

## ADR-7: MongoDB with shared collections

**Context.** ERP reporting needs flexible documents and aggregation; the team knows
MERN.

**Decision.** MongoDB 7/8, one database, shared collections, `tenantId` on every
tenant-owned document.

**Consequences.**

- ✅ Simple operations; one index strategy; cheap.
- ✅ Aggregation pipeline covers grouping, date bucketing, and joins.
- ❌ No enforced schema at the DB level — the registry and Zod carry that weight.
- ❌ `$lookup` is less efficient than SQL joins; mitigated by denormalized fields.
- ❌ Cross-tenant leakage is one missing filter away; mitigated by four enforcement
  layers and tests.

**Revisit if:** a tenant requires physical isolation (then database-per-tenant), or
reporting outgrows aggregation (then a columnar store / read replica).

---

## ADR-8: Tenant identity from the session only

**Context.** Accepting `tenantId` from the request is the classic multi-tenant bug.

**Decision.** `orgId` comes from the verified session membership. The executor injects
it into every `$match` and every `$lookup`. A spec containing `tenantId` is rejected.

**Consequences.**

- ✅ No request shape can select another tenant.
- ✅ One code path for UI, API, exports, and saved reports.
- ❌ Users in multiple orgs must switch org explicitly (correct behaviour).
- ❌ Background jobs must carry an explicit, trusted `orgId` from the job record.

---

## ADR-9: Code-first module registry

**Context.** Options: introspect Mongo, store metadata in the DB, or define modules in
TypeScript.

**Decision.** TypeScript `ModuleDef` objects, validated at boot, serialized into the
prompt at runtime.

**Consequences.**

- ✅ Type safety, code review, tests, versioning with the code.
- ✅ No admin UI to secure; no metadata drift.
- ❌ Adding a module is a deploy, not a config change.
- ❌ Non-developers cannot extend the catalog.

**Revisit if:** business users need self-service modules. Then add a metadata layer
**behind** the same validation and review, not instead of it.

---

## ADR-10: Deterministic compiler, no string queries

**Context.** Building query strings from model output is the fastest route to injection
and unindexable queries.

**Decision.** A typed compiler maps allow-listed operators to MongoDB query documents.
No `eval`, no string concatenation, no `$where`, no user-supplied `$expr`.

**Consequences.**

- ✅ Injection-resistant by construction; regex is escaped.
- ✅ Pipelines are snapshot-testable and index-checkable.
- ❌ New operators require code. That is intentional.
- ❌ Some exotic questions cannot be expressed; they return `unsupported`.

---

## ADR-11: Declared relations, limited joins

**Context.** "Revenue by customer segment" needs data from two modules.

**Decision.** Relations are declared in the registry. Max 2 `$lookup` stages, each with
the tenant filter re-applied inside the sub-pipeline.

**Consequences.**

- ✅ Cross-module reports work without exposing the schema graph to the model.
- ✅ Tenant isolation holds through joins.
- ❌ Deeply relational questions are out of scope for v1.
- ❌ `$lookup` cost grows with collection size; denormalization is the escape hatch.

---

## ADR-12: Conversation = full spec + short history

**Context.** Multi-turn refinement ("now by region") needs context; storing everything
bloats prompts and leaks data.

**Decision.** Each turn returns a complete spec. Keep the last 6 turns and the last
validated spec. Never store or send result rows.

**Consequences.**

- ✅ Stateless model calls; easy to validate; easy to debug.
- ✅ Small prompts; no PII in conversation history.
- ❌ The model may drop a constraint between turns. Mitigation: show the resolved spec
  in the UI so the user sees exactly what will run.
- ❌ No "remember last month's report" beyond the stored spec — use saved reports.

---

## ADR-13: Better Auth with organizations

**Context.** Auth is easy to get subtly wrong; multi-tenant roles are table stakes.

**Decision.** Better Auth + `organization` plugin for users, sessions, orgs, members,
roles, and invitations.

**Consequences.**

- ✅ Battle-tested session handling, CSRF, email verification, invitations.
- ✅ Role model maps to the registry's capability matrix.
- ❌ Another dependency and its migration surface.
- ❌ Custom flows (SSO, SCIM) may need paid/enterprise features — out of scope for v1.

**Revisit if:** enterprise SSO becomes a requirement.

---

## ADR-14: Zod at every boundary

**Context.** Three untrusted inputs: HTTP bodies, model output, and stored specs.

**Decision.** Zod schemas in `src/shared/schemas`, reused by route handlers, the AI
validator, and tests. `.strict()` everywhere.

**Consequences.**

- ✅ One source of truth for shapes; errors are typed and testable.
- ✅ Can generate JSON Schema for the model from the same definition.
- ❌ Zod 4 migration churn; pin the major version.
- ❌ Validation is not authorization — both must run.

---

## ADR-15: Streaming exports with established libraries

**Context.** Hand-rolled XLSX/CSV/PDF is a maintenance trap; Puppeteer is heavy.

**Decision.** ExcelJS (streaming writer), csv-stringify, PDFKit. Synchronous streaming
up to 50k rows; async jobs beyond that.

**Consequences.**

- ✅ Correct escaping, formats, and encoding without custom binary code.
- ✅ No Chromium in the deploy image.
- ❌ PDFKit has no table primitive; a small internal helper is needed.
- ❌ ExcelJS streaming has a narrower API than the in-memory writer.

---

## ADR-16: Yarn 4 via Corepack

**Context.** The project standard is Yarn; reproducibility matters.

**Decision.** Yarn 4 (Berry) with `nodeLinker: node-modules`, `yarn.lock` committed,
`yarn install --immutable` in CI.

**Consequences.**

- ✅ Deterministic installs; no global Yarn version drift.
- ✅ `node-modules` linker keeps tooling (Next.js, native modules) happy.
- ❌ Plug'n'Play would be stricter but breaks some tools; not worth it here.
- ❌ Contributors must run `corepack enable` once.

---

## ADR-17: Deploy app and model separately

**Context.** Ollama wants a GPU; Next.js wants a cheap Node host.

**Decision.** Next.js on Vercel/Render/Railway/Fly; MongoDB Atlas; Ollama on a GPU box
(or Groq) reachable over a private network.

**Consequences.**

- ✅ Each tier scales and is priced independently.
- ✅ The app degrades gracefully to Groq when the GPU box is down.
- ❌ Two things to operate; network latency between them.
- ❌ Ollama must not be exposed publicly; put it behind auth or a private network.

---

## ADR-18: No background queue in v1

**Context.** Queues (BullMQ/Redis) are the usual reflex for exports and AI calls.

**Decision.** Do the work in the request, stream the response, cap the size. Add
`export_jobs` + a worker only when exports exceed ~50k rows or ~15 s.

**Consequences.**

- ✅ One fewer service, one fewer failure mode.
- ✅ Simpler local development.
- ❌ A 100k-row export is rejected rather than queued.
- ❌ No retry semantics for long AI calls; the user retries manually.

**Revisit if:** exports regularly hit the cap, or AI calls need retry/backoff across
minutes.

---

## Accepted trade-offs (explicitly)

1. **Less flexibility for more safety.** The AI cannot answer anything outside the
   registry. We accept that to eliminate an entire class of data-leak and injection
   risk.
2. **Deploys for schema changes.** Code-first registry means a deploy to add a field.
   We accept slower iteration for type safety and review.
3. **Denormalized names.** `customerName` on orders speeds reports but needs a backfill
   on rename. We accept the maintenance job.
4. **No result caching by default.** Stale financial numbers are worse than a 100 ms
   query. We cache specs, not rows.
5. **Single app.** Simpler now; a split later is mechanical because the API is
   versioned.
6. **English-first UI.** The model handles multilingual input, but labels and formats
   are English in v1.

## Open questions

| # | Question | Why it matters | How to resolve |
|---|---|---|---|
| Q1 | Which ERP modules are in the demo? | Drives registry scope and seed data | Confirm with stakeholders; assume Sales, Inventory, Purchases, Customers |
| Q2 | Is data already in MongoDB, or must it be imported? | Import is often 20–30% of the work | Inspect source system early |
| Q3 | What is the target hardware for the demo? | Determines 35B vs 12B model | Ask; prepare both Ollama profiles |
| Q4 | Do users need scheduled/emailed reports? | Adds a job runner and email infra | Defer to v2 unless required for the demo |
| Q5 | Is row-level security needed (e.g. sales rep sees only own orders)? | Changes the compiler and registry | Add an optional `rowFilter` per role if yes |
| Q6 | What is the retention policy for conversations? | Compliance and storage | Default 90 days; confirm with legal |
| Q7 | Multi-currency reporting? | Requires FX rates and a conversion date policy | Out of scope for v1; store `currency` and report per currency |
| Q8 | Fiscal year vs calendar year? | "YTD" and "Q3" depend on it | Store `fiscalYearStart` per org; default April (IN) or January |
| Q9 | Should exports be async from day one? | Simpler if not | Start sync; cap at 50k rows |
| Q10 | Who approves new modules/fields? | Governance | Define a lightweight review checklist |

## Risk register

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| Model accuracy below target | Poor UX | Medium | Golden-set evals before UI work; tighten catalog; add examples |
| Local GPU unavailable on demo day | Demo failure | Medium | Groq fallback + pre-recorded flow |
| Cross-tenant leak | Critical | Low | 4 enforcement layers + CI isolation tests |
| Prompt injection via ERP data | Data leak attempt | Medium | No tools; delimited data; validation |
| Export memory blow-up | Outage | Low | Streaming writers, row caps, backpressure |
| Registry drift from real ERP | Wrong reports | Medium | Registry review, boot validation, golden specs |
| Scope creep to "chat with data" | Architecture erosion | High | ADR-1 as the line in the sand |

## References

- OWASP LLM Top 10 — https://genai.owasp.org/llm-top-10/
- Ollama structured outputs — https://docs.ollama.com/capabilities/structured-outputs
- Groq rate limits — https://console.groq.com/docs/rate-limits
- MongoDB aggregation — https://www.mongodb.com/docs/manual/core/aggregation-pipeline/
- Next.js Route Handlers — https://nextjs.org/docs/app/getting-started/route-handlers
- Better Auth organization plugin — https://www.better-auth.com/docs/plugins/organization
- Zod — https://zod.dev/
- ExcelJS — https://github.com/exceljs/exceljs
- csv-stringify — https://csv.js.org/stringify/
- PDFKit — https://pdfkit.org/
- Yarn — https://yarnpkg.com/getting-started/install
