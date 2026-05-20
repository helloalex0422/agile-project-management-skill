# Notion 适配器实现指引

本文件描述如何使用 Notion MCP 实现 `board-interface.md` 中定义的抽象操作。

## 前提条件

- 已配置 Notion MCP（`project-manage:setup` 完成）
- 已获得 Notion Integration Token
- 已创建 Task Database（Database ID 保存在配置中）
- 已创建 Dashboard Page（Page ID 保存在配置中）

## 配置参数

运行 `project-manage:setup` 后，以下配置会被记录：

```
NOTION_TASK_DB_ID=<Task Database 的 ID>
NOTION_DASHBOARD_PAGE_ID=<Dashboard Page 的 ID>
BOARD_BACKEND=notion
```

---

## 操作实现映射

### list_tasks(filter)

调用 Notion MCP 的 `query_database` 工具：
- `database_id`: `NOTION_TASK_DB_ID`
- `filter`: 将抽象 filter 转换为 Notion filter 格式

**Notion filter 转换示例：**
```json
// status = "In Progress" 的 Notion filter
{
  "property": "Status",
  "select": { "equals": "In Progress" }
}

// 多条件（status + sprint）
{
  "and": [
    { "property": "Status", "select": { "equals": "In Progress" } },
    { "property": "Sprint", "select": { "equals": "Sprint-01" } }
  ]
}
```

**返回值映射：** 将 Notion page 对象的 `properties` 字段映射到 Task 数据模型，`id` 取 Notion page ID。

---

### create_task(fields)

调用 Notion MCP 的 `create_page` 工具：
- `parent`: `{ "database_id": "NOTION_TASK_DB_ID" }`
- `properties`: 将 fields 映射到 Notion property 格式

**Notion property 格式示例：**
```json
{
  "Name": { "title": [{ "text": { "content": "任务标题" } }] },
  "Status": { "select": { "name": "Backlog" } },
  "Type": { "select": { "name": "工作项目" } },
  "Project": { "select": { "name": "ssb-tool" } },
  "Priority": { "select": { "name": "P1" } },
  "Estimate": { "number": 4 },
  "Description": { "rich_text": [{ "text": { "content": "详细描述" } }] },
  "GitLab Issue": { "url": "http://gitlab.internal/..." },
  "GitLab MR": { "url": "http://gitlab.internal/..." },
  "Due Date": { "date": { "start": "2026-05-25" } },
  "Notes": { "rich_text": [{ "text": { "content": "站会备注" } }] }
}
```

---

### update_task(id, fields) / move_task(id, status)

调用 Notion MCP 的 `update_page` 工具：
- `page_id`: 任务的 Notion page ID
- `properties`: 只包含需要更新的字段（格式同 create_task）

---

### list_sprints()

调用 `query_database`，获取 `Sprint` 字段的所有唯一值：
- 使用 Notion MCP 的 `retrieve_database` 获取 Sprint select 选项列表
- 已归档的 Sprint 名称包含 `-archived` 后缀

---

### create_sprint(name, dates)

Sprint 在 Notion 中表现为 Task Database 的 `Sprint` select 选项。
- 调用 `update_database` 向 Sprint 属性添加新的 select 选项（`name` 即 Sprint 名称）
- Sprint 的起止日期记录在 Dashboard Page 的 Sprint 概览模块中

---

### archive_sprint(sprint_name)

1. 调用 `list_tasks({ sprint: sprint_name })` 获取该 Sprint 的所有任务
2. 对每个任务调用 `update_task(id, { sprint: sprint_name + "-archived" })`

---

### get_sprint_stats(sprint_name)

1. 调用 `list_tasks({ sprint: sprint_name })` 获取所有任务
2. 在内存中统计各状态数量
3. 识别 Blocker：`priority = "P0"` 或 `notes` 包含 `[blocker]` 标记

---

### update_dashboard(sections) / get_dashboard()

Dashboard 是一个 Notion Page，使用 `append_block_children` 和 `delete_block` 更新内容。

**页面结构（block 顺序）：**
1. Heading 1: "📊 项目管理 Dashboard"
2. Heading 2: "🏃 当前 Sprint 概览" + Paragraph（Sprint 名称、日期、完成率）
3. Heading 2: "📋 任务状态分布" + Bulleted list（各状态数量）
4. Heading 2: "🚨 Blocker 列表" + Bulleted list（P0任务）
5. Heading 2: "📁 各项目进度" + Bulleted list（按项目分组）
6. Heading 2: "📅 最近站会记录" + Bulleted list（最近3次，格式：`YYYY-MM-DD: <摘要>`）

**更新策略：** 先获取 Dashboard Page 的所有 block，删除旧内容，再 append 新内容。
