---
name: kissflow-process
description: Write JavaScript that works with Kissflow Process (workflow) items using @kissflow/lowcode-client-sdk — list the user's own requests, their pending tasks, items they participated in and (for process admins) all items; read an item and its progress; create a request, update it, submit or approve it, reject it, withdraw it, delete a draft; admin read/edit; AI-parse a smart attachment. Use when generated custom-component or low-code code works with a Kissflow process. States the role (initiator, assignee, participant, process admin) each method requires and the backend rules that make a call fail.
---

# Kissflow Process

A process is a multi-step workflow, for example Leave Request → Manager Approval
→ HR Approval. Each **item** is one request moving through the steps (called
activities). Code reaches a process through a handle:

```js
// Custom component:  import KFSDK from "@kissflow/lowcode-client-sdk"; const kf = await KFSDK.initialize();
// Low-code script:   `kf` is already defined.
const leave = kf.app.getProcess("Leave_Request_A00"); // process flow id from the builder; no request is sent
```

## Methods in this skill

| Method | What it does | Who can call it |
|---|---|---|
| `getMyItems` | Requests the user raised | Any process user (returns only their own) |
| `getMyTasksItems` | The user's inbox: items waiting on them | Any process user (returns only theirs) |
| `getParticipatedItems` | Items the user acted on earlier, per step | Participant or assignee |
| `getAdminItems` | Every item in the process | **Process admin** |
| `getItem` | One item with all field values | Any process user |
| `getProgress` | Step-by-step progress of an item | Any process user |
| `getAdminItem` | One item, admin view | **Process admin** |
| `createItem` | Start a new request (Draft) | Initiator |
| `updateItem` | Change field values at the current step | User assigned to that step |
| `submitItem` | Submit a draft / approve the current step | Initiator (draft) or assigned user |
| `rejectItem` | Reject at the current step | Assigned user |
| `withdrawItem` | Pull back a running request | The item's initiator |
| `deleteItem` / `discardItem` | Delete a draft | The item's initiator |
| `updateAdminItem` | Edit any item as admin | **Process admin** |
| `parseAttachment` | AI-fill fields from an uploaded document | Initiator, assignee or admin |

Use only these methods. `getParticipatedItems()` without `activityId`,
`sendbackItem`, `reassignItem`, `getReassignees`, `restartItem`, the
`get*Fields` methods, `getFieldOptions` and `openForm` are out of scope; do not
generate code that calls them.

## Roles and permissions

Generated code runs **as the signed-in user**. The backend checks the user's
role on the process and on the specific item. A correct call by the wrong user
fails, so build each screen for one role.

| Role | Who it is |
|---|---|
| **Process user** | Anyone with any permission on the process. |
| **Initiator** | A user allowed to start items (permission `InitiateItems`). For a given item, the user who created it (`_created_by`). |
| **Assignee** | The user the item's **current step** is assigned to. |
| **Participant** | A user who acted on an earlier step of the item. |
| **Process admin** | Permission `Admin` on the process. Admin read calls also accept `ViewProcessAdministration`; `getAdminItems` also accepts `ViewMetrics`; `updateAdminItem` also accepts `ModifyItem`. |

Typical screens:

| Screen | Methods | User |
|---|---|---|
| "My requests" | `getMyItems`, `getItem`, `getProgress`, `withdrawItem`, `deleteItem` | Initiator |
| "New request" | `createItem`, `updateItem`, `submitItem` (and `parseAttachment`) | Initiator |
| "My approvals / inbox" | `getMyTasksItems`, `getItem`, `updateItem`, `submitItem`, `rejectItem` | Assignee |
| "History" | `getParticipatedItems({ activityId })`, `getItem`, `getProgress` | Participant |
| "Admin console" | `getAdminItems`, `getAdminItem`, `updateAdminItem` | Process admin |

Check the user before showing admin screens. `kf.user` is `{ _id, Name, Email, AppRoles }`.
If you cannot tell, call the admin method and show "You don't have access" on failure.

## Item ids

Most calls need two ids, and both come from the item you are acting on:

- `instanceId`: the item id, `item._id`
- `activityInstanceId`: the id of the item's **current step**, `item._activity_instance_id`

Every list result and `createItem` return both. Never build them by hand,
and refresh them after a submit: the next step has a new `_activity_instance_id`.

## Listing items

All list methods take the same options and resolve to `{ Data, Columns, ... }`:

| Option | Default | Meaning |
|---|---|---|
| `pageNumber` | `1` | Page, 1-based |
| `pageSize` | `50` | Items per page |
| `searchValue` | `""` | Free-text search |
| `payload` | `{}` | `{ Columns?: [...], Filters?: {...}, Sort?: [{ Id: "<fieldId>", SortType: "ASC" \| "DESC" }] }` |

- `Data` is the array of items.
- `Columns` describes the columns (`Id`, `Name`, `Type`, ...).
- There is **no total count**. To page, keep requesting until a page returns
  fewer than `pageSize` items. For totals or charts use the `kissflow-reports` skill.

Every item has `_id`, `_activity_instance_id` and `_status`, plus the field
values of the list's columns, keyed by field id. Child tables are not included.

```js
// Requests the user raised.
// status: "draft" | "inprogress" | "completed" | "rejected" | "withdrawn" | "all" (default)
const { Data: myRequests } = await leave.getMyItems({
  status: "inprogress",
  pageSize: 20,
  payload: { Sort: [{ Id: "From_Date", SortType: "DESC" }] },
});

// Inbox: items whose current step is assigned to the user. Optional activityId = one step's id.
const { Data: inbox } = await leave.getMyTasksItems({ activityId: "Manager_Approval" });

// Items the user acted on, for ONE step. activityId is REQUIRED.
const { Data: history } = await leave.getParticipatedItems({ activityId: "Manager_Approval" });

// Process admin only: every item. Items also carry _current_assigned_to.
const { Data: all } = await leave.getAdminItems({ searchValue: "Annual", pageNumber: 2 });
```

Step ids (`activityId`) are the step ids from the process builder, for example
`Manager_Approval`. To show participated items for several steps, call
`getParticipatedItems` once per step and merge the results by `_id`.

## Reading one item

```js
const item = await leave.getItem({
  instanceId: row._id,
  activityInstanceId: row._activity_instance_id,
});
// {
//   _id, _activity_instance_id, _activity_id (current step id), _status,
//   <fieldId>: value, ...,
//   "Table::<tableId>": [{ _id, <column>: value, ... }]   // each child table
// }
```

```js
const progress = await leave.getProgress({ instanceId: row._id });
// {
//   ProcessName, _status, _stage, _progress (0–100), _last_completed_step,
//   Steps: [{ Id, Name, _status, _stage, NodeType, AssignedTo, ActedBy, _activity_instance_id, ... }]
// }
// Use it for a progress bar or a "who has it now" timeline.
```

Process admin only:

```js
const adminItem = await leave.getAdminItem({ instanceId: row._id }); // full item, regardless of assignment
```

## Raising a request (initiator)

```js
// 1. Create a Draft at the first step. Resolves to { _id, _activity_instance_id, ... }.
const draft = await leave.createItem({
  data: { Leave_Type: "Annual", From_Date: "2026-11-02", To_Date: "2026-11-04" },
});

// 2. Optional: change fields while it is a draft (the initiator is assigned to the draft step).
await leave.updateItem({
  instanceId: draft._id,
  activityInstanceId: draft._activity_instance_id,
  data: { Reason: "Family trip" },
});

// 3. Submit it into the workflow.
await leave.submitItem({
  instanceId: draft._id,
  activityInstanceId: draft._activity_instance_id,
});
```

To abandon a draft:

```js
await leave.deleteItem({ instanceId: draft._id }); // discardItem({ instanceId }) does exactly the same
```

Delete works **only while the item is a Draft and only for its initiator**.
There is no admin override; anything else fails.

Field value formats for `data`:

| Field type | Value |
|---|---|
| Text | string |
| Number | number |
| Date | `"yyyy-MM-dd"`, e.g. `"2026-11-02"` |
| DateTime | `"2026-11-02T14:30:00+05:30 Asia/Kolkata"` |
| Currency | `"123.45 USD"` |
| Boolean | `true` / `false` |
| Dropdown | option string; multi-select: array of strings |
| User | `{ _id, Name, Email }`; multi-user: array |
| Lookup / Reference | the referenced record `{ _id, ... }` |

Send only top-level fields in `data`. Child-table rows are read from
`getItem` (`"Table::<tableId>"`); editing them is outside this skill.

## Acting on a task (assignee)

Take the ids from `getMyTasksItems()` results; those items are assigned to the user.

```js
const ids = { instanceId: task._id, activityInstanceId: task._activity_instance_id };

// Optional: fill in fields this step allows
await leave.updateItem({ ...ids, data: { Approved_Days: 3 } });

// Approve / complete the step
const res = await leave.submitItem({ ...ids, comment: "Approved" });
// usually { success: "Submitted successfully" }
// { status: "accepted", request_id } means the step waits on a sub-process. Treat it as submitted.

// Or reject (comment is required)
await leave.rejectItem({ ...ids, comment: "Overlaps with release week" });
```

Backend rules:

| Call | Fails when |
|---|---|
| `updateItem` | The user is not assigned to that step, or the step is not Draft or In progress. Fields that are read-only at this step are not changed. |
| `submitItem` | The user is not assigned, already acted on the step, or the step is not current. For a draft, the user is not allowed to initiate. |
| `rejectItem` | The user is not assigned, already acted, or the step is not In progress. |

After a successful submit or reject, remove the item from the inbox UI, or
reload the list. The item has moved on.

## Withdrawing (initiator)

```js
await leave.withdrawItem({ instanceId: item._id, comment: "No longer needed" }); // comment optional
```

Only the item's **initiator**, and only while the item is **In progress**.
Offer the button only on the user's own in-progress items from `getMyItems({ status: "inprogress" })`.

## Admin edits (process admin)

```js
await leave.updateAdminItem({ instanceId: item._id, data: { Approved_Days: 2 } });
```

- Needs Admin, ViewProcessAdministration or ModifyItem on the process.
- Works on any item regardless of who it is assigned to.
- The backend skips the usual state checks here. Guard the UI yourself: do
  not offer edits on deleted items or drafts.

## Smart attachment parsing (`parseAttachment`)

Fills empty fields of an item from an uploaded document (invoice, ID, ...)
using Kissflow's AI document parser.

Requirements:
- The user is the initiator, the assignee, or a process admin.
- The account has the Smart Attachment feature.
- The process has a Smart Attachment field.

`parseAttachment` works on an item **loaded with `initForm()`**. Use
`initForm` only for this:

```js
// 1. Load the item into a form store (both ids from the item; for a new request, create it first)
const form = await leave.initForm(item._id, item._activity_instance_id);

// 2. Let the user pick the document, then store it in the Smart Attachment field
const [file] = await kf.client.openFilePicker({ maxCount: 1, fileExtensions: ["PDF", "PNG", "JPG"] });
if (!file) return; // user cancelled
await form.updateField({ Invoice_File: [file] });

// 3. Parse and auto-fill
const { appliedFields, suggestedBy } = await leave.parseAttachment({
  instanceId: form.instanceId,
  activityInstanceId: form.activityInstanceId,
  fieldId: "Invoice_File",
  file, // { id, name, key, size?, fileExtension? } from openFilePicker
});
// appliedFields: ids of the fields that were filled; suggestedBy: the file name.
// Show the user which fields were filled so they can review before submitting.
```

- Only **empty** fields are filled. Existing values are kept.
- Parsing runs on the server and can take several seconds; show a spinner.
- A failed parse rejects with the parser's error message.

## Errors

Every method returns a Promise. Failures reject with
`{ isError: true, message, error }`, where `error` is the useful text.
A missing required argument rejects before any request is sent.

```js
try {
  await leave.submitItem({ instanceId, activityInstanceId });
} catch (err) {
  kf.client.showInfo(`Could not submit: ${err.error || err.message}`); // do not await showInfo
}
```

The HTTP status is not passed through, so a permission denial and a
validation error look alike. Prevent most failures by showing each action only
to the role that can perform it, per the tables above.
