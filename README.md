# QueryFlow

QueryFlow is an AI-powered ERP reporting platform that turns natural-language questions into
validated, executable reports. Ask about sales, inventory, purchases, or customers, then refine,
save, and export the result to Excel, PDF, or CSV.

Powered by open-weight models running locally, with a hard architectural boundary: **the AI never
touches the database.**

> **Status:** research and architecture complete. Application code has not been written yet.
> See [`research/`](./research/) for the full design.

---

## The idea

```
"Top 10 customers by revenue last quarter, with order count and average order value"
        │
        ▼
   ReportSpec (JSON)  →  validated  →  compiled query  →  results table  →  XLSX / PDF / CSV
```

The model's only job is to translate a question into a **structured report specification**. Trusted
backend code validates that specification, applies tenant permissions, queries MongoDB, and shapes
the result.

It is **not** text-to-SQL or text-to-MongoDB. The model never sees a connection string, a
collection name, a raw document, or a query it could execute.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ Browser — Next.js App Router UI                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │ POST /api/v1/assistant/query
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Backend (Next.js Route Handlers, Node runtime)                  │
│  1. Authenticate session        → userId, orgId, role           │
│  2. Rate limit + size limit                                     │
│  3. Build role-filtered module catalog                          │
│  4. Call model with a strict JSON schema                        │
│  5. Zod validate + semantic validate + authorize                │
│  6. Compile ReportSpec → MongoDB query                          │
│  7. Execute, audit, respond                                     │
└──────────────┬───────────────────────────────┬──────────────────┘
               │ schema metadata + question    │ trusted query
               ▼                               ▼
    ┌──────────────────────┐         ┌──────────────────────┐
    │ Local model (Ollama) │         │ MongoDB (tenant-     │
    │ Qwen3.6-35B-A3B      │         │ scoped collections)  │
    │ no DB access, no     │         └──────────────────────┘
    │ tools, no function   │
    │ calling              │
    └──────────────────────┘
```

### Security model

| Rule | How it is enforced |
|---|---|
| AI cannot read or write the database | No driver import, no connection string, no tools — plus a lint rule banning DB imports from `src/server/ai/**` |
| Tenant identity cannot be spoofed | `orgId` comes from the verified session; injected at execution, never accepted from a request or the model |
| The model can only use known fields | Code-first module registry; unknown modules/fields/operators are rejected |
| Model output is untrusted input | JSON-Schema-constrained generation **and** Zod + semantic validation |
| No arbitrary query syntax | Allow-listed operators compiled to typed MongoDB filters; no `$where`, no generated strings |
| Exports cannot attack the user | Formula-injection escaping on every cell; role checks and row caps on every export |

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| App | Next.js (App Router) + TypeScript strict | One deploy, shared types, server-side data access |
| UI | React + Tailwind CSS | Fast to build, consistent |
| API | Next.js Route Handlers | Same runtime as the DB and export libraries |
| Database | MongoDB 7/8 | Nested ERP documents + aggregation for reporting |
| AI runtime | Ollama (local) | Private, no per-token cost, structured outputs |
| Primary model | Qwen3.6-35B-A3B (Apache 2.0) | Strong JSON adherence, MoE speed, 262K context |
| Small-machine model | Gemma 4 12B (Apache 2.0) | Runs on 16 GB laptops |
| Hosted fallback | Groq free tier (OpenAI-compatible) | Development and demo resilience |
| Validation | Zod 4 | One schema for API, model output, and tests |
| Auth | Better Auth + organization plugin | Sessions, orgs, roles, invitations |
| Exports | ExcelJS · csv-stringify · PDFKit | Streaming, no native dependencies |
| Package manager | Yarn 4 (Corepack) | Deterministic installs |

## Project structure (planned)

```
query-flow/
├─ src/
│  ├─ app/                 # pages + route handlers
│  ├─ server/
│  │  ├─ ai/               # provider adapter, prompt, schema, repair
│  │  ├─ modules/          # module registry (the AI's vocabulary)
│  │  ├─ report/           # validate, compile, execute, shape
│  │  ├─ auth/             # session + RBAC
│  │  ├─ db/               # client, scoped queries, indexes
│  │  └─ export/           # xlsx, csv, pdf
│  ├─ shared/              # Zod schemas and types
│  └─ components/
├─ scripts/                # seed, create-indexes, migrate
├─ research/               # architecture and technology research
└─ .env.example
```

The current `client/` and `server/` folders are empty scaffolds and will be consolidated into the
single Next.js app above.

## Documentation

Start with [`research/README.md`](./research/README.md), the index for the full design.

| Document | What it covers |
|---|---|
| [`research/ai-model-comparison.md`](./research/ai-model-comparison.md) | Qwen vs Gemma vs Llama, local vs hosted free tiers, hardware sizing, evaluation plan |
| [`research/ai-architecture.md`](./research/ai-architecture.md) | The single AI endpoint, prompt contract, JSON schema, retries, provider adapter |
| [`research/module-registry.md`](./research/module-registry.md) | Registry schema, roles, example modules, prompt serialization |
| [`research/report-engine.md`](./research/report-engine.md) | ReportSpec, filter compiler, aggregation, joins, pagination, guard rails |
| [`research/database-and-api-design.md`](./research/database-and-api-design.md) | Collections, indexes, multi-tenancy, API contracts |
| [`research/security-and-validation.md`](./research/security-and-validation.md) | Threat model, OWASP LLM Top 10 mapping, test plan |
| [`research/export-system.md`](./research/export-system.md) | XLSX/CSV/PDF generation, streaming, formula injection |
| [`research/implementation-roadmap.md`](./research/implementation-roadmap.md) | Phases, deliverables, acceptance criteria, Yarn setup |
| [`research/technical-decisions.md`](./research/technical-decisions.md) | ADRs, trade-offs, risks, open questions |

## Roadmap

| Phase | Scope |
|---|---|
| 0 | Tooling: Next.js, Yarn 4, TypeScript, lint, test, CI |
| 1 | Data layer, indexes, seed data, tenant isolation tests |
| 2 | Auth, organizations, roles |
| 3 | Module registry + report engine (no AI yet) |
| 4 | AI endpoint, prompt, validation, repair loop |
| 5 | Assistant UI, filter editor, saved reports |
| 6 | Excel / CSV / PDF exports |
| 7 | Evaluation suite, hardening, demo packaging |

Estimated at ~10 weeks for a first release. Details in
[`research/implementation-roadmap.md`](./research/implementation-roadmap.md).

## Design principles

1. **AI proposes, backend disposes.** Model output is untrusted input, exactly like a request body.
2. **No database access for the AI.** Not a prompt instruction — an architectural boundary.
3. **Allowlists everywhere.** Modules, fields, operators, sort keys, page sizes.
4. **Tenant identity comes from the session.** Never from a prompt, body, or query string.
5. **Deterministic execution.** Report results are computed by code, not generated by a model.
6. **Modular monolith.** No microservices, message broker, or vector database for the MVP.

## Open questions

- Target hardware for the demo (decides 35B vs 12B model).
- ERP modules in scope for v1 (assumed Sales, Inventory, Purchases, Customers).
- Whether ERP data must be imported or already lives in MongoDB.
- Fiscal-year definition for "Q3" / "YTD".
- Need for row-level security (e.g. a sales rep seeing only their own orders).

## License

Not yet decided.
