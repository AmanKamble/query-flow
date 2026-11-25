# Report Engine

> The report engine is the only component allowed to turn a `ReportSpec` into a MongoDB
> query. It is deterministic, allow-listed, and tenant-aware. No AI code lives here.

## 1. Pipeline

```
ReportSpec (untrusted)
   │
   ├─ 1. structural validation   Zod: shape, types, ranges, enums
   ├─ 2. semantic validation     module/field/operator/metric/sort existence
   ├─ 3. authorization           role → module, fields, metrics
   ├─ 4. normalization           period macros, dedupe, clamp limit, default sort
   ├─ 5. compilation             build filter document + aggregation pipeline
   ├─ 6. guard rails             maxTimeMS, max stages, cost checks
   ├─ 7. execution               MongoDB driver
   └─ 8. shaping                 rows + column metadata + formatting hints
```

Steps 1–6 are pure functions where possible, so they are unit-testable without a
database. That matters: most bugs in this system will be in the compiler, not the AI.

## 2. ReportSpec (canonical form)

```ts
export interface ReportSpec {
  module: string;
  reportType: "table" | "summary" | "timeseries" | "breakdown";
  filters: FilterGroup;
  groupBy: GroupKey[];
  metrics: MetricRef[];
  columns: ColumnRef[];
  sort: SortKey[];
  limit: number;
  dateRange?: { from: string; to: string }; // resolved ISO dates, server-owned
}

export interface FilterGroup {
  and?: FilterNode[];
  or?: FilterNode[];
}
export type FilterNode =
  | { field: string; op: Operator; value?: unknown }
  | FilterGroup;

export interface GroupKey {
  field: string;
  // optional time bucketing for timeseries
  bucket?: "day" | "week" | "month" | "quarter" | "year";
}

export interface MetricRef {
  key: string;                 // output key
  agg: "sum" | "avg" | "count" | "min" | "max" | "countDistinct";
  field: string | null;
  label?: string;
  format?: ColumnFormat;
}

export interface ColumnRef { key: string; label?: string; format?: ColumnFormat }
export interface SortKey { key: string; direction: "asc" | "desc" }
```

Invariants enforced by normalization:

- `limit = clamp(limit, 1, module.maxLimit)`.
- `columns` must reference either a `groupBy` field, a metric `key`, or a selected
  non-aggregated field. Unknown keys are rejected.
- `sort.key` must be in `columns`.
- If `groupBy` is empty and metrics exist, treat as a single-row summary.
- Duplicate `columns`/`metrics` keys are rejected (not silently deduped — a duplicate
  key means the model is confused).
- `dateRange`, when present, is injected as an `and` filter on `primaryDateField`.

## 3. Filter compilation

Filters compile to a plain MongoDB query document — **never** to a string, never via
`$where`, never via user-supplied operators.

```ts
function compileFilter(node: FilterNode, module: ModuleDef): object {
  if ("and" in node || "or" in node) {
    const op = "and" in node ? "$and" : "$or";
    const children = (node.and ?? node.or ?? []).map((c) => compileFilter(c, module));
    if (children.length === 0) throw new ValidationError("empty filter group");
    if (children.length > 10) throw new ValidationError("too many filters");
    return { [op]: children };
  }

  const field = module.fields.find((f) => f.key === node.field);
  if (!field) throw new ValidationError(`unknown field: ${node.field}`);
  if (field.filterable === false) throw new ValidationError(`field not filterable: ${field.key}`);

  const op = node.op;
  if (!allowedOperators(field).includes(op)) {
    throw new ValidationError(`operator ${op} not allowed on ${field.type}`);
  }
  const value = coerceValue(field, op, node.value); // strict type coercion
  return { [field.path]: mapOperator(op, value) };
}

function mapOperator(op: Operator, value: unknown) {
  switch (op) {
    case "eq":  return { $eq: value };
    case "ne":  return { $ne: value };
    case "gt":  return { $gt: value };
    case "gte": return { $gte: value };
    case "lt":  return { $lt: value };
    case "lte": return { $lte: value };
    case "in":  return { $in: value as unknown[] };
    case "nin": return { $nin: value as unknown[] };
    case "contains":    return { $regex: escapeRegex(String(value)), $options: "i" };
    case "startsWith":  return { $regex: `^${escapeRegex(String(value))}`, $options: "i" };
    case "between":     return { $gte: (value as [unknown, unknown])[0],
                                 $lte: (value as [unknown, unknown])[1] };
    case "isNull":      return { $eq: null };
    case "isNotNull":   return { $ne: null };
  }
}
```

Critical details:

- **`escapeRegex` is mandatory.** A user-supplied `.*` or `(` otherwise causes ReDoS or
  errors. Escape `.*+?^${}()|[]\`.
- **Type coercion is strict.** `"10"` for an integer field becomes `10`; `"abc"` throws.
  Dates must be ISO `YYYY-MM-DD` or full ISO timestamps; anything else throws.
- **`in`/`nin` arrays are capped** (e.g. 100 items) and type-checked element-wise.
- **No `$where`, `$expr`, `$function`, `$accumulator`, `$regex` from user input.** Only
  the compiler may emit them, and only `$regex` with escaping.
- **`tenantId` is injected by the executor**, not by the compiler's caller, and cannot
  be overridden: if the compiled filter somehow contains `tenantId`, throw.

## 4. Aggregation construction

Table report (no grouping):

```js
[
  { $match: { tenantId, deletedAt: null, ...compiledFilter } },
  { $sort: { [sortPath]: dir } },
  { $skip: offset },
  { $limit: limit },
  { $project: { _id: 0, ...projection } }
]
```

Grouped report:

```js
[
  { $match: { tenantId, deletedAt: null, ...compiledFilter } },
  { $group: {
      _id: groupIdExpression,          // { customerName: "$customerName" } or bucketed date
      revenue:      { $sum: "$totalAmount" },
      orderCount:   { $sum: 1 },
      avgOrderValue:{ $avg: "$totalAmount" }
  } },
  { $addFields: { customerName: "$_id.customerName" } },
  { $project: { _id: 0, customerName: 1, revenue: 1, orderCount: 1, avgOrderValue: 1 } },
  { $sort: { revenue: -1 } },
  { $limit: limit }
]
```

Timeseries bucketing:

```js
{ $group: {
    _id: { $dateTrunc: { date: "$orderDate", unit: "month", timezone: org.timezone } },
    revenue: { $sum: "$totalAmount" }
} }
```

`$dateTrunc` is preferred over `$dateToString` because it returns a real date that
sorts correctly and serializes cleanly. Always pass the org timezone.

### Metric compilation

| `agg` | Stage expression |
|---|---|
| `sum` | `{ $sum: "$field" }` |
| `avg` | `{ $avg: "$field" }` |
| `min` / `max` | `{ $min: "$field" }` / `{ $max: "$field" }` |
| `count` | `{ $sum: 1 }` |
| `countDistinct` | `{ $addToSet: "$field" }` then `{ $size: ... }` in a later `$project` |

`countDistinct` needs two stages; implement it explicitly rather than pretending it is
one expression. For large sets, `$addToSet` can exceed the 16 MB document limit — cap
it or reject it above a configurable cardinality for v1.

### Metric semantics belong in the registry

The AI picks `revenue`; the registry decides that means `sum(totalAmount)` **and**
that cancelled orders are excluded. Implement metric-level `baseFilter` in the
registry:

```ts
{ key: "revenue", agg: "sum", field: "totalAmount",
  baseFilter: { status: { $in: ["confirmed", "fulfilled"] } } }
```

The compiler merges `baseFilter` into `$match` for reports that include that metric.
If metrics in one report need conflicting base filters, reject with a clear error
rather than silently applying one. This is the single biggest correctness win over
"let the model write the filter".

## 5. Cross-module reports

Keep it to **one hop** in v1, at most **two** joins.

```js
[
  { $match: { tenantId, ... } },
  { $lookup: {
      from: "customers",
      let: { cid: "$customerId", tid: "$tenantId" },
      pipeline: [
        { $match: { $expr: { $and: [
            { $eq: ["$_id", "$$cid"] },
            { $eq: ["$tenantId", "$$tid"] }   // ← tenant filter re-applied inside lookup
        ] } } },
        { $project: { _id: 0, segment: 1, city: 1 } }
      ],
      as: "customer"
  } },
  { $unwind: { path: "$customer", preserveNullAndEmptyArrays: true } }
]
```

Rules:

- The **inner `$lookup` pipeline must include the tenant filter.** A naive
  `localField/foreignField` lookup is a cross-tenant data leak if IDs ever collide or
  if a document is mis-tagged.
- Join targets are declared in the registry (`RelationDef`), never chosen by the AI.
- Max 2 joins; each join adds latency and risk.
- `$unwind` on a `many` relation multiplies rows — reject `count`/`sum` metrics on the
  parent in that case, or require the user to report from the child module instead.
- Prefer denormalized fields (see `customerName` in `module-registry.md`) to avoid
  joins entirely.

## 6. Pagination

Two modes:

| Mode | Use | Implementation |
|---|---|---|
| Offset (`page`, `pageSize`) | UI tables, small/medium results | `$skip` + `$limit`; `countDocuments` for totals |
| Cursor (`after`) | Exports, deep pages | Sort on `(sortKey, _id)`, filter `> last`, `$limit n+1` |

Defaults: `pageSize = 50`, max `200` for UI. Exports use cursor mode with batches of
1,000 and a hard cap (e.g. 50,000 rows) before switching to an async job.

`$skip` is O(n) on the server; cap `page * pageSize` at e.g. 10,000 and require
narrower filters beyond that.

Counts are expensive on large collections. Make `total` optional (`includeTotal=false`
by default for grouped reports) and show "50+ results" in the UI.

## 7. Guard rails

| Guard | Value | Purpose |
|---|---|---|
| `maxTimeMS` | 5,000 ms | Bound a single query |
| Max filter nodes | 10 per group, depth 2 | Prevent pathological queries |
| Max `$lookup` | 2 | Latency and leak surface |
| Max limit | 500 (UI), 50,000 (export) | Memory |
| Max columns | 12 | Payload size, readability |
| Max regex length | 100 chars | ReDoS |
| Reject `$where`, `$expr` from users | Always | Injection / cost |
| Reject unbounded `contains` on large collections | Warn or block | Full scan |
| `allowDiskUse: false` for UI queries | — | Fail fast instead of spilling |

Optional: run `explain("executionStats")` in development and fail the test suite if
`totalDocsExamined / nReturned > 100` for the golden queries. That is a practical way to
catch missing indexes before production.

## 8. Result shaping

The engine returns data, not presentation:

```ts
export interface ReportResult {
  columns: { key: string; label: string; type: FieldType; format?: ColumnFormat }[];
  rows: unknown[][];              // array-of-arrays: smaller and order-stable
  rowCount: number;
  truncated: boolean;
  total?: number;
  executionMs: number;
  warnings: string[];
}
```

- Decimal values: store money as `Decimal128` in MongoDB, convert to `number` only at
  the edge if precision is acceptable, otherwise serialize as strings with a `decimal`
  format. For ERP financials, **string + format** is safer than IEEE floats.
- Dates: ISO 8601 strings in the payload; format in the UI/export layer using the org
  timezone.
- Nulls are preserved, not converted to `0`. "No orders" and "zero revenue" are
  different facts.
- `truncated` is set when `limit` was hit, so the UI can say "showing first 500".

## 9. Caching

- **Spec cache:** hash `(orgId, role, normalizedQuestion, catalogVersion)` → validated
  spec, TTL 10 min. Skips the model, not the database.
- **Result cache:** optional, TTL 30–60 s, keyed by `(orgId, specHash, dataVersion)`.
  Only enable if the ERP has a cheap `dataVersion` counter; otherwise stale financial
  numbers are worse than a slow query.
- **Do not cache across tenants.** Include `orgId` in every key, always.

## 10. Error taxonomy

| Code | HTTP | Meaning |
|---|---|---|
| `SPEC_INVALID` | 422 | Zod/semantic validation failed after repair |
| `FIELD_NOT_ALLOWED` | 403 | Field exists but role cannot see it |
| `MODULE_NOT_ALLOWED` | 403 | Role cannot query the module |
| `UNSUPPORTED_QUESTION` | 200 | Out of scope; `status: "unsupported"` |
| `QUERY_TIMEOUT` | 504 | `maxTimeMS` exceeded |
| `QUERY_TOO_BROAD` | 422 | Guard rail tripped (e.g. unbounded regex) |
| `RATE_LIMITED` | 429 | Per-user/org limit |
| `PROVIDER_UNAVAILABLE` | 503 | Model provider down and no fallback |

Errors return a stable `code` and a human message. Never echo raw Mongo errors to the
client (they leak field and index names).

## 11. Testing strategy

1. **Compiler unit tests** — snapshot the generated pipeline for ~30 specs. Review the
   snapshots in PRs; a changed pipeline is a changed report.
2. **Property tests** — generate random valid specs; assert the compiler never emits
   `$where`, never omits `tenantId`, and always respects `limit`.
3. **Integration tests** — seed two tenants; assert org A's spec cannot return org B
   rows, including through `$lookup`.
4. **Golden AI evals** — question → expected spec (see `ai-model-comparison.md`).
5. **Performance tests** — golden queries with `explain`, index coverage assertions.

## 12. References

- Aggregation pipeline — https://www.mongodb.com/docs/manual/core/aggregation-pipeline/
- Aggregation limits (16 MB docs, 100 MB memory) — https://www.mongodb.com/docs/manual/core/aggregation-pipeline-limits/
- `$lookup` — https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/
- `$dateTrunc` — https://www.mongodb.com/docs/manual/reference/operator/aggregation/dateTrunc/
- `$group` — https://www.mongodb.com/docs/manual/reference/operator/aggregation/group/
- Query optimization / ESR — https://www.mongodb.com/docs/manual/core/query-optimization/
- `maxTimeMS` — https://www.mongodb.com/docs/manual/reference/method/cursor.maxTimeMS/
