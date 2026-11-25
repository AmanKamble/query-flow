# Export System

> Exports reuse the report executor. There is no separate query path, so a user cannot
> export data they could not see on screen.

## 1. Formats and libraries

| Format | Library | Why | Notes |
|---|---|---|---|
| XLSX | **ExcelJS** | Styling, number formats, autofilter, freeze panes, streaming writer | Pure JS, no native deps |
| CSV | **csv-stringify** | Streaming `Transform`, correct quoting, battle-tested | Part of `node-csv` |
| PDF | **PDFKit** | Pure JS, streams, good control, no browser needed | Manual layout for tables |
| PDF (alt) | Puppeteer + HTML | Best-looking reports, reuse React/CSS | ~300 MB Chromium, slow cold start, sandbox concerns |

**Recommendation:** ExcelJS + csv-stringify + PDFKit. Add Puppeteer only if a
customer explicitly needs pixel-perfect branded PDFs, and run it in a separate worker
with a strict CSP.

## 2. Decision: synchronous vs asynchronous

| Result size | Mode | Why |
|---|---|---|
| ≤ 5,000 rows | Synchronous stream | Under ~2 s; simple UX |
| 5,000–50,000 rows | Synchronous stream with progress | 2–15 s; keep connection open |
| > 50,000 rows | **Async job** + download link | Avoids timeouts and memory spikes |

Start synchronous. Add `export_jobs` only when real data demands it; the schema is
already sketched in `database-and-api-design.md` §3.

Hard caps regardless of mode:

- 50,000 rows per export (configurable per org),
- 200 MB in-memory buffer,
- 120 s wall clock,
- 3 concurrent exports per org.

## 3. API

```http
POST /api/v1/reports/export
Content-Type: application/json

{
  "spec": { ...ReportSpec... },
  "format": "xlsx",              // xlsx | csv | pdf
  "options": {
    "filename": "q3-revenue-by-customer",
    "title": "Q3 2026 Revenue by Customer",
    "includeFilters": true,      // print applied filters in a header/footer
    "includeGeneratedAt": true,
    "timezone": "Asia/Kolkata"
  }
}
```

Response: `200` with the file streamed.

```
Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
Content-Disposition: attachment; filename="q3-revenue-by-customer-20261125.xlsx"
Cache-Control: no-store
X-Export-Rows: 4210
```

Async variant returns `202`:

```json
{ "jobId": "exp_01J...", "status": "queued", "pollUrl": "/api/v1/exports/exp_01J..." }
```

Rules:

- The export route runs the **same** `authorize()` + `validateSpec()` + compiler as the
  UI. No shortcuts.
- The spec comes from the client, but it is re-validated from scratch; it is not trusted
  because "the UI generated it".
- Exports are audited: `action: "report.export"`, format, rowCount, specHash.
- Rate limit: 10/hour, 50/day per user.

## 4. Shared data pipeline

```
spec → validate → authorize → compile → cursor (batch 1,000)
     → row mapper (format values, escape)
     → writer (ExcelJS / csv-stringify / PDFKit)
     → stream to response
```

Design points:

- **One cursor, one pass.** Do not call `countDocuments` for exports unless the user
  asked for a total; it doubles the work on large collections.
- **Batch size 1,000.** Larger batches increase memory; smaller ones add round trips.
- **Backpressure.** `await` the writer's drain before pulling the next batch. Ignoring
  this is the classic cause of a 1 GB heap on a 50k-row export.
- **Format at the edge.** Dates → org timezone; money → fixed decimals; enums → labels.
  The database returns raw values.
- **Streaming, not buffering.** `ReadableStream` in the route handler; never
  `Buffer.concat` the whole file.

## 5. XLSX with ExcelJS

Features worth using:

- Header row: bold, fill color, frozen (`views: [{ state: "frozen", ySplit: 1 }]`).
- Auto filter on the header range.
- Column widths sized from the header + a sample of values (cap at 60 chars).
- Number formats: `#,##0.00` for money, `dd-mmm-yyyy` for dates, `0.0%` for percent.
- Multiple sheets: `Report`, `Filters`, `Metadata` (generated at, user, org, spec hash).
- Totals row for numeric columns (optional, computed from the returned rows only).

```ts
const wb = new ExcelJS.stream.xlsx.WorkbookWriter({ stream, useStyles: true });
const ws = wb.addWorksheet("Report");
ws.columns = columns.map((c) => ({ header: c.label, key: c.key, width: widthFor(c) }));
ws.getRow(1).font = { bold: true };
ws.views = [{ state: "frozen", ySplit: 1 }];
for (const row of rows) ws.addRow(row).commit();
await wb.commit();
```

Use `ExcelJS.stream.xlsx.WorkbookWriter` (not the in-memory writer) for anything above
a few thousand rows.

Gotchas:

- ExcelJS writes strings starting with `=` as strings, but some viewers still interpret
  them. **Escape anyway** (see §7).
- `Decimal128` values must be converted to `number` or `string` before writing.
- Very long text cells: cap at 32,767 chars (Excel limit) and truncate with a marker.

## 6. CSV with csv-stringify

```ts
const stringifier = stringify({
  header: true,
  columns: columns.map((c) => ({ key: c.key, header: c.label })),
  quoted: true,
  record_delimiter: "\r\n",     // Excel-friendly
});
stringifier.pipe(res);
for await (const batch of batches) for (const row of batch) stringifier.write(row);
stringifier.end();
```

- **BOM:** prepend `\uFEFF` so Excel opens UTF-8 correctly (important for ₹, é, etc.).
- `quoted: true` is the safe default; it handles commas, quotes, and newlines.
- Delimiter: comma by default; offer `;` for locales where comma is the decimal
  separator.
- Never build CSV with `join(",")`. It will break on the first address with a comma.

## 7. Formula injection (mandatory)

Any cell whose text starts with `=`, `+`, `-`, `@`, tab (`\t`), or carriage return
(`\r`) can execute as a formula in Excel/LibreOffice/Sheets. This is OWASP "CSV
Injection".

```ts
const RISKY = /^[=+\-@\t\r]/;
export function escapeCell(value: unknown): string {
  if (value === null || value === undefined) return "";
  let s = String(value);
  if (RISKY.test(s)) s = "'" + s;
  if (s.length > 32767) s = s.slice(0, 32764) + "...";
  return s;
}
```

Apply it to **every** string cell in CSV and XLSX. Do not rely on "our data is clean" —
customer names, notes, and imported supplier data are user-controlled.

For PDF, the equivalent concern is HTML injection if you use an HTML renderer; PDFKit's
text API is safe, but escape anything you interpolate into an HTML template.

## 8. PDF with PDFKit

Layout plan:

- **A4 landscape** for wide tables, portrait for summaries.
- Margins 40 pt; header band with title, org name, date range.
- Table: fixed column widths proportional to content; header row repeated on each page.
- Zebra striping for readability; right-align numeric columns.
- Footer: page number, "Generated by QueryFlow on <date>", optional spec hash.
- Fonts: embed a Unicode font (e.g. Noto Sans / Inter) so ₹, €, and non-Latin names
  render. PDFKit's built-in Helvetica is Latin-1 only.
- Cap rows per PDF (e.g. 5,000) — PDFs are for reading, not data transfer.

PDFKit gives you primitives, not a table component. Budget a small internal helper
(`drawTable(doc, columns, rows)`) that handles page breaks and repeated headers. Keep
it ~150 lines; do not pull in a heavy table library for v1.

Puppeteer alternative: render a print-styled HTML page. Better fidelity, worse
operations (Chromium download, memory, sandbox flags). Choose only if branding is a
hard requirement.

## 9. Formatting rules

| Type | XLSX | CSV | PDF |
|---|---|---|---|
| currency | `#,##0.00` + currency code in header | `1234.50` (no symbol) | `₹1,234.50` |
| decimal | `0.00` | `1234.50` | `1,234.50` |
| integer | `0` | `1234` | `1,234` |
| percent | `0.0%` (value 0–1) | `12.3` | `12.3%` |
| date | `dd-mmm-yyyy` | `YYYY-MM-DD` | `25 Nov 2026` |
| datetime | `dd-mmm-yyyy hh:mm` | ISO 8601 | `25 Nov 2026, 14:32` |
| boolean | `Yes`/`No` | `true`/`false` | `Yes`/`No` |
| null | empty | empty | `—` |

Decide once, in `src/server/export/format.ts`, and share it with the UI so screen and
file agree.

## 10. File naming and storage

```
{slug(title)}-{YYYYMMDD-HHmm}-{shortHash}.{ext}
```

- Slugify: lowercase, ASCII, `[a-z0-9-]`, max 60 chars, no path separators.
- Never include raw user input, emails, or IDs.
- Synchronous exports stream; nothing is stored.
- Async exports write to a private directory (`EXPORT_DIR`) or object storage with
  server-side encryption, and expire after 24 h via a cleanup job.
- Downloads are authenticated; verify `tenantId` and `userId` (or `visibility: org`)
  before streaming a stored file.

## 11. Memory and performance budget

| Concern | Target |
|---|---|
| 10k-row XLSX | < 3 s, < 150 MB heap |
| 10k-row CSV | < 1 s, < 50 MB heap |
| 5k-row PDF | < 8 s, < 200 MB heap |
| Concurrent exports | 3 per org, 10 per instance |

If a run exceeds the budget, the fix is usually: use the streaming writer, increase
batch size to 1,000, avoid `countDocuments`, and stop building intermediate arrays.

## 12. Testing

- **Golden files:** 100 rows → compare cell values and formats (not bytes; XLSX zip
  timestamps differ).
- **Injection tests:** cells `=1+1`, `+cmd`, `-2+3`, `@SUM(A1)`, `\t=1+1` must be
  prefixed and inert.
- **Encoding:** `₹`, `€`, `Müller`, `北京` survive CSV (BOM) and PDF (embedded font).
- **Limits:** 50,001 rows is rejected with `EXPORT_TOO_LARGE`.
- **Authorization:** a `viewer` cannot export; a role without `costPrice` cannot export
  it even if the spec asks.
- **Tenant:** exporting with a crafted spec never includes another org's rows.

## 13. References

- ExcelJS — https://github.com/exceljs/exceljs
- ExcelJS streaming writer — https://github.com/exceljs/exceljs#streaming-xlsx-writer
- csv-stringify — https://csv.js.org/stringify/
- csv-stringify options — https://csv.js.org/stringify/options/
- PDFKit — https://pdfkit.org/
- PDFKit text/fonts — https://pdfkit.org/docs/text.html
- OWASP CSV Injection — https://owasp.org/www-community/attacks/CSV_Injection
- Node.js streams/backpressure — https://nodejs.org/api/stream.html
- Next.js Route Handlers (streaming responses) — https://nextjs.org/docs/app/getting-started/route-handlers
