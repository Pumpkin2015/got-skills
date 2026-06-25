---
name: got-field-full
description: Add a new field to a GOT entity and display it in the form DataGrid in one pass. Covers MySQL column, entity TableField, Java Base getter/setter/cast/const, JDT compile, FormDataSourceField, FormItem, and git safety workflow. Use when adding a new field that needs both database and UI display. Trigger phrases: "新增字段", "add field", "加字段", "新加一列", "表里加字段并显示", "加字段一步到位".
---

# GOT Field Full — 新增字段一步到位

Add a new field from database through to form display in one pass.

## CRITICAL: Git safety

```powershell
# Before ANY edit:
C:\Git\cmd\git.exe -C D:\Gongqi\PJ\ZongYu\Project\Server add -A
C:\Git\cmd\git.exe -C D:\Gongqi\PJ\ZongYu\Project\Server commit -m "<description>"

# After each successful step, commit again.
# If anything breaks: git checkout -- <files>
```

## Workflow (6 layers)

### Layer 1: MySQL

```powershell
D:\Gongqi\PJ\ZongYu\MySQL\bin\mysql.exe -h 127.0.0.1 -P 40000 -u root -pg-o-n-g-q-i gongqi -e "ALTER TABLE gongqi_df_<module>`$<TableName> ADD COLUMN <FieldName> VARCHAR(20) DEFAULT NULL;"
```

**Note**: `$` in table names must be escaped as `` `$ `` in PowerShell.

### Layer 2: Entity Table XML

File: `GongqiERP\WEB-INF\layers\ext\gongqi.df.<module>\model\got\datadictionary\tables\gongqi.df.<module>.<TableName>.xml`

1. Read `nextInnerId` from `<Table>` root element
2. Use it as the new `TableField`'s `id`, then bump `nextInnerId` by 1
3. Insert after a similar field (e.g., WeekId)

TableField template (String type):
```xml
<TableField name="YearId" id="60062" createdLayer="ext" modifiedLayer="ext" type="String">
  <Properties>
    <Property><Name>Extends</Name><Value/></Property>
    <Property><Name>Label</Name><Value>年</Value></Property>
    <Property><Name>HelpText</Name><Value/></Property>
    <Property><Name>DisplayLength</Name><Value>20</Value></Property>
    <Property><Name>AllowEdit</Name><Value>Always</Value></Property>
    <Property><Name>Default</Name><Value/></Property>
    <Property><Name>StringSize</Name><Value>20</Value></Property>
    <Property><Name>Regular</Name><Value/></Property>
    <Property><Name>FilterField</Name><Value/></Property>
    <Property><Name>NotNull</Name><Value>false</Value></Property>
    <Property><Name>ChangeCase</Name><Value>None</Value></Property>
    <Property><Name>Trim</Name><Value>LeftRight</Value></Property>
  </Properties>
</TableField>
```

### Layer 3: Java Base Class

File: `LayerProject\ext_gongqi.df.<module>\gongqi\df\<module>\layers\ext\tables\base\Base<TableName>.java`

Add four things after a similar field (e.g., WeekId):

**Field declaration:**
```java
private String yearId;
```

**Getter:**
```java
public String getYearId() { return yearId; }
```

**Setter** (note: use `doFormat_String` for String, `doFormat_Decimal` for BigDecimal, `doFormat_Date` for Date):
```java
public void setYearId(String yearId) {
    this.yearId = this.doFormat_String(_YearId, yearId);
}
```

**Cast line** (in `cast()` method):
```java
this.setYearId(gongqi.erp.gotmodel.table.GongqiRecordHelper.castString(map, "yearId"));
```

**Constant:**
```java
final public static String _YearId = "YearId";
```

**⚠️ After editing the .java file**, remove any BOM and escaped quotes:
```powershell
$bytes = [System.IO.File]::ReadAllBytes($javaFile)
if ($bytes[0] -eq 0xEF) { $newBytes = $bytes[3..$bytes.Length]; [System.IO.File]::WriteAllBytes($javaFile, $newBytes) }
(Get-Content $javaFile -Raw) -replace '\\"', '"' | Set-Content $javaFile -NoNewline
```

### Layer 4: Compile

Eclipse JDT auto-compiles on project refresh. Verify `.class` appeared:
```powershell
Get-ChildItem "GongqiERP\WEB-INF\layers\ext\gongqi.df.<module>\binary\classes" -Recurse -Filter "Base<TableName>.class"
```

If JDT didn't auto-build, compile manually:
```powershell
$src = "LayerProject\ext_...\base\Base<TableName>.java"
$dest = "GongqiERP\WEB-INF\layers\ext\...\binary\classes"
$cp = "GongqiERP\WEB-INF\lib\*;$dest;GongqiERP\WEB-INF\layers\app\...\binary\classes"
D:\Gongqi\PJ\ZongYu\Java\jdk\bin\javac.exe -encoding UTF8 -cp $cp -d $dest $src
```

### Layer 5: Form DataSource Field

File: `GongqiERP\WEB-INF\layers\ext\gongqi.df.<module>\model\got\forms\gongqi.df.<module>.<TableName>.xml`

Insert after a similar FormDataSourceField. Template:
```xml
<FormDataSourceField name="YearId" id="60226" type="String" createdLayer="ext" modifiedLayer="ext">
  <Properties>
    <Property><Name>Label</Name><Value/></Property>
    <Property><Name>HelpText</Name><Value/></Property>
    <Property><Name>DisplayLength</Name><Value/></Property>
    <Property><Name>AllowEdit</Name><Value/></Property>
    <Property><Name>Visible</Name><Value>true</Value></Property>
    <Property><Name>Regular</Name><Value/></Property>
    <Property refentityid="gongqi.df.<module>.<entityId>" refinnerid="<entity-field-id>">
      <Name>Field</Name>
      <Value>YearId</Value>
    </Property>
    <Property><Name>Default</Name><Value/></Property>
    <Property><Name>Fetch</Name><Value>true</Value></Property>
    <Property><Name>NotNull</Name><Value/></Property>
    <Property><Name>AggregateType</Name><Value/></Property>
  </Properties>
</FormDataSourceField>
```

### Layer 6: Form Item in DataGrid

Insert after a sibling FormItem in the DataGrid. Template (StringEditor, editable):
```xml
<FormItem name="MasterScheduleTable_YearId" id="60227" createdLayer="ext" modifiedLayer="ext" type="StringEditor">
  <Properties>
    <Property><Name>Width</Name><Value/></Property>
    <Property><Name>Height</Name><Value/></Property>
    <Property><Name>Label</Name><Value>年</Value></Property>
    <Property><Name>AllowEdit</Name><Value>true</Value></Property>
    <Property><Name>Visible</Name><Value/></Property>
    <Property><Name>PlaceHolder</Name><Value>false</Value></Property>
    <Property><Name>SkipTab</Name><Value>false</Value></Property>
    <Property><Name>HelpText</Name><Value/></Property>
    <Property><Name>DisplayLength</Name><Value/></Property>
    <Property><Name>DisplayHeight</Name><Value>1</Value></Property>
    <Property><Name>Regular</Name><Value/></Property>
    <Property><Name>Default</Name><Value/></Property>
    <Property innerid="<ds-id>"><Name>DataSource</Name><Value>MasterScheduleTable</Value></Property>
    <Property refentityid="gongqi.df.<module>.<entityId>" refinnerid="<entity-field-id>">
      <Name>Extends</Name>
      <Value>YearId</Value>
    </Property>
    <Property><Name>DataMethod</Name><Value/></Property>
    <Property><Name>AggregationField</Name><Value/></Property>
    <Property><Name>QuerySource</Name><Value/></Property>
    <Property><Name>QueryField</Name><Value/></Property>
    <Property><Name>QueryFilter</Name><Value/></Property>
    <Property><Name>HasDropDown</Name><Value/></Property>
    <Property><Name>DropDownMode</Name><Value>PullDown</Value></Property>
    <Property><Name>StringSize</Name><Value>20</Value></Property>
    <Property><Name>SearchMode</Name><Value>AfterInput</Value></Property>
    <Property><Name>FormLookup</Name><Value/></Property>
    <Property><Name>ValidateLookup</Name><Value>true</Value></Property>
    <Property><Name>ReturnLookup</Name><Value>Replace</Value></Property>
    <Property><Name>AllowKeyIn</Name><Value>true</Value></Property>
    <Property><Name>FilterEditor</Name><Value/></Property>
    <Property><Name>ChangeCase</Name><Value>None</Value></Property>
    <Property><Name>Trim</Name><Value>LeftRight</Value></Property>
    <Property><Name>NotNull</Name><Value>false</Value></Property>
  </Properties>
</FormItem>
```

Bump form `nextInnerId` on root `<Form>` element each time you add a new item.

## nextInnerId rule

| Context | Use |
|---------|-----|
| Entity Table XML | `nextInnerId` on `<Table>` → use as TableField id, then +1 |
| Form XML | `nextInnerId` on `<Form>` → use as FormItem id, then +1 |
| FormDataSourceField id | Use same as entity TableField id or find free slot |

## Verification

```powershell
Select-String -Path "<file>" -Pattern "<FieldName>|nextInnerId"
```

Check GOT error log:
```powershell
Get-Content "D:\Gongqi\PJ\ZongYu\logs\erp\error.log" -Tail 30
```

## Type reference

| GOT Type | Java Type | cast method | Setter helper |
|----------|-----------|-------------|---------------|
| String | String | castString | doFormat_String |
| Decimal | BigDecimal | castDecimal | doFormat_Decimal |
| Date | Date | castDate | doFormat_Date |
| Boolean | boolean | castBoolean | (direct assign) |
