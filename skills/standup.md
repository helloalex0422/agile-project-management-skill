# project-manage:standup

每日站会。支持混合模式：Claude 读取 Notion 实时状态，用户已自行更新的任务直接采纳，仅对未处理任务进行对话引导。

---

## 前置检查：是否已完成今日站会

读取 `~/.claude/standup-state.json`：

```bash
cat ~/.claude/standup-state.json
```

若文件存在且 `last_done` 等于今日日期（`YYYY-MM-DD`），输出：

```
✅ 今日站会已于早些时候完成，无需重复进行。
如需重新开始，请删除或更新 ~/.claude/standup-state.json 中的 last_done 字段。
```

然后停止，不继续执行。

---

## 站会流程

### Step 1：读取当前 Notion 状态

参照 `adapters/notion.md` 中的 `list_tasks` 实现，调用 Notion MCP 的 `query_database`，读取以下任务：
- Status = "In Progress"
- Status = "Sprint Todo"（当前 Sprint）

同时读取 `~/.claude/standup-state.json` 中的 `last_snapshot`（上次站会快照）。若文件不存在，视为首次运行，`last_snapshot` 为空。

### Step 2：差异识别

对比当前 Notion 数据与 `last_snapshot`：

**已变更的任务**（当前状态或 notes 与快照不同）：
- 直接展示变更，无需再询问：
  ```
  ✅ 已检测到您自行更新的任务：
  - [任务名]：In Progress → Done
  - [任务名]：Notes 有更新
  ```

**未变更的 In Progress 任务**（快照中 status=In Progress，当前无变化）：
- 需要对话引导（见 Step 3）

**新增任务**（快照中不存在的任务）：
- 直接展示，确认已知晓

### Step 3：对话引导未更新的 In Progress 任务

对每个上次快照中 Status = "In Progress" 且本次无变更的任务，逐一询问：

```
📌 [任务名]（Sprint-01 | P1 | 工作项目）
昨日状态：In Progress
请选择：
1. 已完成（移至 Done）
2. 继续进行（保持 In Progress，有进展可简要说明）
3. 有阻塞（记录阻塞原因，Priority 升级为 P0）
4. 跳过（不做更改）
```

根据用户选择，调用 Notion MCP 的 `update_page` 更新对应任务：
- 选 1：更新 Status = "Done"
- 选 2：若用户有补充说明，追加到 Notes（原内容 + "\n" + 今日日期 + ": " + 说明）
- 选 3：更新 Priority = "P0"，追加到 Notes（原内容 + "\n" + 今日日期 + ": [blocker] " + 原因）

### Step 4：今日计划

展示当前 Sprint 的 "Sprint Todo" 任务列表，询问：

```
📋 Sprint Backlog（待处理）：
1. [任务A]（P1 | 个人项目 | 预估2h）
2. [任务B]（P0 | 工作项目 | 预估4h）
...

今日计划推进哪些任务？（输入序号，多个用逗号分隔，或输入"跳过"）
```

对用户选中的任务，调用 Notion MCP 的 `update_page` 更新 Status = "In Progress"。

### Step 5：Blocker 汇总

若 Step 3 中有新增 Blocker，输出摘要：

```
🚨 今日 Blocker：
- [任务名]：<阻塞原因>
```

对 type = "工作项目" 且有 GitLab Issue 链接的 Blocker 任务，询问：是否需要在 GitLab issue 上添加评论说明阻塞情况？

若用户确认，从 `gitlab_issue` URL 中提取 issue ID（URL 最后一段数字），调用 GitLab MCP 的 `create_note` 工具添加评论。

### Step 6：GitLab 状态同步

对本次站会中状态发生变更的工作项目任务（type = "工作项目" 且有 gitlab_issue 链接）：
- 任务移至 Done → 调用 GitLab MCP `update_issue`，设置 `state_event = "close"`
- 任务从 Sprint Todo 移至 In Progress → 调用 GitLab MCP `update_issue`，设置 `state_event = "reopen"`

从 `gitlab_issue` URL 中提取 issue ID（取 URL 路径最后一段数字），以及项目路径（URL 中 `/issues/` 前的 `group/repo` 部分）。

### Step 7：生成今日站会摘要

统计本次站会的结果，生成一行摘要：

```
YYYY-MM-DD: 完成 X 项，推进 Y 项，阻塞 Z 项
```

### Step 8：Dashboard 刷新

1. 调用 Notion MCP 查询当前 Sprint 所有任务，统计各状态数量（参照 `adapters/notion.md` 的 `get_sprint_stats` 实现）
2. 从 Dashboard Page 读取现有 `recent_standups`（最多保留最近3条，通过读取 Dashboard page blocks 提取）
3. 将今日摘要插入最前，超出3条则丢弃最旧的
4. 更新 Dashboard Page（参照 `adapters/notion.md` 的 `update_dashboard` 实现：删除旧 blocks，append 新内容）

Dashboard 结构：
1. Heading 1: "📊 项目管理 Dashboard"
2. Heading 2: "🏃 当前 Sprint 概览" + Paragraph（Sprint名称、日期、完成率）
3. Heading 2: "📋 任务状态分布" + Bulleted list（各状态数量）
4. Heading 2: "🚨 Blocker 列表" + Bulleted list（Priority=P0的任务）
5. Heading 2: "📁 各项目进度" + Bulleted list（按Project分组）
6. Heading 2: "📅 最近站会记录" + Bulleted list（最近3次摘要）

### Step 9：持久化状态

将当前状态写入 `~/.claude/standup-state.json`：

```json
{
  "last_done": "今日日期（YYYY-MM-DD格式）",
  "last_snapshot": {
    "<task-id>": {
      "status": "<当前status>",
      "notes": "<当前notes>"
    }
  }
}
```

快照应包含当前所有 In Progress 和 Done 的任务（用于下次站会的差异比较）。

---

## 站会完成输出

```
✅ 今日站会完成（YYYY-MM-DD）

📊 站会摘要：
- 昨日完成：X 项（自行更新 A 项 + 本次确认 B 项）
- 今日计划：Y 项
- 阻塞：Z 项

Dashboard 已更新。定时站会今日不再触发。
```
