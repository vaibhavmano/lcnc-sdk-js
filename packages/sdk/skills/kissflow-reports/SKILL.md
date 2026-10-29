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

Every response has the report's `Appearance`, the user's saved `Filter`, `Sort`
and `PageSize` (usually `null`), `Style`, and `Data`. The rest depends on the
type. **`Data` rows are keyed by the `Id` of a dimension or measure entry**
(`Column_PClOVjYse3`, `Measure_c844d`), never by `FieldId` or `Name`, so read
the keys from the entries at runtime and never hard-code them.

Each dimension and measure entry describes one key:

| Key | Meaning |
|---|---|
| `Id` | The key in each `Data` row |
| `Name` | The field's label: use it for axis titles, legends and headers |
| `FieldId`, `Type` | The source field and its type (`Select`, `Number`, `User`, `Date`, ...) |
| `ProjectionFunction` | Date part of a date dimension: `DAY`, `MONTH`, `QUARTER`, `YEAR`, `DATE`, or `null` |
| `AggregationFunction` | Measure aggregate: `COUNT`, `SUM`, `AVERAGE`, `MIN`, `MAX`, `VARIANCE`, or `UNIQUE_COUNT` (cards only); `null` on dimensions |
| `CurrencyTypes` | On a Currency measure: the field's currencies |

Dimension values in `Data`:

- An empty value is grouped as `"Other"`.
- A date dimension holds its date part: `DAY` is the day of the month (1–31),
  `MONTH` is 1–12, `QUARTER` is `"Q1"`–`"Q4"`, `YEAR` is the year, and `DATE`
  is text in `MM-DD-YYYY`.
  Format months and dates for display yourself.
- A User dimension holds the user's name in a chart. In a pivot it holds the
  user object as a JSON string, so `JSON.parse` it and show `.Name` (an empty
  one is still the plain text `"Other"`).
- A Process `_status` dimension holds the workflow engine's values
  (`Draft`, `InProgress`, `Completed`, `Rejected`, `Withdrawn`): show labels for them.

#### Chart (bar, column, line, area, pie, doughnut, bubble, ...)

The response has `Values` (the measures) and up to two dimensions, each a
**one-item array**:

- **`Columns[0]`** is the **category**: the x-axis, or the slices of a pie or
  doughnut. It is missing when the chart has no category.
- **`Rows[0]`** is the **legend** (series). It is missing when the chart has no legend.

Without a legend, each `Data` row is one category with one key per measure:

```js
// { Columns: [{ Id: "Column_PClOVjYse3", FieldId: "Dropdown_dd686", Name: "Availability", Type: "Select", ... }],
//   Values:  [{ Id: "Measure_c844d", Name: "Assets", AggregationFunction: "COUNT", ... }],
//   Data: [{ Column_PClOVjYse3: "Assigned", Measure_c844d: 3 },
//          { Column_PClOVjYse3: "Under Repair", Measure_c844d: 1 }] }
const category = result.Columns?.[0];
const legend = result.Rows?.[0];
const measures = result.Values.map((v) => ({ id: v.Id, label: v.Name }));
const labels = category ? result.Data.map((r) => String(r[category.Id])) : [""];
const series = measures.map((m) => ({ name: m.label, data: result.Data.map((r) => r[m.id] ?? 0) }));
// A pie or doughnut has one measure: slice i is labels[i] with value series[0].data[i].
```

**With a legend and exactly one measure**, the server pivots the legend into keys.
Each `Data` row is one category, and every other key is a **legend value**
(as text; an empty legend value is `"Other"`) holding the measure. There is no
legend key and no measure key in the row:

```js
// { Columns: [{ Id: "Column_dept", ... }], Rows: [{ Id: "Column_status", ... }], Values: [{ Id: "Measure_count", ... }],
//   Data: [{ Column_dept: "HR", Approved: 4, Rejected: 1 }, { Column_dept: "Eng", Approved: 9 }] }
const legendValues = [...new Set(result.Data.flatMap((r) => Object.keys(r).filter((k) => k !== category.Id)))];
const stacked = legendValues.map((name) => ({ name, data: result.Data.map((r) => r[name] ?? 0) }));
```

**With a legend and two or more measures**, nothing is pivoted. Each row has
the category key, the legend key and one key per measure, so group by
`r[legend.Id]` yourself.

#### Pivot

The response has `Columns`, `Rows` and `Values` as arrays (any number of dimensions
on each axis, at most one measure), plus `Formatting`, `DrilledMembers` and
`FieldFilters`. Each `Data` row is one cell: one key per row dimension, one per
column dimension, and the measure:

```js
// Data: [{ Column_region: "APAC", Column_quarter: "Q1", Measure_total: 1200 }, ...]
const rowKeys = result.Rows.map((d) => d.Id), colKeys = result.Columns.map((d) => d.Id);
const measure = result.Values[0].Id;
const cell = new Map(result.Data.map((r) => [JSON.stringify([...rowKeys, ...colKeys].map((k) => r[k])), r[measure]]));
```

The server sends **no subtotals or grand totals** and applies no percentage
view; the report's `Appearance` says what to show, and you compute it:

- `Appearance.Total`: `RowTotal`, `ColumnTotal` and `GrandTotal` booleans.
- `Appearance.ValueCalculation`: `KeepAsIs` (or missing), `PercentOfGrandTotal`,
  `PercentOfRow` or `PercentOfColumn`.

A `Formatting` rule (`Measure`, `Operator`, `Value` or `StartValue`/`EndValue`,
`Style` colours) highlights cells; apply it if you show the colours.

#### Card (simple, icon, KPI, comparison)

The response has `Values` and `FieldMapping`, and **`Data` holds at most one row**.
`Data` is `[]` when no item matches, so show the empty state.

- **`Values`** entries are the measures. `ValueType` names each one's slot:
  `Value` (simple and icon cards) or `Actual` and `Target` (KPI and comparison cards).
- **`FieldMapping`** entries are the card's display settings as `{ Name, Value }`
  pairs. `Value` is missing when the setting is empty or `false`. The names are
  `Label`, `Description`, `ActualLabel`, `TargetLabel`, `DisplayActualValue`,
  `DisplayTargetValue`, `ValueFormat` (`NUMBER`, `CURRENCY`, `PERCENTAGE`,
  `CUSTOM`), `DisplayUnit` (`NONE`, `THOUSANDS`, `MILLIONS`, `BILLIONS`),
  `DecimalPoints` (`"none"` or `"0"`–`"5"`; cards created by the AI builder name it
  `DecimalPoint`), `Prefix`, `Suffix`, `Icon`, `IconPosition`, `KpiValueType`,
  `TrendDirection` (`HIGH_IS_GOOD`, `LOW_IS_GOOD`), `TrendPosition` and the trend icons.
- **KPI cards** show a number chosen by `KpiValueType`. Compute it from Actual
  and Target the way Kissflow's own card does. The result is signed, and
  `TrendDirection` says whether a positive one is good:

  | `KpiValueType` | Value |
  |---|---|
  | `ABSOLUTE_DIFFERENCE` | `actual - target` |
  | `PERCENTAGE_OF_TARGET` | `actual / target * 100` |
  | `PERCENTAGE_OF_CHANGE` | `(actual - target) / target * 100` |
  | `PERCENTAGE_OF_DIFFERENCE` | `(actual - target) / ((actual + target) / 2) * 100` |

  Show "-" when either number is missing or the divisor is 0.

```js
const card = result.Data[0];                       // undefined: show the empty state
const slot = (type) => result.Values.find((v) => v.ValueType === type)?.Id;
const setting = (name) => result.FieldMapping?.find((f) => f.Name === name)?.Value;
const actual = card?.[slot("Actual") ?? slot("Value")];
const target = card?.[slot("Target")];
const title = setting("Label");
const decimals = setting("DecimalPoints") ?? setting("DecimalPoint");
```

#### Table

The response has `Id`, `Name`, `Columns` (the visible columns, in the user's
order, each with `Id`, `Name`, `Type`, `Width`) and `Data`. Each row is keyed by
column `Id`, plus `_id` and system keys you can ignore. A child-table column has
its own `Columns`, and its cell holds an **array of row objects** keyed by those
column `Id`s. Page with `page_number` / `page_size`, and get the total from `/count`.

### Report parameters (`FilterParam`)

A report can declare runtime parameters in the builder, such as a region or a
date range. Pass values as keys **starting with `$`**; other keys are ignored.
Send `{}` when the report has none.

```js
body: { FilterParam: { $Region: "APAC", $From: "2026-01-01" } }
```

Use the parameter names exactly as defined in the report builder.

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
