# QueryFlow — Research & Architecture

> Status: research complete. No application code, dependencies, or infrastructure were changed.
> Scope: architecture, AI models, module registry, report engine, security, exports, and delivery plan.

## 1. What QueryFlow is

QueryFlow is an AI-assisted reporting layer for an ERP. A user asks a business question in
natural language:

> "Show the top 10 customers by revenue last quarter, with order count and average order value."

QueryFlow turns that into a **validated report specification**, executes it with trusted backend
code against MongoDB, and returns a table or summary that can be exported to Excel, PDF, or CSV.

It is **not** a text-to-SQL or text-to-MongoDB product. The AI never sees a database connection,
credentials, collection names, raw documents, or query syntax.

## 2. Core architecture at a glance

```
┌──────────────────────────────────────────────────────────────────┐
│ Browser                                                          │
│  Next.js App Router UI  ── POST /api/assistant/query ──┐         │
└────────────────────────────────────────────────────────┼─────────┘
                                                         v
┌──────────────────────────────────────────────────────────────────┐
│ Next.js Route Handler (Node runtime)                             │
│  1. Authenticate session  →  orgId, userId, role                 │
│  2. Rate limit + size limit                                      │
│  3. Load conversation (last spec + recent turns)                 │
│  4. Build permission-filtered module catalog from the registry   │
│  5. Call AI provider with a strict JSON schema                   │
│  6. Parse + Zod validate + semantic validate + authorize         │
│  7. Execute ReportSpec through the report service                │
│  8. Persist turn, audit log, return results                      │
└───────────────┬──────────────────────────────┬───────────────────┘
                │ prompt: schema metadata only │ trusted query
                v                              v
┌───────────────────────────────┐   ┌──────────────────────────────┐
│ AI provider                   │   │ Report service               │
│ Qwen / Gemma via Ollama,      │   │ - allowlisted fields only    │
│ Groq, or another host         │   │ - tenant filter injected     │
│ NO database access            │   │ - typed filters + sort       │
│ NO tools / function calling   │   │ - pagination + limits        │
└───────────────────────────────┘   └──────────────┬───────────────┘
                                                   v
                                        ┌─────────────────────────┐
                                        │ MongoDB (tenant-scoped) │
                                        └─────────────────────────┘
```

The single AI endpoint is the **only** place natural language is interpreted. All other endpoints
accept structured, validated specifications.

## 3. Non-negotiable rules

1. **AI proposes, backend disposes.** Model output is untrusted input, exactly like a request body.
2. **No database access for AI.** No MongoDB URI, driver import, tool calling, or generated query.
3. **Tenant identity comes from the session**, never from the prompt or request body.
4. **The module registry is the contract.** AI sees logical module/field keys, never collection names.
5. **Allowlists everywhere.** Modules, fields, operators, sort keys, report types, and page sizes.
6. **Keep it a modular monolith.** No microservices, message broker, vector database, or agent
   framework for the MVP.

## 4. Research conclusions

| Area | Recommendation |
|---|---|
| App architecture | One Next.js (App Router) full-stack app; React UI + Route Handlers + MongoDB. Collapse the empty `client/` and `server/` folders into `src/app` and `src/server`. |
| Primary AI model | **Qwen3.6-35B-A3B** (Apache 2.0, MoE, 256K context) via local Ollama for privacy and zero per-request cost. Qwen3.5-35B-A3B is an equivalent fallback. |
| Secondary AI model | **Gemma 4 26B A4B or 31B** (Apache 2.0) as a local fallback / comparison. Gemma 4 12B for 16 GB machines. |
| Hosted option | **Groq free tier** with `qwen/qwen3.8-27b` or `openai/gpt-oss-120b` for development/demo. Limits: 30 RPM, 1,000 RPD, 8K TPM, 200K TPD. |
| Not recommended for MVP | Llama 4 Scout/Maverick: custom community license, very large weights, no small dense variant. |
| Structured output | JSON Schema constrained decoding where available (Ollama `format`, Groq `strict`, Gemini `responseSchema`), plus **Zod validation** on every response. Never trust the decoder alone. |
| AI interface | `POST /api/assistant/query` returns a `ReportSpec` (module, report type, filters, columns, sorting, grouping, limit) plus a short user-facing message. |
| Module registry | Code-first TypeScript registry, generated into the prompt at runtime. Adding a module requires no AI code changes. |
| Report engine | Normalized filter AST → typed MongoDB query builder; server-owned operators, joins, computed fields, pagination, and limits. |
| Cross-module reports | Declared relations only, resolved with `$lookup` pipelines that re-apply the tenant filter. Max 2–3 joins. |
| Conversation | Store the last validated spec and 6–10 recent turns. The model returns a full spec; the server merges/validates deterministically. |
| Auth | **Better Auth** with the organization plugin (sessions, orgs, roles, invitations). Session-derived `orgId`. |
| Tenant isolation | Shared collections with a mandatory `tenantId` on every document; `tenantId` first in every compound index; repository-level enforcement + tests. |
| Exports | **ExcelJS** (XLSX), **PDFKit** (PDF), **csv-stringify** (CSV). Reuse the report executor in export mode; stream results. |
| Validation | **Zod 4** schemas shared by API, AI output, and tests. |
| Package manager | **Yarn 4** via Corepack. No npm commands. |
| Deployment | Next.js on a Node host (Vercel/Render/Railway/Fly) + MongoDB Atlas; Ollama on a GPU box or Groq for the AI tier. |

## 5. Why this is the right level of complexity

- A single deployable app is enough for the expected load and keeps debugging simple.
- A code-first registry is easier to review, test, and version than database-driven metadata.
- JSON-schema-constrained generation is more reliable than prompt-only "return JSON".
- The security boundary is architectural (no DB access), not a prompt instruction.
- Exports can start synchronous and streaming; a job queue is only needed when real usage demands it.

## 6. Document index

| File | Contents |
|---|---|
| [`ai-model-comparison.md`](./ai-model-comparison.md) | Qwen vs Gemma vs Llama, local vs hosted free tiers, hardware, recommendation, evaluation plan |
| [`ai-architecture.md`](./ai-architecture.md) | Single-endpoint design, prompt contract, JSON schema, follow-ups, failure handling |
| [`module-registry.md`](./module-registry.md) | Registry schema, permissions, adding modules, prompt generation |
| [`report-engine.md`](./report-engine.md) | ReportSpec, filter validation, sorting, pagination, cross-module queries |
| [`database-and-api-design.md`](./database-and-api-design.md) | Collections, indexes, multi-tenancy, API contracts, folder layout |
| [`security-and-validation.md`](./security-and-validation.md) | AI isolation, prompt injection, authZ, tenant isolation, OWASP LLM Top 10 |
| [`export-system.md`](./export-system.md) | XLSX/PDF/CSV libraries, streaming, formatting, formula injection |
| [`implementation-roadmap.md`](./implementation-roadmap.md) | Phases, deliverables, acceptance criteria, Yarn setup |
| [`technical-decisions.md`](./technical-decisions.md) | ADR-style decisions, trade-offs, risks, open questions |

## 7. Assumptions

- Greenfield project; `client/` and `server/` are empty scaffolds.
- Node.js 20+ (Node 22 LTS preferred), TypeScript strict mode.
- MongoDB 7/8, MongoDB Atlas or self-hosted.
- Next.js current stable with the App Router and Route Handlers.
- Yarn 4 (Berry) via Corepack; `nodeLinker: node-modules` for maximum tool compatibility.
- One organization = one tenant. A user may belong to multiple organizations.
- English UI for v1; the model choice still supports multilingual input.
- The ERP demo modules are Sales, Inventory, Purchases, and Customers.

## 8. Primary references

- Qwen3.5 model card — https://huggingface.co/Qwen/Qwen3.5-35B-A3B
- Qwen3.6-35B-A3B model card — https://huggingface.co/Qwen/Qwen3.6-35B-A3B
- Qwen3.5 blog — https://qwen.ai/blog?id=qwen3.5
- Ollama Qwen3.6 library — https://ollama.com/library/qwen3.6
- Gemma 4 model card — https://ai.google.dev/gemma/docs/core/model_card_4
- Ollama Gemma 4 library — https://ollama.com/library/gemma4
- Llama 4 model card — https://github.com/meta-llama/llama-models/blob/main/models/llama4/MODEL_CARD.md
- Ollama structured outputs — https://docs.ollama.com/capabilities/structured-outputs
- Groq rate limits — https://console.groq.com/docs/rate-limits
- Groq structured outputs — https://console.groq.com/docs/structured-outputs
- Gemini API pricing / free tier — https://ai.google.dev/gemini-api/docs/pricing
- OpenRouter limits — https://openrouter.ai/docs/api-reference/limits
- Next.js Route Handlers — https://nextjs.org/docs/app/getting-started/route-handlers
- MongoDB aggregation pipeline — https://www.mongodb.com/docs/manual/core/aggregation-pipeline/
- MongoDB compound indexes / ESR — https://www.mongodb.com/docs/manual/core/index-compound/
- Better Auth organization plugin — https://www.better-auth.com/docs/plugins/organization
- Zod — https://zod.dev/
- OWASP Top 10 for LLM Applications — https://genai.owasp.org/llm-top-10/
