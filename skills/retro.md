# project-manage:retro

Sprint 回顾。在 Sprint 结束时运行，统计完成情况、记录复盘总结、归档任务。

---

## 执行流程

### Step 1：选择回顾的 Sprint

调用 Notion MCP 的 `retrieve_database` 获取 Sprint select 选项列表，展示未归档的 Sprint（名称不包含 `-archived` 后缀）：

```
请选择要回顾的 Sprint：
1. Sprint-01
2. Sprint-02
```

### Step 2：统计完成情况

调用 Notion MCP 的 `query_database`，查询该 Sprint 的所有任务（filter: sprint = 选中的 Sprint 名称），在内存中统计：
- 总任务数
- 各状态数量：Done / In Progress / In Review / Sprint Todo
- Blocker（Priority = "P0" 或 Notes 包含 "[blocker]" 的任务）

展示统计结果：

```
📊 Sprint-01 完成情况

计划总任务：10 项
✅ 完成（Done）：6 项（60%）
🔄 进行中（In Progress）：2 项
👀 待Review（In Review）：1 项
📋 未开始（Sprint Todo）：1 项

🚨 未完成的 Blocker（P0）：
- [任务名]：<Notes 中的阻塞原因>
```

### Step 3：复盘问题（逐一对话）

**问题 1：**
```
🌟 这个 Sprint 做得好的地方是什么？（可以是多条，用换行分隔，或输入"跳过"）
```

**问题 2：**
```
🔧 哪些地方需要改进？（或输入"跳过"）
```

**问题 3：**
```
📌 下个 Sprint 的具体行动项是什么？（每条将自动添加到 Backlog，用换行分隔，或输入"跳过"）
```

对问题 3 的每条行动项，调用 Notion MCP 的 `create_page` 创建任务：
- Status = "Backlog"
- Priority = "P1"
- Description = "来自 Sprint-01 回顾的行动项"

### Step 4：生成回顾总结 Page

调用 Notion MCP 的 `create_page`，在主工作区 Page 下创建子 Page，标题为 `Sprint-01 回顾（YYYY-MM-DD）`，内容使用 `children` blocks：

Page 内容结构：
1. Heading 1: "Sprint-01 回顾"
2. Paragraph: "Sprint 周期：<开始日期> ~ <结束日期>"
3. Paragraph: "完成率：60%（6/10）"
4. Heading 2: "📊 数据统计"
5. Bulleted list: 完成 6 项、未完成 4 项（其中 Blocker 1 项）
6. Heading 2: "🌟 做得好的地方"
7. Paragraph: <用户输入，若跳过则写"（未填写）"）>
8. Heading 2: "🔧 需要改进的地方"
9. Paragraph: <用户输入，若跳过则写"（未填写）"）>
10. Heading 2: "📌 下 Sprint 行动项"
11. Bulleted list: <每条行动项，注明"已添加到 Backlog"）>

### Step 5：归档 Sprint

询问用户：是否现在归档 Sprint-01 的任务？

```
归档后，Sprint-01 的所有任务 Sprint 字段将变为 "Sprint-01-archived"。
确认归档？（是/否）
```

若确认：
- 调用 Notion MCP 的 `query_database`，获取该 Sprint 的所有任务
- 对每个任务调用 `update_page`，更新 `Sprint` 字段 = "Sprint-01-archived"
- 输出：`✅ Sprint-01 已归档（N 个任务标记为 Sprint-01-archived）`

### Step 6：Dashboard 刷新

调用 Notion MCP 更新 Dashboard Page，清空当前 Sprint 数据（参照 `adapters/notion.md` 的 `update_dashboard` 实现），sprint_overview 部分写入已归档提示。

### Step 7：完成输出

```
✅ Sprint-01 回顾完成

📋 摘要：
- 完成率：60%（6/10）
- 行动项已添加到 Backlog：3 项
- 回顾文档：<Notion Page 链接>
- Sprint-01 已归档（或：跳过归档）

下一步：运行 project-manage:sprint 开始规划下一个 Sprint。
```
