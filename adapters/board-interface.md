# Board Interface — 抽象看板操作规范

所有 skill 文件只调用本规范中定义的操作，不直接调用 Notion 或飞书 API。
具体实现由当前配置的适配器提供（见 `adapters/notion.md`）。

## 配置

当前后端由环境变量或用户配置决定：
- `BOARD_BACKEND=notion`（默认）
- `BOARD_BACKEND=feishu`（未来支持）

使用时，根据 `BOARD_BACKEND` 加载对应适配器的实现指引。

---

## 操作规范

### list_tasks(filter)

查询任务列表。

**filter 可选参数：**
- `status`: `"Backlog"` | `"Sprint Todo"` | `"In Progress"` | `"In Review"` | `"Done"`
- `type`: `"个人项目"` | `"工作项目"`
- `project`: 项目名字符串
- `sprint`: Sprint 编号，如 `"Sprint-01"`
- `priority`: `"P0"` | `"P1"` | `"P2"`

**返回：** 任务列表，每个任务包含所有字段（见数据模型）。

---

### create_task(fields)

新建任务。

**fields 必填：**
- `name`: 任务标题
- `status`: 初始状态（通常为 `"Backlog"`）
- `type`: `"个人项目"` | `"工作项目"`
- `project`: 所属项目名

**fields 可选：**
- `sprint`, `priority`, `estimate`, `description`, `gitlab_issue`, `due_date`, `notes`

**返回：** 创建的任务对象（含 `id`）。

---

### update_task(id, fields)

更新任务的一个或多个字段。

**参数：**
- `id`: 任务唯一标识
- `fields`: 需要更新的字段（键值对，同 create_task 的可选字段）

**返回：** 更新后的任务对象。

---

### move_task(id, status)

移动任务到指定看板列（更新 status 字段的语义快捷方式）。

**参数：**
- `id`: 任务唯一标识
- `status`: 目标状态（`"Sprint Todo"` | `"In Progress"` | `"In Review"` | `"Done"` | `"Backlog"`）

**返回：** 更新后的任务对象。

---

### list_sprints()

查询所有 Sprint 列表（包含已归档的）。

**返回：** Sprint 列表，每项包含 `{ name, start_date, end_date, is_archived }`。

---

### create_sprint(name, dates)

创建新 Sprint。

**参数：**
- `name`: Sprint 名称，格式 `"Sprint-XX"`（XX为两位数字，如 `"Sprint-01"`）
- `dates`: `{ start: "YYYY-MM-DD", end: "YYYY-MM-DD" }`

**返回：** 创建成功确认。

---

### archive_sprint(sprint_name)

归档指定 Sprint：将该 Sprint 下所有任务的 `sprint` 字段追加 `-archived` 后缀。

**参数：**
- `sprint_name`: 如 `"Sprint-01"`，执行后变为 `"Sprint-01-archived"`

**返回：** 归档的任务数量。

---

### get_sprint_stats(sprint_name)

统计指定 Sprint 的完成情况。

**参数：**
- `sprint_name`: 如 `"Sprint-01"`

**返回：**
```json
{
  "sprint": "Sprint-01",
  "total": 10,
  "done": 6,
  "in_progress": 2,
  "in_review": 1,
  "todo": 1,
  "completion_rate": 0.6,
  "blockers": [{ "id": "...", "name": "...", "priority": "P0" }]
}
```

---

### update_dashboard(sections)

更新 Notion Dashboard 页面内容。

**sections 结构：**
```json
{
  "sprint_overview": {
    "sprint_name": "Sprint-01",
    "start_date": "2026-05-18",
    "end_date": "2026-05-31",
    "completion_rate": 0.6,
    "done": 6,
    "total": 10
  },
  "status_distribution": {
    "backlog": 0,
    "sprint_todo": 1,
    "in_progress": 2,
    "in_review": 1,
    "done": 6
  },
  "blockers": [{ "id": "...", "name": "...", "priority": "P0", "notes": "..." }],
  "project_progress": [
    { "project": "ssb-tool", "done": 3, "total": 5 }
  ],
  "recent_standups": [
    { "date": "2026-05-20", "summary": "完成了X，今日计划Y，无阻塞" }
  ]
}
```

**返回：** 更新成功确认。

---

### get_dashboard()

读取当前 Dashboard 状态。

**返回：** 同 `update_dashboard` 的 sections 结构。

---

## 数据模型：Task 对象

```json
{
  "id": "notion-page-id",
  "name": "任务标题",
  "status": "In Progress",
  "type": "工作项目",
  "project": "ssb-tool",
  "sprint": "Sprint-01",
  "priority": "P1",
  "estimate": 4,
  "description": "任务详细描述",
  "gitlab_issue": "http://gitlab.internal/group/repo/-/issues/42",
  "due_date": "2026-05-25",
  "notes": "站会备注"
}
```
