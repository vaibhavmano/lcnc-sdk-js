---
name: kissflow-reports
description: Get pre-computed Kissflow report results for Process, Board (Case) and Dataform reports — chart, pivot, card and table data, table row counts and column totals — plus the board item count, to feed chart, card and KPI widgets in custom pages. Use instead of fetching items and aggregating in JavaScript. Covers the kf.api calls, request bodies, response shapes per report type, and who can read a report.
---

# Kissflow Reports

Kissflow computes reports **on the server**. A chart, pivot, KPI card or
summary table on a custom page should ask the backend for the computed result.
Fetching items and summing them in the browser is slow, limited by paging,
and costly. Do not do it.

## APIs in this skill

| API | Use it for |
|---|---|
| `POST /{svc}/2/{acc}/{flowId}/{reportId}` | Computed data of a saved report: chart, pivot, card or table |
| `POST /{svc}/2/{acc}/{flowId}/{reportId}/count` | Row count of a table report |
| `POST /{svc}/2/{acc}/{flowId}/{reportId}/aggregation` | Column totals (sum, avg, min, max, counts) of a table report |
| `GET  /case/2/{acc}/{boardId}/count` | Number of items in a board (or a board view) |

`{svc}` depends on the flow type:

| Flow | `{svc}` | `{flowId}` |
|---|---|---|
| Process | `process-report` | process id |
| Board | `case-report` | board id |
| Dataform | `form-report` | dataform id |

Listing a flow's reports, report-parameter discovery, board count-by-field,
dataform count and process admin KPI endpoints are out of scope.

**Report ids are fixed at build time.** Take `{reportId}` from the report in
the Kissflow builder (report settings / URL) and hard-code it with the flow id.

## Calling the backend: `kfRequest` helper

The SDK has no report methods. Call the backend with `kf.api`, which sends an
authenticated same-origin request. It **does not reject on HTTP errors**, so
copy this helper into the generated code:

```js
// kf: `await KFSDK.initialize()` in a custom component, or the global `kf` in a low-code script
async function kfRequest(path, { method = "GET", query, body } = {}) {
  const qs = query ? `?${new URLSearchParams(query)}` : "";
  const res = await kf.api(path + qs, {
    method,
    headers: { "Content-Type": "application/json" },
    ...(body !== undefined ? { body: JSON.stringify(body) } : {}),
  });
  if (res === undefined) throw new Error("Request failed or timed out"); // custom components time out after 10 s
  if (typeof Response !== "undefined" && res instanceof Response) {
    throw new Error(`Unexpected non-JSON response (${res.status})`);
  }
  if (res && res.status === "error") throw res; // backend error: { error_code, message, status: "error" }
  return res;
}
const ACC = kf.account._id;
```

## Who can read a report

A user can read a saved report's data, count and totals if **either** holds:

- the report is **shared with them** (Admin, Share or View on the report), or
- they **manage reports on the flow**:

  | Flow | Permissions |
  |---|---|
  | Process | Admin, ManageReports, EditReports, ShareReports, DeleteReports, CreateReports or ViewAllReports |
  | Board | Admin or CreateReport |
  | Dataform | Admin or Edit |

In accounts with apps, the app's Admins and Editors can read reports too.

- **Process only:** each report type (table, chart, pivot, card) must also be
  enabled for the account, or the call fails.
- **Share every report the page uses with the roles that open the page.**
  Otherwise those users get an error for that widget.
- Load each widget independently, so one denied report doesn't blank the
  page. See [Rendering guidance](#rendering-guidance).

The board count endpoint has its own permission: Admin, ViewCaseHome or
ManageCustomFilters on the board.

## 1. Report data

One endpoint serves every report type; the server picks the computation from
the report's type.

```js
const result = await kfRequest(`/process-report/2/${ACC}/Leave_Request_A00/${REPORT_ID}`, {
  method: "POST",
  query: { apply_preference: true }, // optional: apply the user's saved filter/sort for this report
  body: { FilterParam: {} },         // see "Report parameters" below
});
```

Query parameters:

| Param | Applies to | Meaning |
|---|---|---|
| `page_number`, `page_size` | Table reports only | Paging. Charts, pivots and cards ignore them. |
| `q` | Table reports | Search text |
| `apply_preference` | All | `true` applies the user's saved filter and sort |

### Response by report type

Every response has `Data` plus the report's `Appearance`, `Filter`, `Sort`,
`PageSize` and `Style`. The rest depends on the type:

**Chart (bar, column, line, pie, …)**: `Rows` holds the dimensions (x-axis /
slices), `Columns` the optional series dimension, `Values` the measures. Each
`Data` row has one key per dimension and per measure, named by their `Id`:

```js
// { Rows: [{ Id: "Department", Name: "Department", ... }],
//   Values: [{ Id: "Days_Sum", Name: "Total days", AggFunction: "Sum", ... }],
//   Data: [{ Department: "HR", Days_Sum: 42 }, { Department: "Eng", Days_Sum: 97 }] }
const dim = result.Rows[0].Id;
const measures = result.Values.map((v) => ({ id: v.Id, label: v.Name }));
const labels = result.Data.map((r) => r[dim]);
const series = measures.map((m) => ({ name: m.label, data: result.Data.map((r) => r[m.id] ?? 0) }));
```

Read the key names from `Rows`, `Columns` and `Values` at runtime; never
hard-code them. When `Columns` is non-empty, also group by `r[result.Columns[0].Id]`.

**Pivot**: same `Rows`, `Columns` and `Values` plus `Formatting`. Build the
grid from row dimension × column dimension, with measure values in the cells.

**Card (simple, KPI, icon, comparison)**: `Values` and `FieldMapping`;
`Data[0]` holds one key per value `Id`. KPI cards also carry computed keys
such as `ABSOLUTE_DIFFERENCE` and `PERCENTAGE_OF_TARGET`:

```js
// { Values: [{ Id: "Value_1", Name: "Closed" }, { Id: "Value_2", Name: "Target" }],
//   Data: [{ Value_1: 1200, Value_2: 1500, PERCENTAGE_OF_TARGET: 80 }] }
const card = result.Data[0] ?? {};
const main = card[result.Values[0].Id];
```

**Table**: `Columns` (`Id`, `Name`, `Type`) and `Data` rows keyed by column
`Id`. Page with `page_number` / `page_size`, and get the total from `/count`.

### Report parameters (`FilterParam`)

A report can declare runtime parameters in the builder, such as a region or a
date range. Pass values as keys **starting with `$`**; other keys are ignored.
Send `{}` when the report has none.

```js
body: { FilterParam: { $Region: "APAC", $From: "2026-01-01" } }
```

Use the parameter names exactly as defined in the report builder.

Don't rely on a `ComponentFilter` body key. It may be ignored by the current
routing, so use report parameters instead.

## 2. Table row count

```js
const { Count } = await kfRequest(`/case-report/2/${ACC}/IT_Helpdesk_A00/${REPORT_ID}/count`, {
  method: "POST",
  body: { FilterParam: {} },
  query: { q: "printer" }, // optional, same search as the data call
});
// → { Count: 128 }. Use with page_size for "Page 3 of 6".
```

## 3. Table column totals

```js
const totals = await kfRequest(`/form-report/2/${ACC}/Employee_Master_A00/${REPORT_ID}/aggregation`, {
  method: "POST",
  body: { FilterParam: {} },
});
// {
//   "<textColumnId>":   { Count: 10, Blank: 2, NonBlank: 8, CountUnique: 7 },
//   "<numberColumnId>": { Sum: 52000, Min: 1200, Max: 9800, Average: 5200 }
// }
// Date columns return Min/Max as strings.
```

Totals cover **all rows**. Pass `page_number` / `page_size` only if you want
the totals of one page.

## 4. Board item count

The number of items on a board, or in one of its views. Use it for KPI tiles
such as "Open tickets: 42".

```js
const { Count } = await kfRequest(`/case/2/${ACC}/IT_Helpdesk_A00/count`, {
  query: {
    visualization: "List",                // always send this, to get a single number
    view_id: "IT_Helpdesk_A00_overdue",   // optional: count only that view's items (its filter applies)
    q: "printer",                         // optional search
  },
});
```

- **Always send `visualization=List`.** For views shown as Kanban or Matrix
  the response is a per-group map (`{ "<status_id>": 5, ... }`) instead of
  `{ Count }`.
- Counts exclude deleted items and drafts. Archived items are included unless
  the view filters them out.
- Needs Admin, ViewCaseHome or ManageCustomFilters on the board.

## Rendering guidance

- Load each widget independently and in parallel with
  `Promise.allSettled`. Show an empty state when `Data` is `[]`, and a short
  "Not available" message when a widget's call fails.
- Show a loading placeholder per widget. Report calls can take a few seconds
  on large flows.
- Refresh on a user action or a page parameter change, not on a timer.
- If no report or count fits a widget, ask the app builder to create a report
  in Kissflow. Do not page through items to compute it.
