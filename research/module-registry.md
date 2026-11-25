# Module Registry

> The registry is the contract between the ERP and the AI. It is the only vocabulary the
> model is allowed to use, and the only vocabulary the report engine will execute.

## 1. Why a registry (not a database of metadata)

| Approach | Verdict |
|---|---|
| **Code-first TypeScript registry** | ✅ Recommended. Type-checked, reviewable in PRs, versioned with the code, testable |
| MongoDB "metadata" collections | ❌ Easy to drift, needs its own admin UI, untyped, another attack surface |
| Model introspects Mongo schema | ❌ Violates the core rule; exposes collections |

A code-first registry means: adding a module or field is a normal code change with
review, tests, and a migration. That is a feature, not friction.

## 2. Core types

```ts
// src/server/modules/types.ts
export type FieldType =
  | "string" | "number" | "integer" | "decimal" | "currency"
  | "boolean" | "date" | "datetime" | "enum" | "id" | "reference";

export type Operator =
  | "eq" | "ne" | "gt" | "gte" | "lt" | "lte"
  | "in" | "nin" | "contains" | "startsWith"
  | "between" | "isNull" | "isNotNull";

export interface FieldDef {
  key: string;                  // logical key exposed to AI, e.g. "orderDate"
  label: string;                // human label, used in prompts and column headers
  type: FieldType;
  path: string;                 // document path, e.g. "orderDate" or "items.sku"
  description?: string;         // one line, included in the prompt
  synonyms?: string[];          // "revenue", "turnover", "sales value"
  filterable?: boolean;         // default true
  sortable?: boolean;           // default true
  groupable?: boolean;          // default false for free text
  selectable?: boolean;         // default true
  operators?: Operator[];       // defaults derived from type
  enumValues?: { value: string; label: string }[];
  format?: ColumnFormat;
  pii?: boolean;                // redact in logs, restrict by role
  roles?: string[];             // field-level visibility; omit = all roles
  unit?: string;                // "currency:INR", "kg", "%"
}

export interface RelationDef {
  key: string;                          // "customer"
  label: string;
  to: string;                           // target module key
  localField: string;                   // "customerId"
  foreignField: string;                 // "_id" in target
  cardinality: "one" | "many";
  // fields that may be pulled through the relation
  expose: { key: string; label: string; field: string; format?: ColumnFormat }[];
}

export interface MetricDef {
  key: string;                          // "revenue"
  label: string;
  agg: "sum" | "avg" | "count" | "min" | "max" | "countDistinct";
  field: string | null;                 // null for count(*)
  description: string;                  // "Sum of confirmed order totals, net of tax"
  synonyms?: string[];
  format?: ColumnFormat;
  roles?: string[];
}

export interface ModuleDef {
  key: string;                          // "sales_orders"
  label: string;                        // "Sales Orders"
  description: string;
  collection: string;                   // INTERNAL ONLY — never sent to the AI
  primaryDateField: string;             // default date filter target
  fields: FieldDef[];
  metrics: MetricDef[];
  relations: RelationDef[];
  defaultSort: { key: string; direction: "asc" | "desc" }[];
  defaultColumns: string[];
  maxLimit: number;                     // per-module cap, ≤ global cap
  roles: string[];                      // who may query this module at all
  tenantField?: string;                 // default "tenantId"
  softDeleteField?: string;             // default "deletedAt"
}
```

Key point: **`collection` is server-side only.** The prompt builder strips it. The
report engine is the only consumer.

## 3. Example module

```ts
export const salesOrders: ModuleDef = {
  key: "sales_orders",
  label: "Sales Orders",
  description: "Confirmed and fulfilled customer orders, one row per order.",
  collection: "sales_orders",
  primaryDateField: "orderDate",
  roles: ["owner", "admin", "manager", "analyst", "viewer"],
  maxLimit: 500,
  defaultColumns: ["orderNumber", "customerName", "orderDate", "totalAmount", "status"],
  defaultSort: [{ key: "orderDate", direction: "desc" }],
  fields: [
    { key: "orderNumber", label: "Order #", type: "string", path: "orderNumber", sortable: true },
    { key: "orderDate", label: "Order Date", type: "date", path: "orderDate" },
    { key: "status", label: "Status", type: "enum", path: "status",
      enumValues: [
        { value: "draft", label: "Draft" },
        { value: "confirmed", label: "Confirmed" },
        { value: "fulfilled", label: "Fulfilled" },
        { value: "cancelled", label: "Cancelled" },
      ] },
    { key: "totalAmount", label: "Total", type: "currency", path: "totalAmount",
      format: "currency", synonyms: ["revenue", "sales value", "turnover", "amount"] },
    { key: "region", label: "Region", type: "string", path: "shippingAddress.region", groupable: true },
    { key: "customerId", label: "Customer", type: "reference", path: "customerId",
      groupable: true, filterable: false },
    { key: "customerName", label: "Customer", type: "string", path: "customerName",
      description: "Denormalized customer name for display and grouping" },
    { key: "salesRep", label: "Sales Rep", type: "string", path: "salesRepName", groupable: true },
  ],
  metrics: [
    { key: "revenue", label: "Revenue", agg: "sum", field: "totalAmount", format: "currency",
      description: "Sum of order totals excluding cancelled orders",
      synonyms: ["sales", "turnover", "income"] },
    { key: "orderCount", label: "Orders", agg: "count", field: null },
    { key: "avgOrderValue", label: "Avg Order Value", agg: "avg", field: "totalAmount", format: "currency" },
  ],
  relations: [
    { key: "customer", label: "Customer", to: "customers",
      localField: "customerId", foreignField: "_id", cardinality: "one",
      expose: [
        { key: "customerSegment", label: "Segment", field: "segment" },
        { key: "customerCity", label: "City", field: "city" },
      ] },
  ],
};
```

Notes on this example:

- `customerName` is **denormalized** onto the order. This is intentional: it avoids a
  join for the most common report and keeps the query fast. The trade-off is that a
  customer rename needs a backfill job — document it.
- `customerId` is `filterable: false` and `groupable: true`. Users group by customer;
  they do not type raw IDs.
- `totalAmount` carries synonyms so "revenue" maps without a custom rule.

## 4. Operator defaults by type

| Type | Default operators |
|---|---|
| string | `eq`, `ne`, `contains`, `startsWith`, `in`, `nin`, `isNull`, `isNotNull` |
| enum | `eq`, `ne`, `in`, `nin`, `isNull`, `isNotNull` |
| number / integer / decimal / currency | `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `between` |
| date / datetime | `eq`, `gt`, `gte`, `lt`, `lte`, `between` |
| boolean | `eq`, `ne` |
| id / reference | `eq`, `in` |

Modules may narrow but **never widen** this list. `contains` on a non-indexed text field
is allowed but flagged as slow (see `report-engine.md`).

## 5. Roles and field-level visibility

Roles for v1: `owner`, `admin`, `manager`, `analyst`, `viewer`.

| Capability | owner | admin | manager | analyst | viewer |
|---|---|---|---|---|---|
| Query modules | ✅ | ✅ | ✅ | ✅ | ✅ |
| Create/save reports | ✅ | ✅ | ✅ | ✅ | ❌ |
| Export | ✅ | ✅ | ✅ | ✅ | ❌ |
| See cost/margin fields | ✅ | ✅ | ✅ | ❌ | ❌ |
| See PII fields | ✅ | ✅ | ✅ | ❌ | ❌ |
| Manage org/members | ✅ | ✅ | ❌ | ❌ | ❌ |

The catalog builder filters modules **and** fields by role before serializing. A field
the role cannot see is not in the prompt and is rejected by validation. This is
defense in depth: the model cannot request what it never saw, and the validator
rejects it if it guesses.

## 6. Prompt serialization

Do not dump JSON. A compact, line-oriented catalog is cheaper and easier for small
models to follow.

```
MODULE: sales_orders — Sales Orders
  desc: Confirmed and fulfilled customer orders, one row per order.
  date: orderDate (default)
  fields:
    orderNumber   string    "Order #"
    orderDate     date
    status        enum      draft|confirmed|fulfilled|cancelled
    totalAmount   currency  aliases: revenue, sales value, turnover
    region        string    groupable
    customerName  string    groupable
    salesRep      string    groupable
  metrics:
    revenue        sum(totalAmount)   aliases: sales, turnover
    orderCount     count(*)
    avgOrderValue  avg(totalAmount)
  related: customer -> customers (segment, city)
```

Rules:

- One line per field, fixed column order: `key type [constraints] [aliases]`.
- Omit `collection`, `path`, roles, and anything internal.
- Include `enum` values inline; they are the most common source of invalid values.
- Keep the whole catalog under ~2,500 tokens for 4 modules. If it grows, split into
  per-module prompts selected by a cheap first-pass classifier, or paginate modules.
- Hash the serialized catalog; store the hash with each conversation turn so prompt
  drift is diagnosable.

## 7. Validation at boot

Run at server start (fail fast):

- unique module keys, field keys, metric keys within a module;
- every `metric.field` and `defaultColumns` entry exists;
- every `relation.to` module exists; every `expose.field` exists on the target;
- every `field.operators` is legal for its type;
- every `path` matches `/^[a-zA-Z0-9_.]+$/` (no `$`, no leading `_` except `_id`);
- `maxLimit ≤ GLOBAL_MAX_LIMIT`;
- `roles` non-empty and drawn from the known role list;
- no field marked both `pii` and `groupable` unless explicitly reviewed.

This is a plain function called from the app bootstrap; it costs milliseconds and
prevents an entire class of runtime errors.

## 8. Adding a module — checklist

1. Create `src/server/modules/<key>.ts` exporting a `ModuleDef`.
2. Add indexes for the fields users will filter/sort on (`tenantId` first).
3. Add the module to `registry.ts`.
4. Add 2–3 golden questions and expected specs to the eval set.
5. Add a tenant-isolation test (org A cannot see org B rows).
6. Run the registry validation test.
7. Update the demo seed data.
8. Only then does the AI "know" about it — no prompt edits required.

## 9. Versioning

- The registry has a monotonically increasing `REGISTRY_VERSION` constant.
- Conversation turns store `registryVersion`; if it changed, do not replay old specs
  blindly — re-validate before executing a saved report.
- Saved reports store a **snapshot of the resolved spec**, not a reference to a
  registry entry. If a field is later removed, execution fails with a clear
  "report uses a field that no longer exists" error rather than silently returning
  wrong data.

## 10. Anti-patterns

| Anti-pattern | Why it hurts |
|---|---|
| Exposing `collection` or `path` to the model | Invites query-shaped output and leaks schema |
| Letting the model pick operators freely | `$where`/regex injection surface; meaningless combos |
| Auto-generating modules from Mongoose schemas | Exposes internal fields, no curation, no synonyms |
| One giant "erp" module | Useless grouping, ambiguous fields, huge prompt |
| Free-text `groupBy` | Hallucinated dimensions |
| Registry in the database with an admin UI | Extra surface, drift, no type safety — defer to v2 |
| Different field sets per endpoint | The AI and the UI must see the same contract |

## 11. References

- MongoDB compound indexes — https://www.mongodb.com/docs/manual/core/index-compound/
- MongoDB data modeling — https://www.mongodb.com/docs/manual/core/data-modeling-introduction/
- Zod — https://zod.dev/
- OWASP LLM Top 10 (LLM05 improper output handling) — https://genai.owasp.org/llm-top-10/
