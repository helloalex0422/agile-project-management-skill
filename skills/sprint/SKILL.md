---
name: sprint
description: "Sprint 规划。用户说"新 Sprint"、"规划 Sprint"、"Sprint planning"时触发。创建新 Sprint、从 Backlog 选取任务、检查 GitLab 关联、设定 Sprint 目标。"
---

# agile-project-management:sprint

Sprint 规划。在每个 Sprint 开始时运行，完成 Sprint 创建、任务分配和 GitLab 关联。

---

## 执行流程

### Step 1：确认当前 Sprint 状态

调用 Notion MCP 的 `retrieve_database` 获取 Sprint select 选项列表，展示给用户（已归档的 Sprint 名称包含 `-archived` 后缀）：

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

调用 Notion MCP 的 `update_database`，向 Task Database 的 `Sprint` 属性添加新的 select 选项（即 Sprint 名称字符串）。

输出：
```
✅ Sprint-02 已创建（2026-06-01 ~ 2026-06-14）
```

### Step 3：从 Backlog 选取任务

调用 Notion MCP 的 `query_database`，查询 Status = "Backlog" 的任务，按 Priority 排序展示：

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

对用户选中的任务，调用 Notion MCP 的 `update_page`，更新：
- `Sprint` 字段 = 新 Sprint 名称
- `Status` 字段 = "Sprint Todo"

### Step 4：检查工作项目的 GitLab 关联

调用 Notion MCP 的 `query_database`，查询当前 Sprint 内所有 `type = "工作项目"` 且 `gitlab_issue` 为空的任务，逐一提示：

```
⚠️  [任务名] 是工作项目，但未关联 GitLab issue。
选择：
1. 现在创建 GitLab issue
2. 输入已有 issue 链接
3. 跳过（此任务不需要关联）
```

**选 1：创建 GitLab issue**
- 询问 GitLab 项目路径（如 `group/repo`）
- 调用 GitLab MCP `create_issue`，title = 任务名称，description = 任务的 Description 字段内容
- 调用 Notion MCP `update_page`，将返回的 issue URL 写入 `GitLab Issue` 字段

**选 2：**
- 调用 Notion MCP `update_page`，将用户输入的 URL 写入 `GitLab Issue` 字段

### Step 5：生成 Sprint 目标摘要

询问用户：本 Sprint 的核心目标是什么？（一两句话描述）

将 Sprint 目标摘要更新到 Notion Dashboard Page 的"当前 Sprint 概览"模块（参照 `adapters/notion.md` 的 `update_dashboard` 实现）。

### Step 6：完成输出

统计本 Sprint 的任务数量后输出：

```
✅ Sprint-02 规划完成

📊 本 Sprint 计划：
- 总任务数：N 项
- 工作项目：X 项（已关联 GitLab issue Y 项）
- 个人项目：Z 项

Sprint 目标：<用户输入的目标>

Dashboard 已更新。
```
