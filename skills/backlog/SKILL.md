---
name: backlog
description: "Backlog 管理。用户说"添加任务"、"新任务"、"看看 backlog"、"调整优先级"时触发。支持新增任务、查看和排序 Backlog。"
---

# agile-project-management:backlog

Backlog 管理。用于新增任务、查看和调整优先级、工作项目同步 GitLab issue。

---

## 前置：读取配置

读取 `~/.claude/agile-pm/config.json`，获取 `notion.task_db_id` 和 `notion.dashboard_page_id`。
若文件不存在，提示用户先运行 `agile-project-management:setup`，然后停止执行。

---

## 执行流程

启动后，展示操作菜单：

```
📋 Backlog 管理，请选择操作：
1. 快速录入（从草稿箱导入）
2. 新增任务（逐项填写）
3. 查看 Backlog（可调整优先级）
4. 退出
```

---

### 操作 1：快速录入（从草稿箱导入）

#### Step 1：查询草稿

参照 `adapters/notion.md` 中的 `list_tasks` 实现，使用 notion-tool 的 `query-database` 查询 Status = "Draft" 的所有任务：

```bash
uv run tools/notion_tool.py query-database '{
  "database_id": "NOTION_TASK_DB_ID",
  "filter": { "property": "Status", "select": { "equals": "Draft" } }
}'
```

#### Step 2：分支处理

**2a. 无草稿时：**

从 `~/.claude/agile-pm/config.json` 读取 `notion.draft_view_url`，输出：

```
📭 草稿箱为空。请前往 Notion 添加任务后再次运行：
<draft_view_url>

提示：在草稿箱视图中直接新增行，只需填写任务名称，其他字段可按需补充。
```

然后返回主菜单。

**2b. 有草稿时：**

进入 Step 3。

#### Step 3：智能补全 & 预览

对每个草稿任务，按以下规则填充空字段：

| 字段 | 默认值逻辑 |
|------|-----------|
| Priority | 未填 → `P1` |
| Type | 未填 → 根据 Project 字段推断（查询已有任务中该 Project 的 Type），兜底 `个人项目` |
| Project | 未填 → 使用最近一次创建的任务的 Project 值（查询 Backlog 中最新一条任务） |
| Description | 未填 → 留空 |
| Estimate | 未填 → 留空 |
| Due Date | 未填 → 留空 |

展示汇总表格：

```
📋 草稿箱待导入（共 N 项）：

  #  │ 任务名称         │ 项目       │ 优先级 │ 类型     │ 工时 │ 截止
  1  │ 实现 TPC-H Q6    │ ssb-tool   │ P1     │ 工作项目 │ 4h   │ 5.30
  2  │ 添加数据校验脚本  │ ssb-tool*  │ P1*    │ 工作项目*│ —    │ —
  3  │ 修复内存溢出      │ spark*     │ P1*    │ 个人项目*│ —    │ —

  * 标记 = 由系统自动补全的默认值

确认导入？（y / 输入序号修改该项 / 取消）
```

- 用户输入 `y` 或 `是` → 进入 Step 4
- 用户输入序号（如 `2`）→ 对该任务逐项询问需要修改的字段（任务名/项目/优先级/类型/工时/截止），修改后重新展示汇总表
- 用户输入 `取消` → 放弃导入，返回主菜单（Draft 任务保留不动）

#### Step 4：批量创建

对每个确认的草稿任务，参照 `adapters/notion.md` 中的 `update_task` 实现，使用 notion-tool 的 `update-page`：

1. 将智能补全的默认值写入对应字段
2. 将 Status 从 `Draft` 更新为 `Backlog`

```bash
uv run tools/notion_tool.py update-page '{
  "page_id": "<草稿任务的 page ID>",
  "properties": {
    "Status": { "select": { "name": "Backlog" } },
    "Priority": { "select": { "name": "P1" } },
    "Type": { "select": { "name": "工作项目" } },
    "Project": { "select": { "name": "ssb-tool" } }
  }
}'
```

完成后输出：

```
✅ 已导入 N 个任务到 Backlog：
  • 实现 TPC-H Q6（P1 | 工作项目 | ssb-tool）
  • 添加数据校验脚本（P1 | 工作项目 | ssb-tool）
  • 修复内存溢出（P1 | 个人项目 | spark-deploy）
```

返回主菜单。

---

### 操作 2：新增任务（逐项填写）

#### Step 1：展示任务模板

用户选择"新增任务"后，先展示完整的字段模板，让用户了解需要填写的全部内容：

```
📝 新建任务 — 请逐项填写以下信息：

┌─────────────────────────────────────┐
│ 1. 任务名称：（必填）                 │
│ 2. 项目类型：个人项目 / 工作项目       │
│ 3. 所属项目：（从已有项目选择或新建）   │
│ 4. 优先级  ：P0 / P1 / P2           │
│ 5. 描述    ：（选填）                 │
│ 6. 预估工时：（选填，单位：小时）      │
│ 7. 截止日期：（选填，格式 YYYY-MM-DD） │
└─────────────────────────────────────┘

开始填写 👇
```

#### Step 2：逐项收集

按以下固定顺序逐项收集信息，每步使用 `AskUserQuestion` 工具：

**2.1 任务名称**（必填，文本输入）
```
[1/7] 任务名称：
```

**2.2 项目类型**（选择）
```
[2/7] 项目类型：
1. 个人项目
2. 工作项目
```

**2.3 所属项目**（选择 + 自定义）

参照 `adapters/notion.md` 中的 `list_tasks` 实现，使用 notion-tool 的 `query-database` 查询所有任务，提取唯一的 Project 值列表：
```
[3/7] 所属项目：
1. ssb-tool
2. daily-market-report
3. spark-deploy
... （已有项目列表）
（选择"Other"可输入新项目名）
```

**2.4 优先级**（选择）
```
[4/7] 优先级：
1. P0（紧急，需立即处理）
2. P1（重要，本 Sprint 内完成）
3. P2（一般，有时间再做）
```

**2.5 描述**（文本输入，可跳过）

提供结构化引导模板帮助用户组织信息：
```
[5/7] 任务描述（可输入"跳过"）：

建议按以下结构填写：
  背景：<为什么要做这件事>
  目标：<期望达成什么>
  验收标准：<怎么算做完>
```

**2.6 预估工时**（文本输入，可跳过）
```
[6/7] 预估工时（小时，可输入"跳过"）：
```

**2.7 截止日期**（文本输入，可跳过）

支持简写格式，自动补全年份：
```
[7/7] 截止日期（可输入"跳过"）：
  支持格式：2026-05-24 或简写 5.24
```

若用户输入简写格式（如 `5.24`），自动补全为当前年份的完整日期 `2026-05-24`。

#### Step 3：确认摘要

所有字段收集完毕后，展示确认摘要：

```
📋 任务确认：

  名称  ：fuyu数据迁移方案验证
  类型  ：工作项目
  项目  ：fuyu迁移支持
  优先级：P1
  描述  ：背景：客户fuyu从阿里云迁移到ucloud...
  工时  ：—
  截止  ：2026-05-24

确认创建？（y / 输入字段名修改该项 / 取消）
```

- 用户输入 `y` 或 `是` → 进入 Step 4 创建任务
- 用户输入字段名（如 `优先级`、`描述`）→ 重新收集该字段，然后再次展示确认摘要
- 用户输入 `取消` → 放弃创建，返回主菜单

#### Step 4：创建任务 + GitLab 关联

参照 `adapters/notion.md` 中的 `create_task` 实现，使用 notion-tool 的 `create-page` 创建 Notion 任务（Status="Backlog"）。

完成后输出：
```
✅ 任务已添加到 Backlog：[任务名]（P1 | 工作项目 | ssb-tool）
```

询问：是否继续添加任务？（是→回到 Step 1 重新开始，否→返回主菜单）

---

### 操作 3：查看 Backlog

参照 `adapters/notion.md` 中的 `list_tasks` 实现，使用 notion-tool 的 `query-database` 查询 Status = "Backlog" 的任务，按 Priority 排序展示：

```
📋 当前 Backlog（共 N 项）：

P0:
1. [任务A]（工作项目 | ssb-tool）- <Description 前50字>

P1:
2. [任务B]（个人项目 | daily-market-report）
3. [任务C]（工作项目 | spark-deploy）

P2:
4. [任务D]（个人项目）
```

展示后询问：
```
是否需要调整某个任务的优先级？（输入任务序号，或输入"否"退出）
```

若用户输入序号，询问新优先级（P0/P1/P2），参照 `adapters/notion.md` 中的 `update_task` 实现，使用 notion-tool 的 `update-page` 更新 `Priority` 字段。

可重复调整，直到用户输入"否"。

---

### 操作 4：退出

结束 skill 执行，不做任何操作。
