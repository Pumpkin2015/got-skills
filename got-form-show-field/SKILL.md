---
name: got-form-show-field
description: Show an already-existing entity field in a GOT form DataGrid. Use when a field exists in the entity table and FormDataSourceField but has no FormItem display control in the form design — only need to add the missing FormItem to the DataGrid. Trigger phrases: "显示字段", "show field", "字段显示出来", "form中没有显示", "DataGrid缺列", or when working in GOT Designer and a table field is missing from the grid display.
---

# GOT Form Show Field

Add a missing FormItem to a DataGrid for an entity field that already exists.


## CRITICAL: Git commit before and after

Before ANY edit to GOT XML files, commit the current state. After confirming the edit works, commit again. This ensures rollback safety:

```powershell
C:\Git\cmd\git.exe -C D:\Gongqi\PJ\ZongYu\Project\Server add <changed-files>
C:\Git\cmd\git.exe -C D:\Gongqi\PJ\ZongYu\Project\Server commit -m "<description>"
```

If something goes wrong, revert with:
```powershell
C:\Git\cmd\git.exe -C D:\Gongqi\PJ\ZongYu\Project\Server checkout -- <files>
```
## Workflow

### 1. Locate the form file

Search for the form ID (e.g., `40400016`) in `GongqiERP/WEB-INF/`:

```powershell
cd <project>/Project/Server
Get-ChildItem -Path GongqiERP/WEB-INF/ -Recurse -Include *.xml |
  Select-String -Pattern "<form-id>" | Select-Object -First 5
```

If multiple forms match, confirm with user which one.

### 2. Find the entity field

Search for the field's Chinese label in the entity table XML to get `name` and `id`:

```powershell
Select-String -Path "<table-xml>" -Pattern "<label>"
```

Read context around the match to extract:
- `TableField name="Xxx"` — the field name
- `id="NNNNN"` — the entity field's innerId (used as `refinnerid`)
- `type="String|Decimal|..."` — data type

### 3. Check DataSource field

Search the form XML for the field name:

```powershell
Select-String -Path "<form-xml>" -Pattern "<fieldName>"
```

Confirm the `FormDataSourceField` exists and has `Visible=true`. If missing, this skill does not apply — use `got-field-add` instead.

### 4. Check FormItem

Search for `refinnerid="<entity-field-id>"` in the form XML. If only found in `FormDataSources` section and NOT in `FormDesign`, the UI control is missing.

### 5. Pick a template column

Find a column in the DataGrid with the same type (StringEditor/DateEditor/NumberEditor):

```powershell
Select-String -Path "<form-xml>" -Pattern "FormItem name=" | Select LineNumber, Line
```

Read a nearby StringEditor FormItem for the full property template.

### 6. Determine nextInnerId

Read the root `<Form>` element's `nextInnerId` value. Use it as the new FormItem's `id`, then bump `nextInnerId` by 1.

### 7. Insert FormItem

Insert the new FormItem inside the DataGrid, right after the first column (Status) or at a logical position. The key binding properties:

```xml
<Property innerid="40024">
  <Name>DataSource</Name>
  <Value>MasterScheduleTable</Value>
</Property>
<Property refentityid="<entity-id>" refinnerid="<entity-field-innerId>">
  <Name>Extends</Name>
  <Value><FieldName></Value>
</Property>
```

Use `apply_patch` for precise line-based insertion. Example:

```
@@
               </FormItem>
+              <FormItem name="MasterScheduleTable_WeekId" id="60225" createdLayer="ext" modifiedLayer="ext" type="StringEditor">
+                ...full Properties...
+              </FormItem>
               <FormItem name="MasterScheduleTable_CreatedDate" ...
```

Also update `nextInnerId` on the root `<Form>` element.

### 8. Verify

```powershell
Select-String -Path "<form-xml>" -Pattern "<fieldName>|nextInnerId"
```

Check that:
- new FormItem appears with correct DataSource and Extends bindings
- nextInnerId is bumped
- FormDataSourceField is unchanged
