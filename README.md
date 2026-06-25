# GOT Skills for Codex

工企 GOT ERP 开发辅助技能集，用于 Codex CLI / Codex Desktop 环境。

## 技能列表

| 技能 | 描述 |
|------|------|
| **got-field-add** | 向 GOT 实体表添加字段（含计算逻辑），覆盖 7 层同步工作流（MySQL → Java Base → 编译 → Table XML → Form XML → DataSource → Client JS），同时支持 Plugin/DataSync 表变体 |
| **got-field-full** | 新增字段一步到位：从数据库列到表单 DataGrid 显示一气呵成，覆盖 MySQL、实体 TableField、Java Base getter/setter/cast/const、JDT 编译、FormDataSourceField、FormItem 六层，含 git 安全回滚流程 |
| **got-form-show-field** | 将已存在的实体字段显示到 GOT 表单 DataGrid 中——字段已存在于实体表和 FormDataSourceField，只需补充缺失的 FormItem 显示控件 |
| **got-tree-structure** | 在 GOT ERP 中创建树状结构表单，支持三种模式：(A) 自引用单表树 (ParentXxxId)、(B) 分类+明细双表树 (ItemClass+ItemTable)、(C) 自定义动态树 (TreeSetDataArg+DOM4J，如按年/周分组) |

## 安装

将各子目录复制到 Codex skills 目录：

```powershell
Copy-Item -Recurse got-field-add C:\Users\<用户名>\.codex\skills\
Copy-Item -Recurse got-field-full C:\Users\<用户名>\.codex\skills\
Copy-Item -Recurse got-form-show-field C:\Users\<用户名>\.codex\skills\
Copy-Item -Recurse got-tree-structure C:\Users\<用户名>\.codex\skills\
```

## 环境要求

- Codex CLI 或 Codex Desktop
- GOT ERP 项目环境（ZongYu / ChenGuang / MESold）
- Git（用于安全回滚）
- MySQL 命令行客户端

## 适用环境路径

| 环境 | 基础路径 |
|------|----------|
| ZongYu | `D:\Gongqi\PJ\ZongYu\Project\Server\GongqiERP\WEB-INF\` |
| ChenGuang | `D:\Gongqi\PJ\ChenGuang\Project\Server\GongqiERP\WEB-INF\` |
| MESold | `D:\Gongqi\PJ\MESold\Project\Server\GongqiERP\WEB-INF\` |
