---
name: kissflow-form-scripting
description: Write Kissflow form scripts with @kissflow/lowcode-client-sdk that read and change the open form through kf.context — get and set field values, read the whole record as JSON, check validation errors, read the form's layout and live field state (visibility, read-only, required), work with child-table rows, and handle files, images and barcodes with kf.client (openFilePicker, uploadFile, getImageUrl, openFilePreview, openScanner). Use for form-event and field-event scripts on Process, Board or Dataform forms, and child-table row scripts.
---

# Kissflow form scripting (`kf.context`)

Kissflow forms (Process, Board and Dataform) can run scripts on form and field
events, such as on load, on field change or on a button click. Inside such a
script:

- `kf` is already defined; do not call `initialize()`.
- `kf.context` is the **form that is open**, as a `Form` object.
- In a script attached to a **child-table row**, `kf.context` is that **row**
  (a `TableForm`). Use `kf.context.getParent()` for the main form.

Every method returns a Promise. Use `await` and `try/catch`.

```js
// Example: when Quantity or Unit_Price changes, compute Total
const data = await kf.context.toJSON();
const total = (Number(data.Quantity) || 0) * (Number(data.Unit_Price) || 0);
await kf.context.updateField({ Total: total });
```

## Methods in this skill

| On | Method | Returns |
|---|---|---|
| Form | `toJSON()` | The whole record |
| Form | `getField(fieldId)` | One field's value |
| Form | `updateField({ fieldId: value, ... })` | `"Updated successfully"` |
| Form | `getValidationErrors()` | Errors by field and table row |
| Form | `getFormConfiguration()` | Layout: sections, fields, permissions |
| Form | `getFieldState()` | Live visibility / read-only / required per field |
| Form | `getTable(tableId)` | A `Table` object (no request) |
| Table | `toJSON()`, `getRows()`, `getRow(rowId)`, `addRow(obj)`, `addRows([obj])`, `deleteRow(rowId)`, `deleteRows([rowIds])`, `getSelectedRows()` | See below |
| Row | `getField`, `updateField`, `toJSON`, `getParent()` | See below |
| Client | `kf.client.openFilePicker`, `uploadFile`, `getImageUrl`, `openFilePreview`, `openScanner` | See below |

## Reading the form

### `toJSON()`: the safest read

```js
const record = await kf.context.toJSON();
// { _id, <fieldId>: value, ..., "Table::<tableId>": [{ _id, <column>: value }] }
```

Use `toJSON()` whenever a field might be empty.

### `getField(fieldId)`

```js
const status = await kf.context.getField("Status");
```

**`getField` never resolves when the field has no value.** Your script hangs
on that `await`. Read possibly-empty fields from `toJSON()` instead:

```js
const { Discount } = await kf.context.toJSON();
if (Discount == null) { /* empty */ }
```

### `getValidationErrors()`

```js
const errors = await kf.context.getValidationErrors();
// Normal shape: { _root: { <fieldId>: ["message", ...] }, <tableId>: { <rowId>: { <fieldId>: ["..."] } } }
```

- If only one top-level key has errors, the SDK returns that key's **inner**
  object. Handle both shapes:

  ```js
  const rootErrors = errors._root ?? errors;
  const hasErrors = Object.keys(errors || {}).length > 0;
  ```

- Hidden fields are validated too. Ignore errors for fields that
  `getFieldState()` reports as `IsHidden`.

### `getFormConfiguration()`

The form's layout and permissions:

```js
const config = await kf.context.getFormConfiguration();
// {
//   formPermission: "Edit" | "ReadOnly",
//   sections: [{
//     Id, Name, Type: "Section" | "Model",   // "Model" = a child table
//     IsHidden, Permission,
//     Fields: [{ Id, Name, Type, Widget, Required, IsHidden, IsReadOnly,
//                Permission: "Editable" | "ReadOnly" | "Hidden", Validations,
//                CurrencyTypes?, LookupColumns?: [{ Id, Name, Type }] }]
//   }],
//   tables: { <tableId>: { rows: { <rowId>: { fields: { <fieldId>: { IsHidden } } } } } },
//   // Process forms only:
//   status, isStartStep, isSendBackToInitiator, activityInstanceId
// }
```

### `getFieldState()`

Live state after formulas and visibility rules have run:

```js
const state = await kf.context.getFieldState();
// {
//   formPermission: "Edit" | "ReadOnly",
//   fields:   { <fieldId>: { Permission, IsHidden, IsReadOnly, Required, Validations } },
//   sections: { <sectionId>: { IsHidden, Permission } },
//   tables:   { ... }
// }
if (state.fields.Discount?.IsReadOnly) { /* don't try to set it */ }
```

Permission names differ by level:

- Form level: `"Edit"` / `"ReadOnly"`.
- Field level: `"Editable"` / `"ReadOnly"` / `"Hidden"`.

## Changing the form: `updateField`

```js
await kf.context.updateField({ Total: 1250, Approved: true, Due_Date: "2026-11-30" });
```

- Values go into the open form. Formulas, visibility rules and the form's
  normal saving all run, as if the user had typed them.
- Several fields in one call are applied together and saved once.
- **Permission rules:**
  - If the whole form is read-only for the user, the call is rejected.
  - Fields that are `Hidden` or `ReadOnly` for the user, at this step, are
    refused. The other fields in the same call are still applied. The promise
    then rejects with `{ isError: true, error: [<refused fieldIds>] }`.
  - Check `getFieldState()` first if a field might be read-only.

### Field value formats

| Field type | Value |
|---|---|
| Text, TextArea | string |
| Number | number |
| Date | `"yyyy-MM-dd"`, e.g. `"2026-11-02"` |
| DateTime | `"2026-11-02T14:30:00+05:30 Asia/Kolkata"` |
| Currency | `"123.45 USD"`; allowed codes in the field's `CurrencyTypes` |
| Boolean | `true` / `false` |
| Dropdown | option string; multi-select: array of option strings |
| Checklist | `[{ value, selected }]` |
| User / MultiUser | `{ _id, Name, Email }` / array of them |
| Lookup / Reference | the referenced record `{ _id, Name, ... }` or `null` |
| Image | one file object `{ key, name, size, photos }` |
| Attachment | array of file objects |
| Signature | file object from `kf.client.uploadFile` |
| Barcode / Scanner widget | string |

## Child tables

```js
const items = kf.context.getTable("Line_Items"); // no request; tableId from the builder

const rows = await items.toJSON();               // [{ _id, <column>: value, ... }]
await items.addRow({ Product: "Laptop", Qty: 1 });
await items.addRows([{ Product: "Mouse", Qty: 2 }, { Product: "Bag", Qty: 1 }]);
await items.deleteRow(rows[0]._id);
await items.deleteRows(rows.filter((r) => !r.Qty).map((r) => r._id));

const selected = await items.getSelectedRows();  // rows the user ticked in the table UI: [{ _id, ... }]
```

Row handles:

```js
const handles = await items.getRows();          // [TableForm, ...] one per row
const row = items.getRow(rows[0]._id);          // TableForm for one row (no request)

const qty = (await row.toJSON()).Qty;           // row as JSON (prefer over getField for possibly-empty cells)
await row.updateField({ Qty: 3 });              // same permission rules as the form
const form = row.getParent();                   // back to the main Form
```

In a row script, `kf.context` is the row:

```js
// Row script: line total = Qty * Price, then refresh the form's Grand_Total
const r = await kf.context.toJSON();
await kf.context.updateField({ Line_Total: (r.Qty || 0) * (r.Price || 0) });

const form = kf.context.getParent();
const all = await form.getTable("Line_Items").toJSON();
await form.updateField({ Grand_Total: all.reduce((s, x) => s + (x.Line_Total || 0), 0) });
```

## Files, images and barcodes (`kf.client`)

| Method | Use | Resolves to | Needs an app page? |
|---|---|---|---|
| `openFilePicker(options)` | Let the user pick and upload files | Array of file objects (`[]` if cancelled) | **Yes** |
| `uploadFile(file, fileName?)` | Upload a `File`/`Blob` you made (e.g. a drawn signature) | One file object, or `null` on failure | No |
| `getImageUrl(fileObject)` | Get a displayable image | `data:image/...;base64,...` string | No |
| `openFilePreview({ files, indexOfFile? })` | Open Kissflow's preview lightbox | Resolves when closed | No |
| `openScanner({ localFileScan? })` | Scan a barcode / QR code (web only) | Decoded string, or `null` if closed | **Yes** |

"Needs an app page" means the script runs inside a Kissflow **app page**.
Inside a form opened outside a page, these fail:

- `openFilePicker` resolves `undefined`.
- `openScanner` rejects.

Treat `undefined` like a cancel.

```js
// Image field: pick one image
const files = await kf.client.openFilePicker({ maxCount: 1, fileExtensions: ["JPG", "PNG"], maxSize: 5 * 1024 * 1024 });
if (files?.length) await kf.context.updateField({ Photo: files[0] });

// Attachment field: append files
const picked = await kf.client.openFilePicker({ maxCount: 10 });
if (picked?.length) {
  const { Documents = [] } = await kf.context.toJSON();
  await kf.context.updateField({ Documents: [...Documents, ...picked] });
}

// Show an Image field's picture (custom UI)
const { Photo } = await kf.context.toJSON();
if (Photo) imgEl.src = await kf.client.getImageUrl(Photo);

// Preview all attachments, starting at the second
await kf.client.openFilePreview({ files: Documents, indexOfFile: 1 });

// Scanner field
const code = await kf.client.openScanner({ localFileScan: true }); // localFileScan: allow scanning an uploaded image
if (code) await kf.context.updateField({ Asset_Tag: code });

// Upload a generated file (e.g. canvas signature)
const blob = await new Promise((res) => canvas.toBlob(res, "image/png"));
const sig = await kf.client.uploadFile(new File([blob], "signature.png", { type: "image/png" }));
if (sig) await kf.context.updateField({ Signature: sig });
```

`openFilePicker` options:

| Option | Meaning |
|---|---|
| `fileExtensions` | Allowed extensions, e.g. `["PDF", "JPG"]` |
| `maxSize` | Maximum file size in bytes |
| `maxCount` | Number of files (1 for Image fields) |
| `imageProps` | `{ sizes: [[w, h], ...], crop: boolean }` |

## Errors

Failures reject with `{ isError: true, message, error }`; show
`err.error || err.message`.

Do not `await` `kf.client.showInfo(...)`: it shows a toast but never resolves.

```js
try {
  await kf.context.updateField({ Total: total });
} catch (err) {
  kf.client.showInfo(`Couldn't update: ${Array.isArray(err.error) ? err.error.join(", ") : err.error || err.message}`);
}
```
