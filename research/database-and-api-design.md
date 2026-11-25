# Database and API Design

> MongoDB schema, indexing, multi-tenancy, and the HTTP surface. Keep it small: one
> database, shared collections, `tenantId` on everything.

## 1. Database choice

**MongoDB 7/8** (Atlas or self-hosted), one database per environment (`queryflow_dev`,
`queryflow_prod`).

Why Mongo fits:

- ERP documents are naturally nested (order → line items, address objects).
- Reporting is aggregation-heavy; `$group`/`$lookup`/`$dateTrunc` cover the MVP needs.
- Flexible fields help during early ERP integration.

Where to be careful:

- Money must be `Decimal128`, not `Double`.
- Aggregations on unindexed fields are full scans — the registry's `filterable`/
  `sortable` flags exist to keep this honest.
- No transactions needed for read-only reporting; use them for writes (orders, imports).

## 2. Multi-tenancy model

**Shared collections + mandatory `tenantId`.** One org = one tenant.

| Model | Verdict |
|---|---|
| Shared collections, `tenantId` field | ✅ Recommended. Simple, cheap, one index strategy |
| Database per tenant | ❌ Too much operational overhead for v1 |
| Collection per tenant | ❌ Index/namespace explosion, painful migrations |

Enforcement (all four layers, not just one):

1. **Schema:** `tenantId: ObjectId` required on every tenant-owned document.
2. **Repository:** every query goes through a helper that requires `tenantId`:
   ```ts
   function scoped(tenantId: ObjectId, filter: Filter<Document> = {}) {
     if (!tenantId) throw new Error("tenantId required");
     return { ...filter, tenantId };
   }
   ```
3. **Report engine:** injects `tenantId` and rejects specs/filters that mention it.
4. **Tests:** a suite that seeds two tenants and asserts zero leakage across every
   endpoint, including `$lookup`.

Never accept `tenantId` from a request body, query string, or model output. It comes
from the session only.

## 3. Collections

### Identity (Better Auth)

Better Auth owns `user`, `session`, `account`, `verification`, and with the
organization plugin `organization`, `member`, `invitation`. Let it create and migrate
these; do not hand-roll auth tables.

QueryFlow adds `tenantId` semantics by mapping `organization.id` → `tenantId`.

### Domain

```js
// customers
{
  _id: ObjectId,
  tenantId: ObjectId,          // required, indexed
  name: String,
  email: String,               // pii
  phone: String,               // pii
  segment: String,             // "enterprise" | "smb" | "retail"
  city: String,
  country: String,
  creditLimit: Decimal128,
  isActive: Boolean,
  createdAt: Date, updatedAt: Date, deletedAt: Date | null
}
```

```js
// products
{
  _id, tenantId,
  sku: String, name: String, category: String,
  unitPrice: Decimal128, costPrice: Decimal128,
  unit: String,
  isActive: Boolean,
  createdAt, updatedAt, deletedAt
}
```

```js
// inventory_items  (one row per product per warehouse)
{
  _id, tenantId,
  productId: ObjectId, sku: String, productName: String,
  warehouse: String,
  quantityOnHand: Number, reorderLevel: Number,
  unitCost: Decimal128,
  lastCountedAt: Date,
  createdAt, updatedAt, deletedAt
}
```

```js
// sales_orders
{
  _id, tenantId,
  orderNumber: String,
  customerId: ObjectId,
  customerName: String,        // denormalized for reporting
  orderDate: Date,
  status: String,              // draft|confirmed|fulfilled|cancelled
  region: String,
  salesRepId: ObjectId | null,
  salesRepName: String,
  currency: String,            // "INR"
  subtotal: Decimal128,
  taxAmount: Decimal128,
  totalAmount: Decimal128,
  items: [{ productId, sku, name, quantity, unitPrice, lineTotal }],
  createdAt, updatedAt, deletedAt
}
```

```js
// purchase_orders — same shape as sales_orders with supplierId/supplierName
```

### AI + reporting

```js
// ai_conversations
{
  _id, tenantId, userId,
  title: String,               // first question, truncated
  lastSpec: Object | null,     // last VALIDATED spec
  registryVersion: Number,
  createdAt, updatedAt
}
```

```js
// ai_messages
{
  _id, tenantId, conversationId, role: "user" | "assistant",
  question: String,            // user turn
  spec: Object | null,         // assistant turn: validated spec
  status: "ok" | "clarification" | "unsupported" | "invalid" | "error",
  provider: String, model: String,
  latencyMs: Number, promptTokens: Number, completionTokens: Number,
  validationAttempts: Number,
  createdAt
}
```

Note: **no result rows are stored.** Re-running a report is cheap and storing rows
duplicates PII and grows unbounded.

```js
// saved_reports
{
  _id, tenantId, userId, name: String, description: String,
  spec: Object,                // resolved spec snapshot
  registryVersion: Number,
  visibility: "private" | "org",
  createdAt, updatedAt
}
```

```js
// audit_logs
{
  _id, tenantId, userId,
  action: String,              // "assistant.query" | "report.export" | "report.execute"
  resource: String,            // module key or report id
  specHash: String,
  ip: String, userAgent: String,
  status: String, rowCount: Number, executionMs: Number,
  createdAt
}
```

```js
// export_jobs  (only when async exports are added)
{
  _id, tenantId, userId, reportId: ObjectId | null, spec: Object,
  format: "xlsx" | "csv" | "pdf",
  status: "queued" | "running" | "done" | "failed" | "expired",
  fileKey: String | null, fileSize: Number | null,
  error: String | null,
  expiresAt: Date,
  createdAt, startedAt, finishedAt
}
```

## 4. Indexes

Golden rule: **`tenantId` first**, then equality fields, then sort fields, then range
fields (ESR — Equality, Sort, Range).

```js
// customers
{ tenantId: 1, name: 1 }
{ tenantId: 1, segment: 1, createdAt: -1 }
{ tenantId: 1, email: 1 }                       // unique per tenant
// unique compound: { tenantId: 1, email: 1 }, { unique: true, sparse: true }

// products
{ tenantId: 1, sku: 1 }                         // unique per tenant
{ tenantId: 1, category: 1, name: 1 }

// inventory_items
{ tenantId: 1, warehouse: 1, productId: 1 }
{ tenantId: 1, quantityOnHand: 1 }

// sales_orders
{ tenantId: 1, orderDate: -1 }
{ tenantId: 1, status: 1, orderDate: -1 }
{ tenantId: 1, customerId: 1, orderDate: -1 }
{ tenantId: 1, region: 1, orderDate: -1 }
{ tenantId: 1, salesRepId: 1, orderDate: -1 }
{ tenantId: 1, orderNumber: 1 }                 // unique per tenant

// purchase_orders — mirror sales_orders with supplierId

// ai_conversations
{ tenantId: 1, userId: 1, updatedAt: -1 }
// ai_messages
{ tenantId: 1, conversationId: 1, createdAt: 1 }
// saved_reports
{ tenantId: 1, userId: 1, updatedAt: -1 }
// audit_logs
{ tenantId: 1, createdAt: -1 }
{ tenantId: 1, userId: 1, createdAt: -1 }
```

Notes:

- Compound indexes are usable by their **prefix**, so `{tenantId:1, status:1, orderDate:-1}`
  also serves `{tenantId}` and `{tenantId, status}` queries.
- A unique index on `{tenantId, orderNumber}` prevents duplicates per tenant without
  requiring global uniqueness.
- Every `deletedAt` filter is low-selectivity; do not index it alone.
- Use `partialFilterExpression: { deletedAt: null }` only if soft-deleted rows dominate
  and you measure a win — otherwise it complicates queries.
- Text search (`contains`) does not use a normal index. For v1, restrict `contains` to
  small collections or add a MongoDB Atlas Search index later. Do not let the AI choose
  `contains` on `sales_orders.notes`.

## 5. Data conventions

| Concern | Convention |
|---|---|
| IDs | `ObjectId`; serialize as 24-char hex strings |
| Money | `Decimal128`, always with a `currency` code |
| Dates | BSON `Date` (UTC); display in org timezone |
| Soft delete | `deletedAt: Date \| null`; all reads filter `deletedAt: null` |
| Enums | lowercase strings; validated in the registry |
| Denormalization | `customerName`, `supplierName`, `productName` on line items for reporting |
| Timestamps | `createdAt`, `updatedAt` on every document |
| Audit | `audit_logs` is append-only; never update or delete |

Denormalization is a deliberate trade: faster reports, no joins, at the cost of a
backfill when a name changes. Document the backfill as a maintenance job.

## 6. API surface

Base path `/api/v1`. Session cookie auth (httpOnly, `SameSite=Lax`, `Secure`).

| Method | Path | Purpose |
|---|---|---|
| `*` | `/api/auth/[...all]` | Better Auth handler |
| `GET` | `/api/v1/me` | Current user, orgs, role |
| `GET` | `/api/v1/modules` | Modules + fields visible to the role (drives UI filters) |
| `POST` | `/api/v1/assistant/query` | **The only AI endpoint** |
| `POST` | `/api/v1/reports/execute` | Execute a saved/edited spec (no AI) |
| `GET` | `/api/v1/reports/saved` | List saved reports |
| `POST` | `/api/v1/reports/saved` | Save a spec |
| `PATCH` | `/api/v1/reports/saved/:id` | Rename / edit |
| `DELETE` | `/api/v1/reports/saved/:id` | Delete |
| `POST` | `/api/v1/reports/export` | Export a spec (xlsx/csv/pdf) |
| `GET` | `/api/v1/exports/:id` | Poll/download async export |
| `GET` | `/api/v1/health` | Liveness + DB ping |

Design rules:

- `reports/execute` accepts the **same `ReportSpec`** as the AI endpoint returns. This
  lets the UI edit filters without another model call and keeps one validator.
- Every mutating route requires CSRF protection (Better Auth handles it for its routes;
  for custom routes use an origin check + SameSite cookie).
- All routes are tenant-scoped by session, never by path or body.
- Return `Cache-Control: no-store` on anything containing business data.

### Example: execute a spec

```http
POST /api/v1/reports/execute
{
  "spec": { "module": "sales_orders", "reportType": "table", ... },
  "page": 1,
  "pageSize": 50,
  "includeTotal": false
}
```

```json
{
  "columns": [{ "key": "customerName", "label": "Customer", "type": "string" }],
  "rows": [["Acme Corp", 184230.5]],
  "rowCount": 50,
  "truncated": true,
  "executionMs": 62,
  "warnings": []
}
```

### Error envelope

```json
{
  "error": {
    "code": "FIELD_NOT_ALLOWED",
    "message": "You do not have access to the field 'costPrice'.",
    "requestId": "req_01J..."
  }
}
```

Use stable machine codes (see `report-engine.md` §10), never raw driver messages.

## 7. Project structure

Collapse the empty `client/` + `server/` folders into one Next.js app:

```
query-flow/
├─ src/
│  ├─ app/
│  │  ├─ (auth)/login/page.tsx
│  │  ├─ (app)/
│  │  │  ├─ layout.tsx
│  │  │  ├─ assistant/page.tsx
│  │  │  ├─ reports/[id]/page.tsx
│  │  │  └─ settings/page.tsx
│  │  └─ api/
│  │     ├─ auth/[...all]/route.ts
│  │     └─ v1/
│  │        ├─ assistant/query/route.ts
│  │        ├─ reports/execute/route.ts
│  │        ├─ reports/export/route.ts
│  │        └─ modules/route.ts
│  ├─ server/
│  │  ├─ ai/           provider.ts, prompt.ts, schema.ts, repair.ts
│  │  ├─ modules/      types.ts, registry.ts, sales-orders.ts, customers.ts, ...
│  │  ├─ report/       validate.ts, compile.ts, execute.ts, shape.ts
│  │  ├─ auth/         auth.ts, session.ts, rbac.ts
│  │  ├─ db/           client.ts, collections.ts, indexes.ts, scoped.ts
│  │  ├─ export/       xlsx.ts, csv.ts, pdf.ts
│  │  └─ audit/        log.ts
│  ├─ shared/          schemas (Zod), types, constants
│  └─ components/      ui/
├─ research/
├─ scripts/            seed.ts, create-indexes.ts
├─ .env.example
└─ package.json
```

Why one app instead of `client/` + `server/`:

- One deploy, one auth context, one type system, shared Zod schemas.
- Route Handlers run on Node, so the MongoDB driver and PDF/Excel libraries work.
- If the AI tier later needs a separate service (GPU host), it can be split without
  changing the API contract — but do not start there.

## 8. Environment variables

```bash
# app
NODE_ENV=development
APP_URL=http://localhost:3000
SESSION_SECRET=            # 32+ random bytes

# database
MONGODB_URI=mongodb://localhost:27017/queryflow_dev
MONGODB_DB=queryflow_dev

# AI
AI_PROVIDER=ollama         # ollama | groq
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=qwen3.6:35b
GROQ_API_KEY=              # optional fallback
GROQ_MODEL=qwen/qwen3.8-27b

# limits
AI_TIMEOUT_MS=25000
QUERY_TIMEOUT_MS=5000
MAX_ROWS_UI=500
MAX_ROWS_EXPORT=50000

# exports
EXPORT_DIR=./.exports
EXPORT_TTL_HOURS=24
```

`.env` is gitignored (already in `.gitignore`). Commit `.env.example` only.

## 9. Seeding

`scripts/seed.ts` creates:

- 1 org, 4 users (one per role), 1 second org for isolation tests;
- 40 customers, 60 products, 120 inventory rows;
- 2,000 sales orders across 18 months with realistic status/region distribution;
- 400 purchase orders.

Use a fixed RNG seed so tests and demos are reproducible.

## 10. Migration strategy

- Use `migrate-mongo` or a small `scripts/migrate.ts` with ordered, idempotent steps.
- Index creation is a migration step (`scripts/create-indexes.ts`), run on deploy.
- Never rely on Mongoose `autoIndex` in production — it can block on large collections.

## 11. References

- MongoDB data modeling — https://www.mongodb.com/docs/manual/core/data-modeling-introduction/
- Compound indexes and ESR — https://www.mongodb.com/docs/manual/core/index-compound/
- Index prefixes — https://www.mongodb.com/docs/manual/core/index-compound/#prefixes
- `$lookup` — https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/
- MongoDB limits (16 MB doc) — https://www.mongodb.com/docs/manual/reference/limits/
- Next.js Route Handlers — https://nextjs.org/docs/app/getting-started/route-handlers
- Next.js backend-for-frontend — https://nextjs.org/docs/app/guides/backend-for-frontend
- Better Auth organization plugin — https://www.better-auth.com/docs/plugins/organization
