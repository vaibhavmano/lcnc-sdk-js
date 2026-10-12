# Kissflow Lowcode SDK: Agent Skills

These skills teach AI coding agents to write correct code against the Kissflow
data models through `@kissflow/lowcode-client-sdk`: **Process**, **Board**
(Case), **Dataform** (Form), **Reports**, and **form scripts**.

Each folder is a self-contained skill in the open
[Agent Skills](https://agentskills.io) format: a `SKILL.md` with YAML front
matter (`name`, `description`) followed by instructions. Any agent that reads
Markdown can use them. Load the skill whose description matches the task and
follow it.

The skills cover **only the APIs selected as required** for generated apps.
Each skill lists its methods up front and names the ones that are out of scope.
Every method states **who can call it**, meaning the role or permission the
Kissflow backend checks for the signed-in user. Generated code runs as the end
user, so a call the user lacks permission for fails at runtime even when the
code is correct. Build each screen for the role that will use it.

## Skills

| Skill | Use it when the code needs to… |
|---|---|
| [`kissflow-process`](kissflow-process/SKILL.md) | List my requests, my tasks, participated items or (admin) all items. Read an item and its progress. Create, update, submit/approve, reject, withdraw, or delete a draft. Admin read/edit. AI-parse a smart attachment. |
| [`kissflow-board`](kissflow-board/SKILL.md) | List board items through a view. Read, create or delete an item. Manage an item's subitems. |
| [`kissflow-dataform`](kissflow-dataform/SKILL.md) | List, read, add (draft → submit), update or delete dataform records, optionally through a form view. Count a view's records. Dropdown and lookup options through a view (`kf.api`). |
| [`kissflow-reports`](kissflow-reports/SKILL.md) | Show charts, pivots, cards or KPI numbers using **server-computed** report results, table counts and totals, and the board item count. |
| [`kissflow-form-scripting`](kissflow-form-scripting/SKILL.md) | Form and field event scripts: read and set fields (`kf.context`), validation, layout and field state, child-table rows, file/image/barcode helpers. |

## Rules shared by all skills

**Getting `kf`.**
- In a custom component (page code), call
  `import KFSDK from "@kissflow/lowcode-client-sdk"; const kf = await KFSDK.initialize();`
- In a low-code or form script, `kf` is already defined.
- Then get a handle with `kf.app.getProcess(id)`, `kf.app.getBoard(id)` or
  `kf.app.getDataform(id)`. Ids are the flow ids from the Kissflow builder.
  Getting a handle sends no request.

**Promises and errors.**
- Every data method returns a Promise. Use `await` inside `try/catch`.
- Failures reject with `{ isError: true, message, error }`; show
  `err.error || err.message`.
- The HTTP status is not included, so a permission denial and a validation
  error look alike.

**Single-key responses are unwrapped.** If the host replies with an object
that has exactly one key, the promise resolves to that key's value. For
example, `deleteItem()` resolves to a confirmation string.

**REST calls via `kf.api`.** Board subitems, form-view counts/options and Reports have no SDK methods.
These skills call backend endpoints with `kf.api(path, { method, headers, body })`.
- `kf.api` **does not reject on HTTP errors**. It resolves with the error body
  `{ error_code, message, status: "error" }`.
- In custom components it resolves `undefined` on network failure or after a
  10-second timeout.
- Those skills include a small `kfRequest` helper that turns these cases into
  exceptions. Use `kf.account._id` as the account id in paths.

**Aggregate on the server.** For counts, sums and chart data use
`kissflow-reports`. Never page through items to compute totals in JavaScript.

**Fire-and-forget calls.** Do not `await` `kf.client.showInfo()`: its promise
never resolves.

## Not covered

The skills deliberately leave these out, either by decision or because they
do not work end to end today. Agents should not generate code that calls them.

| Area | Left out |
|---|---|
| Process | `getParticipatedItems()` without `activityId` (broken), `sendbackItem` (broken), `reassignItem`, `getReassignees`, `restartItem`, `getFields`, `getMyTaskFields`, `getParticipatedFields`, `getMyItemsFields`, `getFieldOptions`, `openForm`, sub-process listing. `initForm` is documented only as the first step of `parseAttachment`. |
| Board | `updateItem`, `submitItem`, `discardItem`, `initForm` (all broken), `getFields`, `getFieldOptions`, `openForm`, `importCSV`, listing views, child flows (subflow children/parent). |
| Dataform / Form View | `getFields`, `getFieldOptions`, `initForm`, `openForm`, `importCSV`. View listing, view info/settings/schema/params, bulk update/delete, upsert, export, CSV import, attachments, remote lookup. View aggregation is **broken** (HTTP 500); use report totals. |
| Reports | Listing a flow's reports, report parameters, board count-by-field, dataform count endpoint, process admin KPI endpoints. |
| Forms | Custom Form scaffolding and its `useForm` / `runAction` template, `watchParams`, custom Form Field widgets, `kf.client.pickLocation`. |

## Keeping this accurate

Method signatures come from `packages/sdk/src`. Host behaviour and permission
requirements were checked against the Kissflow web host (`kf_sdk/*.handler.js`)
and the backend route access-control configs. When a method's behaviour
changes, update the matching skill and the tables above.
