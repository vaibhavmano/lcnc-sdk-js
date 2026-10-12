---
name: kissflow-board
description: Write JavaScript that works with Kissflow Board (Case) items using @kissflow/lowcode-client-sdk — list items through a view with search, sort and paging, read one item with its child tables, create an item, delete an item, and manage an item's subitems (list, read, create, update including state changes, delete) via kf.api. Use when generated custom-component or low-code code works with a Kissflow board. States which board permission (Admin, ViewCaseHome, GetItem, CreateItem, DeleteItem, subitem permissions) each call needs.
---

# Kissflow Board

A board, called "Case" in the backend, tracks items that move between
statuses, such as tickets, tasks or assets. Code reaches it through a handle:

```js
// Custom component:  import KFSDK from "@kissflow/lowcode-client-sdk"; const kf = await KFSDK.initialize();
// Low-code script:   `kf` is already defined.
const tickets = kf.app.getBoard("IT_Helpdesk_A00"); // board flow id from the builder; no request is sent
```

## APIs in this skill

| API | What it does |
|---|---|
| `getItems` (SDK) | List items of a view |
| `getItem` (SDK) | Read one item with its child tables |
| `createItem` (SDK) | Create an item |
| `deleteItem` (SDK) | Delete an item |
| Subitems: list, get, create, update, delete (REST via `kf.api`) | Checklist-style child records of an item |

## Permissions

| Permission | Who has it |
|---|---|
| **Admin** | Board admins. Can do everything. |
| **ViewCaseHome** | Users who can open the board and its views. |
| **GetItem** | Users who can open individual items. |
| **CreateItem** | Users who can raise new items. |
| **DeleteItem** | Users who can delete items. |
| **GetSubitems / GetSubitem / CreateSubitem / UpdateSubitem / DeleteSubitem / MoveSubitem** | Subitem rights, granted per board role. |

| API | Permission needed |
|---|---|
| `getItems({ viewId })` | Board: Admin, GetItem or ViewCaseHome. **And** view: GetItem or ViewCaseHome |
| `getItem` | Admin, GetItem or ViewCaseHome |
| `createItem` | Admin or CreateItem |
| `deleteItem` | Admin or DeleteItem |
| List subitems | Admin, GetSubitems or ViewCaseHome |
| Get one subitem | Admin, GetSubitem or ViewCaseHome |
| Create subitem | Admin or CreateSubitem |
| Update subitem | Admin or UpdateSubitem; changing its state also needs **MoveSubitem** |
| Delete subitem | Admin or DeleteSubitem, **and** the user must be the subitem's creator or a board admin |

Design screens around this:

- A **requester-style** user often has only CreateItem. Give them a "raise
  an item" form, not a list; `getItems` and `getItem` will be denied.
- Show Delete only to Admin/DeleteItem users, and ask for confirmation.

## Listing items

Items are always read through a **view**. Without `viewId` the SDK uses the
board's built-in "All items" view, `<boardId>_all`. Other built-in views have
ids `<boardId>_<suffix>`:

| Suffix | Items |
|---|---|
| `assigned_to_me` | Assigned to the user |
| `created_by_me` | Created by the user |
| `due_this_week` | Due this week |
| `unassigned` | Not assigned |
| `overdue` | Past due |
| `critical` | Critical priority |

For example, `IT_Helpdesk_A00_assigned_to_me`. Custom views have their own
ids, shown in the board builder.

```js
const { Data: items, Columns } = await tickets.getItems({
  viewId: "IT_Helpdesk_A00_assigned_to_me", // optional; default `${boardId}_all`
  pageNumber: 1,     // default 1
  pageSize: 25,      // default 50
  searchValue: "printer",
  payload: {         // optional
    Sort: [{ Id: "Due_Date", SortType: "ASC" }],
    // Columns: [...], Filters: {...}
  },
});
```

The result has:

- **`Data`**: the items on this page.
- **`Columns`**: the view's columns.
- View settings: `Sort`, `Filter`, `PageSize`, `Visualization`, ...

Each item has its field values keyed by field id, plus `_id`, `_status_id`,
`_priority_id`, `_category`, `_created_by`.

There is **no total count** in the list. For a count, use the board count
endpoint in the `kissflow-reports` skill. To page, keep requesting until a page
returns fewer than `pageSize` items.

## Reading one item

```js
const ticket = await tickets.getItem({ instanceId: items[0]._id });
// {
//   _id, _status_id, _priority_id, <fieldId>: value, ...,
//   "Table::<tableId>": [{ _id, <column>: value }]   // each child table
// }
```

## Creating and deleting items

```js
// Create. Resolves to { _id } only. Read it back if you need the full item.
const { _id } = await tickets.createItem({
  data: { Title: "Printer jammed on 3rd floor", Category: "Hardware" },
});
const created = await tickets.getItem({ instanceId: _id });

// Delete permanently. Needs Admin or DeleteItem. Resolves to a confirmation string.
await tickets.deleteItem({ instanceId: _id });
```

Field value formats for `data`:

| Field type | Value |
|---|---|
| Text | string |
| Number | number |
| Date | `"yyyy-MM-dd"` |
| DateTime | `"2026-11-02T14:30:00+05:30 Asia/Kolkata"` |
| Currency | `"123.45 USD"` |
| Boolean | `true` / `false` |
| Dropdown | option string; multi-select: array |
| User | `{ _id, Name, Email }`; multi-user: array |
| Lookup / Reference | the referenced record `{ _id, ... }` |

## Subitems

Subitems are lightweight checklist entries under one board item, such as the
steps of a ticket.

- Every board's subitems have the **same fixed fields**; there are no
  per-board custom subitem fields.
- An item can have at most **100** subitems.

| Field | Meaning |
|---|---|
| `_id` | Subitem id (use in URLs) |
| `_item_id` | Display number, e.g. `ITM-12-03` |
| `_parent_id` | The parent item's `_id` |
| `_subitem_title` | Title |
| `_subitem_description` | Description |
| `_state_id` / `_state_name` | State: `State_Not_Started`, `State_In_Progress`, `State_On_Hold`, `State_Done` |
| `AssignedTo` | `{ _id, Name, ... }` |
| `DueDate`, `_start_date` | ISO-8601 date-times |
| `_subitem_index` | Order (higher = newer) |
| `_created_by`, `_created_at`, `_modified_at` | Audit fields |

Do not send `Title`, `_status_id`, `_priority_id`, `_category` or `Requester`
for a subitem. They are rejected (`KISSFLOW_ERROR_210113`).

The SDK has no subitem methods, so call the backend with `kf.api`. `kf.api`
does not reject on HTTP errors, so use this helper:

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

const subitemsUrl = (boardId, itemId) => `/case/2/${kf.account._id}/${boardId}/${itemId}/subitem`;
```

### List

```js
const base = subitemsUrl("IT_Helpdesk_A00", ticket._id);
const { Subitem, _total_subitem_count, _completed_subitem_count } = await kfRequest(base);
// Subitem: up to 100 subitems, newest first, including completed ones.
const done = Subitem.filter((s) => s._state_id === "State_Done");
```

There are no paging or filter parameters; filter in JavaScript.

### Get one

```js
const sub = await kfRequest(`${base}/${subId}`); // flat subitem object; 404 if it belongs to another item
```

### Create

```js
const { _id: subId, _item_id } = await kfRequest(base, {
  method: "POST",
  body: {
    _subitem_title: "Order replacement toner",
    _subitem_description: "HP 26A, 2 units",
    AssignedTo: { _id: assignee._id },         // must be an active user who can be assigned
    DueDate: "2026-11-05T18:00:00+05:30",      // optional
    // _state_id omitted → "State_Not_Started"
  },
});
```

No field is strictly required, but always send `_subitem_title`. Create fails when:

| Cause | Error code |
|---|---|
| The parent item is archived | `KISSFLOW_ERROR_21069` |
| The item already has 100 subitems | `KISSFLOW_ERROR_210243` |
| The assignee can't be assigned | `KISSFLOW_ERROR_210102` |
| `_start_date` is after `DueDate` | `KISSFLOW_ERROR_210246` |

### Update, including marking done

```js
// Partial update: send only what changes. Empty values clear the field.
await kfRequest(`${base}/${subId}`, { method: "PUT", body: { _subitem_title: "Order toner (3 units)" } });

// Change state (needs UpdateSubitem AND MoveSubitem)
await kfRequest(`${base}/${subId}`, {
  method: "PUT",
  body: { _state_id: "State_Done", _state_name: "Done" },
});
```

A user without MoveSubitem gets a permission error on state changes. Hide
the "mark done" control for them, or handle the error.

### Delete

```js
await kfRequest(`${base}/${subId}`, { method: "DELETE" }); // → { status: "success" }
```

- Only the subitem's **creator** or a **board admin** can delete it, on top
  of the DeleteSubitem permission.
- Deletion is permanent; ask for confirmation.

Subitem writes lock the parent item briefly. Run several writes on the same
item **one after another** (`for … of` with `await`), not in parallel with
`Promise.all`.

## Errors

- SDK methods reject with `{ isError: true, message, error }`; show
  `err.error || err.message`.
- `kfRequest` throws the backend error `{ error_code, message, status: "error" }`;
  show `err.message`.
- Do not `await` `kf.client.showInfo(...)`.

```js
try {
  await kfRequest(`${base}/${subId}`, { method: "PUT", body: { _state_id: "State_Done", _state_name: "Done" } });
} catch (err) {
  kf.client.showInfo(`Couldn't update: ${err.message || err.error}`);
}
```
