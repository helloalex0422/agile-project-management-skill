# Notion 适配器实现指引

本文件描述如何使用 `notion-tool`（`tools/notion_tool.py`）实现 `board-interface.md` 中定义的抽象操作。

## 前提条件

- 已获得 Notion Integration Token（`agile-project-management:setup` 完成）
- Token 已保存至 `~/.claude/agile-pm/config.json` 的 `notion.api_key`，或设置了环境变量 `NOTION_API_KEY`
- 已创建 Task Database（Database ID 保存在配置中）
- 已创建 Dashboard Page（Page ID 保存在配置中）

## notion-tool 调用方式

所有 Notion 操作通过 Bash 调用 `tools/notion_tool.py` 完成：

```bash
uv run tools/notion_tool.py <command> '<json_args>'
```

工具输出 JSON 到 stdout，错误输出到 stderr。exit code 非 0 表示失败。

> **注意**：`tools/notion_tool.py` 的路径相对于插件根目录 `agile-project-management-skill/`。执行时使用完整路径或确保工作目录正确。

## 配置参数

运行 `agile-project-management:setup` 后，以下配置保存在 `~/.claude/agile-pm/config.json`：

```json
{
  "board_backend": "notion",
  "notion": {
    "api_key": "<Notion Integration Token>",
    "task_db_id": "<Task Database 的 ID>",
    "dashboard_page_id": "<Dashboard Page 的 ID>",
    "parent_page_id": "<Parent Page 的 ID>"
  }
}
```

所有 skill 在执行前先读取此文件，获取 `notion.task_db_id`（下文简称 `NOTION_TASK_DB_ID`）和 `notion.dashboard_page_id`（下文简称 `NOTION_DASHBOARD_PAGE_ID`）。

---

## 操作实现映射

### list_tasks(filter)

调用 notion-tool 的 `query-database` 命令：

```bash
uv run tools/notion_tool.py query-database '{
  "database_id": "NOTION_TASK_DB_ID",
  "filter": <Notion filter 格式>
}'
```

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

调用 notion-tool 的 `create-page` 命令：

```bash
uv run tools/notion_tool.py create-page '{
  "parent": { "type": "database_id", "database_id": "NOTION_TASK_DB_ID" },
  "properties": <Notion property 格式>
}'
```

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
  "Due Date": { "date": { "start": "2026-05-25" } },
  "Notes": { "rich_text": [{ "text": { "content": "站会备注" } }] }
}
```

**Status 字段有效值：**
- `Draft` — 草稿，用户在 Notion 表格中快速录入的未处理条目
- `Backlog` — 已确认的待办任务
- `Sprint Todo` — 已分配到 Sprint 的待开始任务
- `In Progress` — 进行中
- `In Review` — 审核中
- `Done` — 已完成

---

### update_task(id, fields) / move_task(id, status)

调用 notion-tool 的 `update-page` 命令：

```bash
uv run tools/notion_tool.py update-page '{
  "page_id": "<任务的 Notion page ID>",
  "properties": <只包含需要更新的字段，格式同 create_task>
}'
```

---

### list_sprints()

调用 notion-tool 的 `retrieve-database` 获取 Sprint select 选项列表：

```bash
uv run tools/notion_tool.py retrieve-database '{
  "database_id": "NOTION_TASK_DB_ID"
}'
```

从返回的 `properties.Sprint.select.options` 中提取 Sprint 列表。已归档的 Sprint 名称包含 `-archived` 后缀。

---

### create_sprint(name, dates)

Sprint 在 Notion 中表现为 Task Database 的 `Sprint` select 选项。

调用 notion-tool 的 `update-database` 向 Sprint 属性添加新的 select 选项：

```bash
uv run tools/notion_tool.py update-database '{
  "database_id": "NOTION_TASK_DB_ID",
  "properties": {
    "Sprint": {
      "select": {
        "options": [<现有选项列表>, { "name": "Sprint-02" }]
      }
    }
  }
}'
```

Sprint 的起止日期记录在 `~/.claude/agile-pm/config.json` 的 `current_sprint` 字段和 Dashboard Page 的 Sprint 概览模块中。

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

Dashboard 是一个 Notion Page，使用 notion-tool 的 block 操作更新内容。

**读取 Dashboard：**

```bash
uv run tools/notion_tool.py list-block-children '{
  "block_id": "NOTION_DASHBOARD_PAGE_ID"
}'
```

**删除旧 block：**

对每个需要删除的 block 调用：

```bash
uv run tools/notion_tool.py delete-block '{
  "block_id": "<block_id>"
}'
```

**写入新 block：**

```bash
uv run tools/notion_tool.py append-block-children '{
  "block_id": "NOTION_DASHBOARD_PAGE_ID",
  "children": [<block 列表>]
}'
```

**页面结构（block 顺序）：**
1. Heading 1: "📊 项目管理 Dashboard"
2. Heading 2: "🏃 当前 Sprint 概览" + Paragraph（Sprint 名称、日期、完成率）
3. Heading 2: "📋 任务状态分布" + Bulleted list（各状态数量）
4. Heading 2: "🚨 Blocker 列表" + Bulleted list（P0任务）
5. Heading 2: "📁 各项目进度" + Bulleted list（按项目分组）
6. Heading 2: "📅 最近站会记录" + Bulleted list（最近3次，格式：`YYYY-MM-DD: <摘要>`）

**更新策略：** 先获取 Dashboard Page 的所有 block，删除旧内容，再 append 新内容。
