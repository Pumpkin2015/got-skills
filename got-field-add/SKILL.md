---
name: got-add-field-with-logic
description: Add fields to a GOT entity table with computed logic (sum, etc.) — safe 4-layer sync workflow to avoid DAO failures and Form XML corruption.
---

# GOT Add Field With Logic

## When to use
Adding one or more fields to a GOT table. Two variants:
- **Standard (Layer) entity**: has Form, Java Base, Client JS → use 7-layer checklist below
- **Plugin/DataSync table** (DT* prefix, `createdLayer="plg"`): no Form, no Java → see the "Plugin Table Variant" section at the bottom of this skill
- Reference: `references/plugin-datasync-example.md` for a concrete DTmaster example

## Deployment (Codex)

This skill is installed as a Codex skill at `C:\Users\cgcab\.codex\skills\got-field-add\SKILL.md`.
When Codex runs a GOT field addition task:
1. Codex++ proxy must be reachable — check: `Invoke-WebRequest http://127.0.0.1:57321/v1/models`
2. Invoke via wrapper: `powershell -EP Bypass -File codex-wrapper.ps1 exec --sandbox danger-full-access`
3. Git must be on PATH so Codex can commit intermediate states for crash recovery
4. Codex must follow the 7-layer order exactly — DAO failure means layer sync was broken

## CRITICAL: 4-layer sync rule
**Never** update Table XML without also updating the compiled `.class` files. GOT will fail with "获取DAO失败" if Table XML has a field but the compiled class lacks the getter/setter.

## ⚠️ GOT 添加字段的 7 层检查清单

vendor 审查发现，我们之前的实现漏掉了多个层面和属性。下面是完整的 7 层清单：

| # | 层 | 文件/位置 | 关键内容 |
|---|-----|---------|---------|
| 1 | 数据库 | MySQL `gongqi` | `ALTER TABLE ADD COLUMN` |
| 2 | Java Base | `Base<Entity>.java` | getter/setter/castDecimal/常量 |
| 3 | Java 编译 | `javac` + 部署 `.class` | **必须先于 Table XML** |
| 4 | Table XML | `<Entity>.xml` | `<TableField>` + bump `nextInnerId` |
| 5 | Form XML | `<Entity>.xml` | FormDataSourceField + FormItem + Tab/TabPage |
| 6 | Java DataSource | `DataSource_<Entity>.java` | beforeSaveRecord 等业务逻辑 |
| 7 | Client JS | `Client/LayerProject/` | 自动生成的 JS 桩文件 |

**常见漏项：**
- **漏掉 Client JS 文件** → 前端加载失败
- **漏掉 FormDataSourceField 的属性**（Fetch/Visible/NotNull/Regular）→ 数据绑定异常
- **漏掉 FormItem 的属性**（DisplayStyle/NotNull/AllowBlank）→ 渲染异常
- **漏掉 TabPage 的 LabelLength** → 标签对齐问题
- **TableField 属性不完整** → GOT 运行时使用默认值可能不符合预期

---

## Workflow

### Phase 1: Database (safe, no restart needed)
```sql
ALTER TABLE gongqi.<TableName> ADD COLUMN <FieldName> DECIMAL(26,2) DEFAULT NULL;
```

### Phase 2: Java Base class — add getter/setter + cast
Edit `LayerProject/layer_ent/.../tables/base/Base<Entity>.java`:
```java
private BigDecimal fieldName;
public BigDecimal getFieldName() { return fieldName; }
public void setFieldName(BigDecimal fieldName) { this.fieldName = fieldName; }
// In cast():
this.setFieldName(GongqiRecordHelper.castDecimal(map, "fieldName"));
// Constant:
final public static String _FieldName = "FieldName";
```

### Phase 3: Compile + deploy .class
Use `got-layer-packaging.md` pattern. Deploy to `WEB-INF/layers/ent/binary/classes/`.

### Phase 4: Table XML — COMPLETE field definition

**每个 TableField 必须包含以下全部属性（14 个）：**

```xml
<TableField name="FieldName" id="NNNNN" createdLayer="ent" modifiedLayer="ent" type="Decimal">
  <Properties>
    <Property><Name>Extends</Name><Value/></Property>
    <Property><Name>Label</Name><Value>显示名</Value></Property>
    <Property><Name>HelpText</Name><Value/></Property>
    <Property><Name>DisplayLength</Name><Value/></Property>
    <Property><Name>AllowEdit</Name><Value>Always</Value></Property>
    <Property><Name>Default</Name><Value/></Property>
    <Property><Name>NumOfDecimals</Name><Value>2</Value></Property>
    <Property><Name>DisplayStyle</Name><Value/></Property>
    <Property><Name>Regular</Name><Value/></Property>
    <Property><Name>NotNull</Name><Value>false</Value></Property>
    <Property><Name>ThousandSeperator</Name><Value/></Property>
    <Property><Name>ShowZero</Name><Value>false</Value></Property>
    <Property><Name>RoundOff</Name><Value>HalfUp</Value></Property>
    <Property><Name>DisplayDecimals</Name><Value>2</Value></Property>
  </Properties>
</TableField>
```

Update `nextInnerId` on root `<Table>` element to be higher than the last field's id.

### Phase 5: Form XML — THREE sections to update

#### 5a. FormDataSourceField — COMPLETE property set (11 个属性)

```xml
<FormDataSourceField name="FieldName" id="DSF_ID" createdLayer="ent" modifiedLayer="ent" type="Decimal">
  <Properties>
    <Property><Name>Label</Name><Value/></Property>
    <Property><Name>HelpText</Name><Value/></Property>
    <Property><Name>DisplayLength</Name><Value/></Property>
    <Property><Name>AllowEdit</Name><Value>Always</Value></Property>  <!-- Never for total/readonly -->
    <Property><Name>Visible</Name><Value>true</Value></Property>
    <Property><Name>Regular</Name><Value/></Property>
    <Property refentityid="TABLE_ID" refinnerid="TABLEFIELD_ID"><Name>Field</Name><Value>FieldName</Value></Property>
    <Property><Name>Default</Name><Value/></Property>
    <Property><Name>Fetch</Name><Value>true</Value></Property>
    <Property><Name>AggregateType</Name><Value/></Property>
    <Property><Name>NotNull</Name><Value/></Property>
  </Properties>
</FormDataSourceField>
```

⚠️ **Fetch 为 true** — 缺失会导致数据绑定异常。  
⚠️ **refentityid=TABLE_ID, refinnerid=TABLEFIELD_ID** — 必须匹配 Table XML 中的 ID。

#### 5b. FormItem (DecimalEditor) — COMPLETE property set (18 个属性)

```xml
<FormItem name="EntityName_FieldName" id="FI_ID" createdLayer="ent" modifiedLayer="ent" type="DecimalEditor">
  <Properties>
    <Property><Name>Width</Name><Value/></Property>
    <Property><Name>Height</Name><Value/></Property>
    <Property><Name>Label</Name><Value/></Property>
    <Property><Name>AllowEdit</Name><Value/></Property>  <!-- Never for total/readonly -->
    <Property><Name>Visible</Name><Value>true</Value></Property>
    <Property><Name>PlaceHolder</Name><Value>false</Value></Property>
    <Property><Name>SkipTab</Name><Value>false</Value></Property>
    <Property><Name>DisplayLength</Name><Value>8</Value></Property>
    <Property><Name>DisplayHeight</Name><Value>1</Value></Property>
    <Property><Name>HelpText</Name><Value/></Property>
    <Property><Name>DisplayDecimals</Name><Value>2</Value></Property>
    <Property><Name>ShowZero</Name><Value>true</Value></Property>
    <Property><Name>Regular</Name><Value/></Property>
    <Property><Name>Default</Name><Value/></Property>
    <Property><Name>AllowBlank</Name><Value/></Property>
    <Property><Name>DisplayStyle</Name><Value/></Property>
    <Property><Name>NotNull</Name><Value/></Property>
    <Property innerid="FORM_DS_ID"><Name>DataSource</Name><Value>EntityName</Value></Property>
    <Property refentityid="TABLE_ID" refinnerid="TABLEFIELD_ID"><Name>Extends</Name><Value>FieldName</Value></Property>
    <Property><Name>HasDropDown</Name><Value/></Property>
    <Property><Name>DropDownMode</Name><Value>PullDown</Value></Property>
  </Properties>
</FormItem>
```

**DecimalEditor 必须包含的属性（缺一不可）：**
- `DataSource` (innerid) — 绑定数据源
- `Extends` (refentityid+refinnerid) — 绑定 TableField
- `HasDropDown` — 缺失 → Spring init failure
- `DropDownMode="PullDown"` — 缺失 → `FormItem DropDownMode不能为空`
- `DisplayHeight` — GOT Designer 默认生成
- `AllowBlank` — GOT Designer 默认生成
- `DisplayStyle` — GOT Designer 默认生成
- `NotNull` — GOT Designer 默认生成

#### 5c. Tab/TabPage (if creating new tab)

**Tab 容器属性：**
```xml
<FormItem name="Tab" id="TAB_ID" createdLayer="ent" modifiedLayer="ent" type="Tab">
  <Properties>
    <Property><Name>Width</Name><Value/></Property>
    <Property><Name>Height</Name><Value/></Property>
    <Property><Name>Label</Name><Value/></Property>
    <Property><Name>AllowEdit</Name><Value>true</Value></Property>
    <Property><Name>Visible</Name><Value>true</Value></Property>
    <Property><Name>PlaceHolder</Name><Value>false</Value></Property>
    <Property><Name>SkipTab</Name><Value>false</Value></Property>
    <Property><Name>DataSource</Name><Value/></Property>
    <Property><Name>Layout</Name><Value>vertical</Value></Property>  <!-- 绝对不能为空 -->
  </Properties>
  <!-- TabPages go here -->
</FormItem>
```

**TabPage 属性（含 LabelLength）：**
```xml
<FormItem name="TabPage" id="TP_ID" createdLayer="ent" modifiedLayer="ent" type="TabPage">
  <Properties>
    <Property><Name>Width</Name><Value/></Property>
    <Property><Name>Height</Name><Value/></Property>
    <Property><Name>Label</Name><Value>标签名</Value></Property>
    <Property><Name>AllowEdit</Name><Value>true</Value></Property>
    <Property><Name>Visible</Name><Value>true</Value></Property>
    <Property><Name>PlaceHolder</Name><Value>false</Value></Property>
    <Property><Name>SkipTab</Name><Value>false</Value></Property>
    <Property><Name>DataSource</Name><Value/></Property>
    <Property><Name>Layout</Name><Value>vertical</Value></Property>  <!-- 绝对不能为空 -->
    <Property><Name>LabelLength</Name><Value>7</Value></Property>
  </Properties>
  <!-- Editors go here (Strategy B: direct) -->
</FormItem>
```

⚠️ **Layout 不能为空** — 空值 → Spring init 崩溃  
⚠️ **LabelLength** — GOT Designer 生成的 TabPage 都带这个属性，控制标签列宽度

### Phase 6: Business logic — DataSource override
```java
@Override
protected void beforeSaveRecord(CommandArg arg, GongqiRecord record) {
    Entity e = (Entity) record;
    // compute logic
    e.setComputedField(value);
    super.beforeSaveRecord(arg, record);
}
```
Compile and deploy as in Phase 3.

### Phase 7: Client-side JS stubs
GOT 在 `Client/LayerProject/` 下也需要自动生成的 JS 桩文件：

```
Client/LayerProject/src/layer_ent/.../forms/<Entity>/
├── Form_<Entity>.js
├── base/FormBase_<Entity>.js
└── datasource/
    ├── Form_<Entity>_DataSource_<Entity>.js
    └── base/Form_<Entity>_DataSourceBase_<Entity>.js
```

这些文件是 GOT Designer 自动生成的空壳类。如果新增的字段在 Form XML 中已正确配置，这些 JS 文件通常不需要手动修改——但必须存在。如果缺失，用 GOT Designer 重新生成。

### Phase 8: Restart Tomcat
Clear work cache, kill Tomcat, restart.

## Plugin Table (Data Sync) Variant — Simplified 3-Layer Workflow

**When to use this variant**: adding a field to a GOT **plugin** table (e.g., DTmaster, DTprod, any `gongqi.df.*` plugin with `DT*` prefix tables). These are data-sync tables — no frontend form, no Java source, no Client JS. GOT Designer auto-generates everything from XML at runtime.

### Plugin Table Field Addition Checklist

| # | 层 | 文件/位置 | 关键内容 |
|---|-----|---------|---------|
| 1 | DataType XML | `extract_plugins/<plugin>/.../datatypes/` | **先创建** DataType（如果用 Extends/refentityid） |
| 2 | Table XML | `extract_plugins/<plugin>/.../tables/` | `<TableField>` + bump `nextInnerId` |
| 3 | 数据库 | MySQL | `ALTER TABLE ADD COLUMN` |

**跳过**：Java Base、Form XML、DataSource、Client JS —— 全部不需要。

### Key Differences from Standard (Layer) Tables

| 属性 | 标准 Layer 表 | Plugin 表 |
|------|-------------|----------|
| TableField `createdLayer` | `ent` / `prj` | `plg` |
| `refentityid` 格式 | 数字 ID（如 `20103039`） | 完全限定名（如 `gongqi.df.DTmaster.20520006`） |
| `Extends` Value | 简短名（如 `WeekId`） | 完全限定名（如 `gongqi.df.DTmaster.WeekId`） |
| DataType XML | 通常已有系统 DataType | **需要新建**（放在 plugin 的 datatypes 目录） |
| 副本同步 | 仅 LayerProject + Tomcat | Designer + JDT + Tomcat 三处 |

### Plugin TableField Template (String type)

```xml
<TableField name="WeekId" id="NEXT_INNER_ID" createdLayer="plg" modifiedLayer="plg" type="String"> 
  <Properties> 
    <Property refentityid="gongqi.df.DTmaster.DATATYPE_ID"> 
      <Name>Extends</Name>  
      <Value>gongqi.df.DTmaster.WeekId</Value> 
    </Property>  
    <Property><Name>Label</Name><Value/></Property>
    <Property><Name>HelpText</Name><Value/></Property>
    <Property><Name>DisplayLength</Name><Value/></Property>
    <Property><Name>Alignment</Name><Value/></Property>
    <Property><Name>AllowEdit</Name><Value>false</Value></Property>
    <Property><Name>NotNull</Name><Value>false</Value></Property>
    <Property><Name>AllowEditOnCreate</Name><Value>true</Value></Property>
    <Property><Name>Visible</Name><Value>true</Value></Property>
    <Property><Name>Regular</Name><Value/></Property>
    <Property><Name>Default</Name><Value/></Property>
    <Property><Name>StringSize</Name><Value>20</Value></Property>
    <Property><Name>FormLookup</Name><Value/></Property>
    <Property><Name>ValidateLookup</Name><Value>true</Value></Property>
    <Property><Name>FilterField</Name><Value/></Property>
  </Properties> 
</TableField>
```

### Plugin DataType Template

```xml
<DataType name="gongqi.df.DTmaster.WeekId" id="gongqi.df.DTmaster.NEXT_DATATYPE_ID" version="1" nextInnerId="50000" createdLayer="plg" modifiedLayer="plg" type="String"> 
  <Properties> 
    <Property><Name>Extends</Name><Value/></Property>
    <Property><Name>Label</Name><Value>周号</Value></Property>
    <Property><Name>StringSize</Name><Value>20</Value></Property>
    <Property><Name>SysActive</Name><Value>true</Value></Property>
    ... (standard DataType properties)
  </Properties> 
</DataType>
```

### Copy-Sync Checklist (after modifying Designer extract)

```bash
# Copy DataType and Table XML to ALL locations:
# 1. JDT extract (if exists)
# 2. Tomcat temp extract
# JDT may not exist if not recently compiled — skip if absent
```

## Pitfalls
- **Never** use `replace_all` or global regex on Form XML — use line-precise insertion
- FormItem MUST be inside the correct parent (DataGrid or TabPage), not sibling of it
- FormItem MUST have ALL properties listed above — vendor 审查发现我们漏掉了 7 个属性
- FormDataSourceField MUST have ALL 11 properties — vendor 审查发现我们只写了 3 个
- TableField MUST have ALL 14 properties — 不完整会导致运行时行为不符合预期
- TabPage MUST have LabelLength + Layout
- Client JS stubs must exist even if empty
- Missing DropDownMode causes: "FormItem DropDownMode不能为空"
- Test with ONE field first, then repeat the exact pattern for others
- TabPage 使用 Strategy B (direct editors) — Group 容器不渲染编辑器
- Plugin tables: TableField 的 `refentityid` 和 `Extends` Value 必须用完全限定名（如 `gongqi.df.DTmaster.20520006`），不能用简短名或数字 ID
- Plugin tables: 先创建 DataType XML 再改 Table XML，否则 refentityid 指向不存在的实体
- Plugin tables: Designer/JDT/Tomcat 三处 extract 副本必须全部同步，漏掉任一处都会导致运行时不一致
- Plugin tables: MySQL 端口（40000/60300）在 Windows 防火墙后，WSL 无法直连 —— 生成 `.sql` 文件供用户在 Windows 上手动执行
- **撤销插件表修改**：删除新建的 DataType XML（3处）→ patch 移除 TableField 块 → patch 恢复 nextInnerId → 同步副本 → 清理产出目录。4 步干净还原，不留残留
