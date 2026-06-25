---
name: got-tree-structure
description: >
  Create tree-structured forms in GOT ERP (工企 GOT). Covers 3 patterns:
  (A) self-referencing table with ParentXxxId (ProductionOrganization),
  (B) classification+detail two-table (ItemClass+ItemTable),
  (C) custom dynamic tree with TreeSetDataArg+DOM4J (Year/Week grouping).
  Use when designing a tree navigator, adding Plugin Tree to a form,
  implementing pluginTreeNodeChange_, or grouping records by data columns.
  Trigger phrases: "树状结构", "tree", "Plugin Tree", "树形", "tree navigator",
  "树导航", "按年分组", "按字段分组树", "TreeSetDataArg".
---

# GOT Tree Structure (树状结构)

Create or modify tree-structured forms in GOT ERP. Covers self-referencing
single-table trees and classification+detail two-table patterns.

## Two Tree Patterns

### Pattern A: Self-referencing Tree (ProductionOrganization)

Single table with `ParentXxxId` pointing to itself. One Form with
Plugin(Tree)+DataGrid.

**Table XML** (`datadictionary/tables/<Name>.xml`):
```xml
<Table name="Department" TableType="Normal">
  <TableFields>
    <TableField name="DepartmentId" type="String" />       <!-- PK -->
    <TableField name="ParentDepartmentId" type="String" />  <!-- self-ref FK -->
    <TableField name="Name" type="String" />
  </TableFields>
</Table>
```

**Form XML** (`forms/<Name>.xml`):
```xml
<FormDesign>
  <FormItem type="Plugin" PluginType="Tree" Width="250" DataSource="Department" />
  <FormItem type="Tab">
    <FormItem type="TabPage" Label="基本信息">
      <FormItem type="DataGrid">...</FormItem>
    </FormItem>
  </FormItem>
</FormDesign>
```

**Menu**: Add `MenuRefMenuItem` in `menus/<Menu>.xml`.

### Pattern B: Classification + Detail (ItemClass + ItemTable)

Two tables: a classification tree and a detail table linked by `ItemClassId`.

- **ItemClass** table: `ParentItemClassId` self-ref, `IsLeafNode` Boolean.
  No Form XML needed — uses framework built-in MenuItem (ref `50100062`).
- **ItemTable** form: left Plugin Tree (DataSource empty — framework mounts
  ItemClass), right DataGrid filtered by `ItemClassId`.

**ItemTable Form** pattern:
```xml
<FormItem type="Plugin" PluginType="Tree" Width="200" />  <!-- no DataSource -->
```



### Pattern C: Custom Dynamic Tree (Non-Self-Referencing) – e.g., Year/Week Grouping

When the tree groups records by a data column (no ParentXxxId), build tree XML
manually with TreeSetDataArg + DOM4J. Used for grouping by YearId, WeekId, etc.

#### CRITICAL: Tree Plugin XML Configuration

The Tree plugin **MUST have ALL of DataSource, DataColumn, Parameters set to EMPTY**.
If any of these are set (especially DataSource pointing to a table), the GOT
framework will **auto-build a tree from the table's data structure** (e.g.,
ProductionOrganization hierarchy), which **silently overrides** your custom
Java-built tree. This is the #1 cause of "my tree isn't showing" bugs.

```xml
<!-- CORRECT: All three must be empty -->
<FormItem name="Tree" type="Plugin" PluginType="Tree" Width="250">
  <Property><Name>DataSource</Name><Value/></Property>   <!-- MUST BE EMPTY -->
  <Property><Name>DataColumn</Name><Value/></Property>   <!-- MUST BE EMPTY -->
  <Property><Name>Parameters</Name><Value/></Property>   <!-- MUST BE EMPTY -->
  <Property><Name>Visible</Name><Value>true</Value></Property>
</FormItem>
```

```xml
<!-- WRONG: DataSource set to table name → framework auto-builds tree! -->
<FormItem name="Tree" type="Plugin" PluginType="Tree" Width="250">
  <Property><Name>DataSource</Name><Value>MasterScheduleTable</Value></Property>
  <!-- ↑ THIS WILL OVERRIDE YOUR CUSTOM TREE ↑ -->
</FormItem>
```

**Troubleshooting**: If your custom tree doesn't appear and you see a
production-organization or other unexpected tree instead, check:
1. XML Tree Plugin `DataSource` is empty
2. XML Tree Plugin `DataColumn` is empty
3. XML Tree Plugin `Parameters` is empty

#### Single-Level Tree (e.g., Year only)

**Java – afterInit**: Build tree XML from SQL query, set via TreeSetDataArg.

```java
import gongqi.erp.gotmodel.form.plugins.tree.TreeRefreshArg;
import gongqi.erp.gotmodel.form.plugins.tree.TreeSetDataArg;
import org.dom4j.DocumentHelper;
import org.dom4j.Element;

private void initTreePlugin(CommandResult result, CommandArg arg) {
    // Step 1: CLEAR existing tree first (critical to avoid duplicates!)
    Element emptyRoot = DocumentHelper.createElement("node");
    emptyRoot.addAttribute("id", "root");
    emptyRoot.addAttribute("label", "全部");
    TreeRefreshArg clearArg = new TreeRefreshArg();
    TreeSetDataArg clearData = new TreeSetDataArg();
    clearData.data = emptyRoot.asXML();
    clearArg.arg = clearData;
    clearArg.opr = TreeRefreshArg.SET_DATA;
    result.refreshPlugin_Tree("Tree", clearArg);

    // Step 2: Build custom tree from SQL
    String sql = "select distinct YearId from gongqi_df_master$MasterScheduleTable "
               + "where YearId is not null and YearId != '' order by YearId desc";
    List<Map<String, Object>> rows = DAOHelper.sqlQueryToListAliasMap(sql, new Object[] {});

    Element root = DocumentHelper.createElement("node");
    root.addAttribute("id", "root");
    root.addAttribute("label", "全部");

    for (Map<String, Object> row : rows) {
        String value = (String) row.get("YearId");
        if (StringHelper.isBlank(value)) continue;
        Element node = DocumentHelper.createElement("node");
        node.addAttribute("id", value);
        node.addAttribute("label", value + "年");
        root.add(node);
    }

    TreeRefreshArg refreshArg = new TreeRefreshArg();
    TreeSetDataArg setDataArg = new TreeSetDataArg();
    setDataArg.data = root.asXML();
    setDataArg.expanddeep = 1;
    refreshArg.arg = setDataArg;
    refreshArg.opr = TreeRefreshArg.SET_DATA;
    result.refreshPlugin_Tree("Tree", refreshArg);
}
```

**Tree XML format**:
```xml
<node id="root" label="全部">
  <node id="2026" label="2026年"/>
  <node id="2025" label="2025年"/>
</node>
```

#### Multi-Level Tree (e.g., Year → Week)

For two-level trees, nest child nodes inside parent nodes. Use pipe `|`
separator in child node IDs for later parsing in `pluginTreeNodeChange_`.

```java
private void initTreePlugin(CommandResult result, CommandArg arg) {
    try {
        // Step 1: CLEAR existing tree first
        Element emptyRoot = DocumentHelper.createElement("node");
        emptyRoot.addAttribute("id", "root");
        emptyRoot.addAttribute("label", "全部");
        TreeRefreshArg clearArg = new TreeRefreshArg();
        TreeSetDataArg clearData = new TreeSetDataArg();
        clearData.data = emptyRoot.asXML();
        clearArg.arg = clearData;
        clearArg.opr = TreeRefreshArg.SET_DATA;
        result.refreshPlugin_Tree("Tree", clearArg);

        // Step 2: Build year → week tree
        String yearSql = "select distinct YearId from gongqi_df_master$MasterScheduleTable "
                       + "where YearId is not null and YearId != '' order by YearId desc";
        List<Map<String, Object>> yearRows = DAOHelper.sqlQueryToListAliasMap(yearSql, new Object[] {});

        Element root = DocumentHelper.createElement("node");
        root.addAttribute("id", "root");
        root.addAttribute("label", "全部");

        for (Map<String, Object> row : yearRows) {
            String yearId = (String) row.get("YearId");
            if (StringHelper.isBlank(yearId)) continue;

            Element yearNode = DocumentHelper.createElement("node");
            yearNode.addAttribute("id", yearId);
            yearNode.addAttribute("label", yearId + "年");

            // Week sub-query
            String weekSql = "select distinct WeekId from gongqi_df_master$MasterScheduleTable "
                           + "where YearId = ? and WeekId is not null and WeekId != '' "
                           + "order by cast(WeekId as signed)";
            List<Map<String, Object>> weekRows = DAOHelper.sqlQueryToListAliasMap(weekSql, new Object[] { yearId });
            for (Map<String, Object> wRow : weekRows) {
                String weekId = (String) wRow.get("WeekId");
                if (StringHelper.isBlank(weekId)) continue;

                Element weekNode = DocumentHelper.createElement("node");
                weekNode.addAttribute("id", yearId + "|" + weekId);  // pipe-separated
                weekNode.addAttribute("label", "W" + weekId);
                yearNode.add(weekNode);
            }

            root.add(yearNode);
        }

        TreeRefreshArg refreshArg = new TreeRefreshArg();
        TreeSetDataArg setDataArg = new TreeSetDataArg();
        setDataArg.data = root.asXML();
        setDataArg.expanddeep = 2;  // expand 2 levels
        refreshArg.arg = setDataArg;
        refreshArg.opr = TreeRefreshArg.SET_DATA;
        result.refreshPlugin_Tree("Tree", refreshArg);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

**pluginTreeNodeChange_ for multi-level** – parse pipe-separated ID:
```java
@CommandArgOption(groups="current(fq)", joinGroup="yes")
public CommandResult pluginTreeNodeChange_Tree(CommandArg arg) {
    CommandResult result = new CommandResult();
    String selectedValue = (String) arg.argObject;
    QueryTable qt = arg.getFormQuery();

    // Remove existing YearId and WeekId filters
    QueryWhereItem yearFilter = null;
    QueryWhereItem weekFilter = null;
    for (QueryWhereItem item : qt.queryWhere) {
        if (MasterScheduleTable._YearId.equals(item.fieldName)) yearFilter = item;
        if (MasterScheduleTable._WeekId.equals(item.fieldName)) weekFilter = item;
    }
    if (yearFilter != null) qt.queryWhere.remove(yearFilter);
    if (weekFilter != null) qt.queryWhere.remove(weekFilter);

    if (!"root".equals(selectedValue) && StringHelper.isNotBlank(selectedValue)) {
        if (selectedValue.contains("|")) {
            // Week node: "year|week"
            String[] parts = selectedValue.split("\\|");
            qt.findOrCreateWhereItem(MasterScheduleTable._YearId).setValue(parts[0]);
            qt.findOrCreateWhereItem(MasterScheduleTable._WeekId).setValue(parts[1]);
        } else {
            // Year node
            qt.findOrCreateWhereItem(MasterScheduleTable._YearId).setValue(selectedValue);
        }
    }

    result.updateQuery(qt);
    return result;
}
```
#### Tree Node Stats (Dynamic Count)

To show real-time counts on tree nodes (e.g. "W3 (175/202)" = finished/total),
run one batch GROUP BY query and build a HashMap for O(1) lookup per node.

```java
// Step 2: Build stats map (one query for all year-week counts)
Map<String, long[]> countMap = new HashMap<>();
Map<String, long[]> yearCountMap = new HashMap<>();
long[] rootCounts = new long[2];
String statsSql = "select YearId, WeekId, count(*) total, "
                + "sum(case when Status='0' then 1 else 0 end) finished "
                + "from gongqi_df_master$MasterScheduleTable "
                + "where YearId is not null and WeekId is not null and WeekId != '' "
                + "group by YearId, WeekId";
List<Map<String, Object>> statsRows = DAOHelper.sqlQueryToListAliasMap(statsSql, new Object[] {});
for (Map<String, Object> sr : statsRows) {
    String yId = (String) sr.get("YearId");
    String wId = (String) sr.get("WeekId");
    long total = ((Number) sr.get("total")).longValue();
    long finished = ((Number) sr.get("finished")).longValue();
    countMap.put(yId + "|" + wId, new long[] { total, finished });
    // Year-level aggregate
    long[] yc = yearCountMap.get(yId);
    if (yc == null) { yc = new long[2]; yearCountMap.put(yId, yc); }
    yc[0] += total; yc[1] += finished;
    // Root-level aggregate
    rootCounts[0] += total; rootCounts[1] += finished;
}

// Then use in node labels:
root.addAttribute("label", "全部 (" + rootCounts[1] + "/" + rootCounts[0] + ")");
// Year node: yearLabel = yearId + "年 (" + yc[1] + "/" + yc[0] + ")"
// Week node: weekLabel = "W" + weekId + " (" + wc[1] + "/" + wc[0] + ")"
```

- **Performance**: One SQL query covers all nodes; HashMap lookup is O(1)
- **Format**: `"finished/total"` — finished count first, then total
- **Status mapping**: In this example `Status='0'` = "已结束"; adjust for your domain
- **Number casting**: Use `((Number) row.get("col")).longValue()` for COUNT/SUM results

**Critical: SQL table naming** – Native SQL uses `$`, HQL uses `_`:

| Type | Separator | Example |
|------|-----------|---------|
| Native SQL (select ... from) | `$` | `gongqi_df_master$MasterScheduleTable` |
| HQL (from EntityClass) | `_` | `gongqi_df_master_MasterScheduleTable` |

**Anti-patterns**:
- Do NOT set DataSource/DataColumn/Parameters on Tree Plugin – framework auto-build overrides
- Do NOT call refreshPlugin_Tree without clearing first – nodes will stack duplicate
- Do NOT use `_` separator in native SQL – query silently returns empty
- Do NOT call `pluginTreeNodeChange_` method name without the trailing underscore

## Java Secondary Dev — PluginTree Callbacks

All callback sources: `LayerProject/layer_ent/.../forms/<FormName>/Form_<FormName>.java`

Naming: `plugin<Event>_<PluginName>` (PluginName = Plugin item `name` attribute, default `PluginTree`).

### pluginTreeNodeChange_ — Click to filter
```java
@CommandArgOption(groups="current(fq)", joinGroup="yes")
public CommandResult pluginTreeNodeChange_PluginTree(CommandArg arg) {
    CommandResult result = new CommandResult();
    String selectedId = (String) arg.argObject;
    QueryTable qt = arg.getGroupInfo(__DS_MyTable).getFormQuery();
    if ("root".equals(selectedId)) {
        qt.findOrCreateWhereItem(MyTable._ParentId, QueryWhereStatus.Hide).setValue("");
    } else {
        List<String> leaves = MyTree.util().findLeafNodes(selectedId);
        qt.findOrCreateWhereItem(MyTable._ParentId, QueryWhereStatus.Hide)
          .setValue("#in(" + String.join(",", leaves) + ")");
    }
    result.updateQuery(qt);
    return result;
}
```
> `findLeafNodes(id)` recursively collects all leaf descendants.

### pluginTreeNodeAdd_ — Right-click add
```java
public CommandResult pluginTreeNodeAdd_PluginTree(CommandArg arg) {
    CommandResult result = new CommandResult();
    String selected = (String) arg.getEditorValue(PluginTree);
    Document doc = DocumentHelper.parseText(selected);
    TreeNodeAddArg tna = new TreeNodeAddArg();
    tna.id = doc.getRootElement().attributeValue("id");
    Element node = DocumentHelper.createElement("node");
    node.addAttribute("id", "newNode");
    node.addAttribute("label", "New Node");
    tna.data = node.asXML();
    TreeRefreshArg ta = new TreeRefreshArg();
    ta.arg = tna;
    ta.opr = TreeRefreshArg.ADD_NODE;
    result.refreshPlugin_Tree(PluginTree, ta);
    return result;
}
```

### pluginTreeNodeDelete_ — Right-click delete
```java
public CommandResult pluginTreeNodeDelete_PluginTree(CommandArg arg) {
    CommandResult result = new CommandResult();
    String selected = (String) arg.getEditorValue(PluginTree);
    Document doc = DocumentHelper.parseText(selected);
    TreeRefreshArg ta = new TreeRefreshArg();
    ta.arg = doc.getRootElement().attributeValue("id");
    ta.opr = TreeRefreshArg.DELETE_NODE;
    result.refreshPlugin_Tree(PluginTree, ta);
    return result;
}
```

## Key Classes

| Class | Purpose |
|-------|---------|
| `TreeNodeAddArg` (`.id`, `.data`) | New node params |
| `TreeRefreshArg` (`.opr`, `.arg`) | `ADD_NODE` / `DELETE_NODE` |
| `result.refreshPlugin_Tree(name, arg)` | Execute tree refresh |
| `findLeafNodes(parentId)` | Collect all leaf IDs recursively |

## File Encoding

All GOT XML files: **UTF-8 without BOM + LF (`\n`)**.
```powershell
[System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($False))
```
Never use `Set-Content` or default `Out-File`.

## Environment Quick Reference

| Env | Base Path |
|-----|-----------|
| ZongYu | `D:\Gongqi\PJ\ZongYu\Project\Server\GongqiERP\WEB-INF\layers\ent\model\got\` |
| ChenGuang | `D:\Gongqi\PJ\ChenGuang\Project\Server\GongqiERP\WEB-INF\layers\ent\model\got\` |
| MESold | `D:\Gongqi\PJ\MESold\Project\Server\GongqiERP\WEB-INF\layers\ent\model\got\` |