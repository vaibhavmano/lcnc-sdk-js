---
name: kissflow-dataform
description: Write JavaScript that reads and writes Kissflow Dataform (Form) records with @kissflow/lowcode-client-sdk — list, search, sort and page records (optionally through a form view), read one record with its child tables, add a record (draft → submit), update, delete, discard the user's draft, count a view's records, and fetch dropdown and lookup options through a view. Use when generated custom-component or low-code code works with a Kissflow dataform. States which dataform/view permission (Admin, Edit, ReadOnly, InitiateItems, Delete) each method needs.
---

# Kissflow Dataform

A dataform is a table of records, such as Employees, Products or Vendors.
Code reaches it through a handle:

```js
// Custom component:  import KFSDK from "@kissflow/lowcode-client-sdk"; const kf = await KFSDK.initialize();
// Low-code script:   `kf` is already defined.
const employees = kf.app.getDataform("Employee_Master_A00"); // dataform flow id from the builder; no request is sent
```

## Methods in this skill

| Method | What it does |
|---|---|
| `getItems` | List, search, sort and page records |
| `getItem` | Read one record, with its child tables |
| `createItem` | Create (or replace) the user's draft |
| `submitItem` | Turn the draft into a saved record |
| `updateItem` | Change a draft or a saved record |
| `discardItem` | Delete the user's draft |
| `deleteItem` | Delete a saved record |
| View count (REST) | Number of records in a view |
| Dropdown options (REST) | Choices for a dropdown field, including child-table fields, through a view |
| Lookup options (REST) | Records to pick for a Lookup/Reference/User field, including child-table fields, through a view |

All SDK methods accept an optional `viewId`; the REST endpoints always go
through a view.

## Permissions

The backend checks the signed-in user's permission on the dataform. When a
`viewId` is passed, it **also** checks permission on that view.

| Permission | Who has it | Can |
|---|---|---|
| **Admin** | Dataform admins | Everything |
| **Edit** | Users shared with edit access | Read, add, update, submit |
| **ReadOnly** | Users shared with view access | Read only |
| **InitiateItems** | Users allowed to add records | Add, update, submit. **Cannot list or read** unless they also have Edit or ReadOnly |
| **Delete** | Users allowed to delete | Delete saved records |

| Method | Dataform permission | View permission (only when `viewId` is passed) |
|---|---|---|
| `getItems` | Admin, Edit or ReadOnly | Admin, Edit or ReadOnly |
| `getItem` | Admin, Edit or ReadOnly | Admin, Share or ReadOnly |
| `createItem` | Admin, Edit or InitiateItems | Admin or Edit |
| `submitItem` | Admin, Edit or InitiateItems | Admin or Edit |
| `updateItem` | Admin, Edit or InitiateItems | Admin or Edit |
| `discardItem` | Admin, Edit or InitiateItems | Admin or Edit |
| `deleteItem` | Admin or Delete | Admin or Delete |

Design screens around this:

- A **read-only user** (ReadOnly) gets lists and detail pages, with no add,
  edit or delete buttons.
- An **"add only" user** (InitiateItems only) gets a create-and-submit form and
  no list, because `getItems` and `getItem` are denied.
- Show **Delete** only to users with Admin or Delete.

### Views (`viewId`)

A view is a saved subset of a dataform: some records, some columns, and its
own field permissions.

- When the app is built around a view, pass the **same `viewId` to every
  call**. The backend then applies the view's filter, columns and permissions.
- Without `viewId`, calls work on the whole dataform with dataform-level
  permissions.
- Take view ids from the dataform's builder (Views tab).

## Listing records

```js
const { Data: rows, count } = await employees.getItems({
  viewId: "Active_Employees", // optional
  pageNumber: 1,              // default 1
  pageSize: 25,               // default 50
  searchValue: "asha",        // optional free-text search
  payload: {                  // optional
    Sort: [{ Id: "Joining_Date", SortType: "DESC" }],
    // Columns: [...]  limit the returned columns
    // Filters: {...}  filter expression
  },
});
```

The result is `{ Data, count, Columns, Sort, Filter }`:

- `Data` is the records on this page.
- `count` is the **total** number of matching records. Use it for "Page 2 of
  8" and for a record count, with no extra call.
- Each record has its field values keyed by field id, plus `_id`,
  `_created_by`, `_created_at`, `_modified_at`.
- Child tables are **not** included in list results.

```js
// Simple pager
const pageSize = 25;
const totalPages = Math.max(1, Math.ceil(count / pageSize));
```

## Reading one record

```js
const emp = await employees.getItem({ itemId: rows[0]._id /*, viewId */ });
// {
//   _id, <fieldId>: value, ...,
//   "Table::<tableId>": [{ _id, <column>: value, ... }]   // each child table
// }

const skills = emp["Table::Skills"] ?? []; // child table rows; missing when empty or hidden in the view
```

## Adding a record: draft → submit

A new record is created in two steps.

- Each user has **one draft per dataform**. `createItem` creates that draft, or
  replaces it if one exists.
- `submitItem` turns the draft into a saved record **with a new `_id`**.

```js
// 1. Create the user's draft. Resolves to the draft; its _id looks like "draft_<userId>".
const draft = await employees.createItem({
  data: { Name: "Asha Rao", Department: "Finance", Joining_Date: "2026-11-02" },
  // viewId: "New_Joiners",
});

// 2. Optional: more changes while it is a draft
await employees.updateItem({ itemId: draft._id, data: { Department: "Operations" } });

// 3. Submit. Resolves to the saved record. Use THIS _id from now on, not draft._id.
const record = await employees.submitItem({ itemId: draft._id, data: { Status: "Active" } });
console.log(record._id);
```

To abandon instead of submitting:

```js
await employees.discardItem(); // deletes the signed-in user's draft; pass { viewId } if you used one
```

Rules:

- Do not call `createItem` twice in parallel. The second call replaces the
  first draft.
- `submitItem` is only for drafts. A saved record is changed with `updateItem`.
- If submit fails (for example a required field is empty), the promise
  rejects. Show the message and keep the draft so the user can fix it and
  submit again.

## Updating and deleting saved records

```js
// Change fields of a saved record. Resolves to the updated record.
const updated = await employees.updateItem({
  itemId: record._id,
  data: { Department: "Operations", Salary: 85000 },
  // viewId: "Active_Employees",
});

// Delete a saved record (Admin or Delete). Resolves to a confirmation string.
await employees.deleteItem({ itemId: record._id /*, viewId */ });
```

- Send only the fields you change in `data`.
- Fields that are read-only for the user (or in the view) are refused.
- Ask for confirmation in the UI before `deleteItem`; deletion is permanent.

## Form views: counts, dropdowns and lookups (REST via `kf.api`)

These view-scoped endpoints have no SDK method. Call them with `kf.api`,
which **does not reject on HTTP errors**, through this helper:

```js
async function kfRequest(path, { method = "GET", query, body } = {}) {
  const qs = query ? `?${new URLSearchParams(query)}` : "";
  const res = await kf.api(path + qs, {
    method,
    headers: { "Content-Type": "application/json" },
    ...(body !== undefined ? { body: JSON.stringify(body) } : {}),
  });
  if (res === undefined) throw new Error("Request failed or timed out"); // custom components time out after 10 s
  if (typeof Response !== "undefined" && res instanceof Response) throw new Error(`Unexpected response (${res.status})`);
  if (res && res.status === "error") throw res; // { error_code, message, status: "error" }
  return res;
}

const viewUrl = (dataformId, viewId) => `/form/2/${kf.account._id}/${dataformId}/view/${viewId}`;
```

| Endpoint | Permission (dataform / view) |
|---|---|
| View count | Admin, Edit or ReadOnly / Admin, Edit or ReadOnly |
| Dropdown options (field, child-table field) | Admin, Edit or ReadOnly / Admin, Edit or ReadOnly |
| Lookup options (field) | Any signed-in user |
| Lookup options (child-table field) | View on the dataform / Admin, Edit or ReadOnly on the view |

### Record count of a view

Use it for KPI tiles ("Active employees: 214"). It works for ReadOnly users.

```js
const { count } = await kfRequest(`${viewUrl("Employee_Master_A00", "Active_Employees")}/count`, {
  query: { q: "" }, // optional search text
});
```

- Counts the records the view shows: the view's saved filter applies, and
  drafts and deleted records are excluded.
- Narrow it with the `q` search parameter.
- Response: `{ count: <number> }`.

### Dropdown options

Choices for a dropdown, multi-select or checkbox field of a record, as the
view allows them:

```js
const base = viewUrl("Employee_Master_A00", "Active_Employees");

// Top-level field
const departments = await kfRequest(`${base}/${record._id}/Department/dropdown`, {
  query: { q: "", page_number: 1, page_size: 250 },
});
// → ["Finance", "Operations", "Sales"]

// Field inside a child-table row
const levels = await kfRequest(`${base}/${record._id}/Skills/${row._id}/Level/dropdown`, {
  query: { page_number: 1, page_size: 250 },
});
```

- The response is an **array of strings** (the option values).
- An empty array means there are no options or the user may not see them.
- Options sourced from another dataform or dataset honour `q`,
  `page_number` and `page_size` (default page size is 10, so pass 250 for a
  full list). Fixed option lists return everything and ignore the parameters.
- **Dependent dropdowns** (options filtered by another field) use the
  record's **saved** values. Save the controlling field with `updateItem`
  first, then fetch the options.
- For a new record, use the draft's `_id` (from `createItem`) as the record id.

### Lookup options (Lookup, Reference, User fields)

Records the user can pick for a lookup-style field:

```js
// Top-level field
const { Data: vendors, Columns } = await kfRequest(`${base}/${record._id}/Vendor/lookup`, {
  query: { q: "acme", page_number: 1, page_size: 20 },
});
// Data:    [{ _id: "Pk123", Name: "Acme Corp", Email: "ap@acme.com", ... }]
// Columns: the lookup's display columns ({ Id, Name, Type, ... })

// Field inside a child-table row
const { Data: products } = await kfRequest(`${base}/${record._id}/Line_Items/${row._id}/Product/lookup`, {
  query: { q: "", page_number: 1, page_size: 20 },
});
```

- `q` searches the lookup's visible columns. The default `page_size` is 10.
  Request more pages as the user scrolls.
- To set the field, pass the chosen record as the value:
  `updateItem({ itemId, data: { Vendor: vendors[0] } })`.
- Nothing available: `{ Columns: [], Data: [] }`. Denied: `{ Data: [] }`.
- The same "saved values" rule as dropdowns applies to dependent lookups.
  Draft ids work for new records.

## Field value formats for `data`

| Field type | Value |
|---|---|
| Text, TextArea | string |
| Number | number |
| Date | `"yyyy-MM-dd"`, e.g. `"2026-11-02"` |
| DateTime | `"2026-11-02T14:30:00+05:30 Asia/Kolkata"` |
| Currency | `"123.45 USD"` |
| Boolean | `true` / `false` |
| Dropdown | option string; multi-select: array of strings |
| User | `{ _id, Name, Email }`; multi-user: array |
| Lookup / Reference | the referenced record `{ _id, ... }` |

Send top-level fields in `data`. Child-table rows are returned by `getItem`
under `"Table::<tableId>"`.

## Errors

Every method returns a Promise. Failures reject with
`{ isError: true, message, error }`, where `error` is the useful text.
A missing `itemId` rejects before any request is sent.

```js
try {
  await employees.updateItem({ itemId, data });
} catch (err) {
  kf.client.showInfo(`Could not save: ${err.error || err.message}`); // do not await showInfo
}
```

The HTTP status is not passed through, so a permission denial and a validation
error look alike. Prevent most failures by showing each action only to users
whose permission allows it, per the tables above.
