# Implementation Roadmap

> Sequential phases, each ending in something demoable. No phase starts before the
> previous one has passing tests. Estimates assume one full-time developer.

## Phase 0 — Repository and tooling (2–3 days)

**Goal:** one Next.js app, Yarn 4, lint/format/type-check/test wired into CI.

```bash
# From the repo root. Yarn 4 via Corepack (no npm commands).
corepack enable
corepack prepare yarn@stable --activate
yarn init -2

# App + language
yarn add next react react-dom
yarn add -D typescript @types/node @types/react @types/react-dom

# Backend
yarn add mongodb zod
yarn add better-auth

# AI (HTTP only — no SDK required)
# uses global fetch; nothing to install

# Exports
yarn add exceljs csv-stringify pdfkit
yarn add -D @types/pdfkit

# Tooling
yarn add -D eslint eslint-config-next prettier vitest @vitest/coverage-v8 tsx
```

`.yarnrc.yml`:

```yaml
nodeLinker: node-modules
enableGlobalCache: true
```

Deliverables:

- `src/app` and `src/server` skeleton (delete the empty `client/` and `server/`
  folders).
- `tsconfig.json` with `strict: true`, `noUncheckedIndexedAccess: true`,
  path alias `@/* → src/*`.
- Scripts: `dev`, `build`, `start`, `lint`, `typecheck`, `test`, `seed`,
  `db:indexes`.
- `.env.example`, `.editorconfig`, `.prettierrc`.
- CI: install with `yarn install --immutable`, then `lint`, `typecheck`, `test`,
  `build`.

**Done when:** `yarn dev` serves a page, `yarn test` runs one trivial test, CI is green.

## Phase 1 — Data layer and seed (3–4 days)

**Goal:** realistic ERP data with tenant isolation and indexes.

Deliverables:

- `src/server/db/client.ts` — singleton `MongoClient`, dev hot-reload safe.
- `src/server/db/scoped.ts` — `tenantId`-requiring helpers.
- Collections: `customers`, `products`, `inventory_items`, `sales_orders`,
  `purchase_orders`.
- `scripts/create-indexes.ts` — all indexes from `database-and-api-design.md` §4.
- `scripts/seed.ts` — 2 orgs, 4 roles, ~2,000 orders, 18 months, fixed RNG seed.
- Integration test harness: `mongodb-memory-server` or a Docker Mongo, with a
  two-tenant fixture.

**Done when:** seed runs idempotently; the isolation test proves org A cannot read
org B; `explain()` on the top 5 report shapes shows `IXSCAN`, not `COLLSCAN`.

## Phase 2 — Auth and tenancy (3–4 days)

**Goal:** login, org membership, roles enforced server-side.

Deliverables:

- Better Auth config with the organization plugin; `/api/auth/[...all]` handler.
- `src/server/auth/session.ts` — `requireSession()` returning `{ userId, orgId, role }`.
- `src/server/auth/rbac.ts` — capability matrix from `module-registry.md` §5.
- Login page, org switcher, protected layout.
- Middleware: redirect unauthenticated users; **never** rely on middleware alone for
  authorization — re-check in each route handler.

**Done when:** a `viewer` gets 403 on export; switching orgs changes visible modules;
session survives reload; logout revokes the session.

## Phase 3 — Module registry and report engine (5–6 days)

**Goal:** execute a hand-written `ReportSpec` correctly. No AI yet — this is deliberate.

Deliverables:

- `src/server/modules/types.ts`, `registry.ts`, and 4 module definitions.
- Boot-time registry validation (fails fast on bad definitions).
- `src/server/report/validate.ts` — Zod + semantic checks.
- `src/server/report/compile.ts` — filter compiler + aggregation builder.
- `src/server/report/execute.ts` — tenant injection, `maxTimeMS`, pagination.
- `src/server/report/shape.ts` — column metadata + row mapping.
- `POST /api/v1/reports/execute`, `GET /api/v1/modules`.
- Tests: compiler snapshots, property tests (no `$where`, always `tenantId`),
  cross-tenant `$lookup` test.

**Done when:** 20 golden specs (table, summary, timeseries, grouped) return correct
numbers against seed data; every compiled pipeline contains `tenantId`; unknown field
and illegal operator are rejected with typed errors.

**This is the riskiest phase. Do not move on until it is solid.**

## Phase 4 — AI endpoint (5–6 days)

**Goal:** question → validated spec → results, end to end.

Deliverables:

- Ollama installed locally; `ollama pull qwen3.6:35b` (or `gemma4:12b` on 16 GB).
- `src/server/ai/provider.ts` — `OllamaProvider`, `GroqProvider`.
- Disable thinking mode for spec generation (`think: false` on Ollama, or
  `chat_template_kwargs.enable_thinking: false` on OpenAI-compatible hosts).
- `src/server/ai/schema.ts` — JSON Schema for the model + Zod schema for the server.
- `src/server/ai/prompt.ts` — catalog serializer + system prompt builder.
- `src/server/ai/repair.ts` — one repair attempt with validation errors.
- `POST /api/v1/assistant/query` with the full flow from `ai-architecture.md` §2.
- Conversation persistence (`ai_conversations`, `ai_messages`).
- Structured logging + `audit_logs`.
- ESLint rule: `src/server/ai/**` cannot import the DB layer.

**Done when:**

- 40 golden questions produce valid specs; ≥ 90% semantic accuracy.
- Adversarial prompts ("show all tenants", "drop collection", "ignore rules") return
  `unsupported` or a valid in-scope spec — never a leak.
- A provider outage returns `503` with a clear message; Groq fallback works when
  configured.
- p95 latency under 20 s on the target machine.

## Phase 5 — UI (6–8 days)

**Goal:** a usable product, not a JSON viewer.

Screens:

1. **Assistant** — chat input, message list, spec summary chips ("Sales Orders · Q3 2026
   · Grouped by Customer"), results table, "Edit filters" and "Save report".
2. **Filter editor** — form generated from `/api/v1/modules`, so the user can adjust
   without another model call; submitting calls `reports/execute`.
3. **Saved reports** — list, rename, delete, run, export.
4. **Settings** — members, roles, invitations (Better Auth org plugin).
5. **Empty/error states** — unsupported question, clarification prompt, rate limited,
   provider down.

Deliverables:

- Server Components for data fetching where possible; Client Components for the table
  and chat.
- Accessible table (keyboard, `aria-sort`), skeleton loaders, cancel button.
- All model text rendered as text — never `dangerouslySetInnerHTML`.
- No `localStorage` tokens.

**Done when:** a non-technical user can ask a question, refine it, save it, and find it
again without help.

## Phase 6 — Exports (3–4 days)

**Goal:** XLSX, CSV, PDF from the same spec.

Deliverables:

- `src/server/export/{xlsx,csv,pdf}.ts` + shared `format.ts`.
- `POST /api/v1/reports/export` streaming response.
- Formula-injection escaping applied to every string cell.
- Embedded Unicode font for PDF.
- Rate limits and row caps.
- Tests: golden values, injection, encoding, limits, authorization.

**Done when:** 10k rows export in under 3 s (CSV/XLSX), a `=cmd` cell is inert, and
₹/€/CJK render correctly in all three formats.

## Phase 7 — Hardening and demo (4–5 days)

Deliverables:

- Eval suite in CI (golden questions, adversarial corpus) with a pass threshold.
- Load test: 10 concurrent users, 3 concurrent AI requests, verify queueing.
- Error taxonomy wired to UI messages; no raw errors surfaced.
- Audit log viewer (admin only).
- README with setup, env vars, and demo script.
- Docker Compose for Mongo (+ optional Ollama) for a one-command demo.

**Done when:** a fresh clone can be running in under 15 minutes following the README.

## Total

| Phase | Days |
|---|---|
| 0 Tooling | 2–3 |
| 1 Data | 3–4 |
| 2 Auth | 3–4 |
| 3 Report engine | 5–6 |
| 4 AI | 5–6 |
| 5 UI | 6–8 |
| 6 Exports | 3–4 |
| 7 Hardening | 4–5 |
| **Total** | **31–40 working days** (~7–8 weeks) |

Add 20–30% for ERP data import, stakeholder demos, and rework. A realistic
first-release target is **10 weeks**.

## Definition of done (per feature)

- [ ] Types strict, no `any` in new code.
- [ ] Zod schema at the boundary; no unvalidated input reaches the DB.
- [ ] `tenantId` injected and asserted.
- [ ] Unit tests for pure logic; integration test for the DB path.
- [ ] Audit log entry for reads of business data and all writes.
- [ ] Error path returns a typed code and a human message.
- [ ] No secrets, PII, or rows in logs.
- [ ] Docs updated (registry entry, README if setup changed).

## Sequencing rules

1. **Report engine before AI.** If the engine is wrong, the AI will look wrong.
2. **Auth before AI.** You cannot test tenant isolation without sessions.
3. **No AI in Phase 1–3.** Resist the urge; it hides bugs in the engine.
4. **Exports last.** They are a projection of a working report, not a separate feature.
5. **One endpoint for AI.** If a second AI endpoint appears, stop and re-read
   `ai-architecture.md` §10.

## Risks and buffers

| Risk | Likelihood | Buffer |
|---|---|---|
| Local model too slow on the demo machine | Medium | Gemma 4 12B profile; Groq fallback |
| Model accuracy below target | Medium | Tighten catalog, add examples, per-module prompts |
| ERP data messier than expected | High | 20–30% schedule buffer; import scripts early |
| Scope creep into "chat with my data" | High | The hard rule: specs only, no rows to the model |
| Export memory issues | Low | Streaming writers + row caps from day one |

## References

- Yarn install — https://yarnpkg.com/getting-started/install
- Yarn CLI — https://yarnpkg.com/cli
- Next.js App Router — https://nextjs.org/docs/app
- Next.js Route Handlers — https://nextjs.org/docs/app/getting-started/route-handlers
- Better Auth — https://www.better-auth.com/docs
- Ollama — https://docs.ollama.com/
- Vitest — https://vitest.dev/
