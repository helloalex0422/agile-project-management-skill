---
name: sprint
description: "Sprint 规划。用户说"新 Sprint"、"规划 Sprint"、"Sprint planning"时触发。创建新 Sprint、从 Backlog 选取任务、检查 GitLab 关联、设定 Sprint 目标。"
---

# agile-project-management:sprint

Sprint 规划。在每个 Sprint 开始时运行，完成 Sprint 创建、任务分配和 GitLab 关联。

---

## 前置：读取配置

读取 `~/.claude/agile-pm/config.json`，获取 `notion.task_db_id` 和 `notion.dashboard_page_id`。
若文件不存在，提示用户先运行 `agile-project-management:setup`，然后停止执行。

---

## 执行流程

### Step 1：确认当前 Sprint 状态

参照 `adapters/notion.md` 中的 `list_sprints` 实现，使用 notion-tool 的 `retrieve-database` 获取 Sprint select 选项列表，展示给用户（已归档的 Sprint 名称包含 `-archived` 后缀）：

```
当前 Sprint：
- Sprint-01（进行中）

历史 Sprint：
- Sprint-00（已归档）
```

询问：要创建新 Sprint，还是查看/调整当前 Sprint 的任务？

- 选择**创建新 Sprint**：继续 Step 2
- 选择**调整当前 Sprint**：跳至 Step 4（检查 GitLab 关联）

### Step 2：创建新 Sprint

收集信息：
- Sprint 名称：自动建议下一个编号（如当前最大是 Sprint-01，建议 Sprint-02），用户可修改
- 开始日期：默认为今天（YYYY-MM-DD 格式）
- 结束日期：默认为开始日期 + 14 天，用户可修改

参照 `adapters/notion.md` 中的 `create_sprint` 实现，使用 notion-tool 的 `update-database` 向 Task Database 的 `Sprint` 属性添加新的 select 选项（即 Sprint 名称字符串）。

输出：
```
✅ Sprint-02 已创建（2026-06-01 ~ 2026-06-14）
```

将 Sprint 信息写入 `~/.claude/agile-pm/config.json` 的 `current_sprint` 字段：

```json
"current_sprint": {
  "name": "Sprint-02",
  "start_date": "2026-06-01",
  "end_date": "2026-06-14"
}
```

### Step 3：从 Backlog 选取任务

参照 `adapters/notion.md` 中的 `list_tasks` 实现，使用 notion-tool 的 `query-database` 查询 Status = "Backlog" 的任务，按 Priority 排序展示：

```
📋 Backlog（共 N 项，按优先级排序）：

P0:
1. [任务A]（工作项目 | ssb-tool | 预估4h）- <Description 前50字>

P1:
2. [任务B]（个人项目 | daily-market-report | 预估2h）- <Description 前50字>

P2:
3. [任务C]（个人项目 | 预估1h）- <Description 前50字>

请选择本 Sprint 要纳入的任务（输入序号，多个用逗号分隔，或输入"全部P0"/"跳过"）：
```

对用户选中的任务，参照 `adapters/notion.md` 中的 `update_task` 实现，使用 notion-tool 的 `update-page` 更新：
- `Sprint` 字段 = 新 Sprint 名称
- `Status` 字段 = "Sprint Todo"

### Step 4：生成 Sprint 目标摘要

询问用户：本 Sprint 的核心目标是什么？（一两句话描述）

将 Sprint 目标摘要更新到 Notion Dashboard Page 的"当前 Sprint 概览"模块（参照 `adapters/notion.md` 的 `update_dashboard` 实现）。

### Step 5：完成输出

统计本 Sprint 的任务数量后输出：

```
✅ Sprint-02 规划完成

📊 本 Sprint 计划：
- 总任务数：N 项
- 工作项目：X 项
- 个人项目：Z 项

Sprint 目标：<用户输入的目标>

Dashboard 已更新。
```
