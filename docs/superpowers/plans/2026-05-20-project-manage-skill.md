# Project Manage Skill 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一套 Claude Code Plugin，提供敏捷项目管理能力，包含每日站会、Sprint 规划、Backlog 管理、Sprint 回顾和发布管理五大工作流，通过可替换适配器层支持 Notion（当前）和未来的飞书多维表格。

**Architecture:** Plugin 由 `plugin.json` + 6 个 skill 文件 + 2 个适配器文件组成。所有 skill 通过 `adapters/board-interface.md` 定义的抽象操作访问看板，Notion 具体实现在 `adapters/notion.md`。站会状态持久化在本地 `.claude/standup-state.json`，定时触发由 CronCreate 写入 `.claude/scheduled_tasks.json`。

**Tech Stack:** Claude Code Plugin（Markdown skill 文件）、Notion MCP、GitLab MCP（内网自建或社区版）、Claude Code CronCreate API

---

## 文件结构

```
/Users/alex/Workspace/ai-agent/project-manage/
├── README.md                          # 项目说明与快速上手
├── plugin.json                        # Plugin 元数据，注册6个子 skill
├── skills/
│   ├── setup.md                       # 初次配置：Notion MCP + GitLab MCP + 看板创建 + 定时任务
│   ├── standup.md                     # 每日站会：混合模式（用户自改+对话引导）
│   ├── sprint.md                      # Sprint 规划：创建Sprint + 任务分配 + GitLab关联
│   ├── backlog.md                     # Backlog 管理：新增/排序/GitLab同步
│   ├── retro.md                       # Sprint 回顾：统计 + 总结 + 归档
│   └── release.md                     # 发布管理：GitLab milestone + checklist + tag
└── adapters/
    ├── board-interface.md             # 抽象接口规范（10个操作）
    └── notion.md                      # Notion 适配器：MCP调用实现指引
```

---

## Task 1: Plugin 骨架与 README

**Files:**
- Create: `plugin.json`
- Create: `README.md`

- [ ] **Step 1: 创建 plugin.json**

```json
{
  "name": "project-manage",
  "version": "1.0.0",
  "description": "敏捷项目管理工具集：站会、Sprint规划、Backlog、回顾、发布管理",
  "skills": [
    {
      "name": "setup",
      "path": "skills/setup.md",
      "description": "初次配置：Notion MCP、GitLab MCP、看板结构、每日站会定时任务"
    },
    {
      "name": "standup",
      "path": "skills/standup.md",
      "description": "每日站会：混合模式（读取Notion实时状态 + 对话引导补全），支持手动触发和定时触发"
    },
    {
      "name": "sprint",
      "path": "skills/sprint.md",
      "description": "Sprint规划：创建Sprint、从Backlog选任务、关联GitLab issue"
    },
    {
      "name": "backlog",
      "path": "skills/backlog.md",
      "description": "Backlog管理：新增任务、优先级排序、工作项目同步GitLab issue"
    },
    {
      "name": "retro",
      "path": "skills/retro.md",
      "description": "Sprint回顾：完成率统计、复盘总结、归档任务"
    },
    {
      "name": "release",
      "path": "skills/release.md",
      "description": "发布管理：GitLab milestone检查、发布checklist、打tag"
    }
  ]
}
```

- [ ] **Step 2: 创建 README.md**

```markdown
# project-manage

Claude Code Plugin，提供敏捷项目管理能力。作为独立 Scrum Master 使用，支持个人项目和工作项目管理。

## 快速上手

首次使用，运行初始化配置：

```
project-manage:setup
```

配置完成后，每日工作流：

```
project-manage:standup   # 每日站会（默认10:00自动触发）
project-manage:backlog   # 新增或整理任务
project-manage:sprint    # Sprint规划（每个Sprint开始时）
project-manage:retro     # Sprint回顾（每个Sprint结束时）
project-manage:release   # 发布管理
```

## 项目类型

- **个人项目**：仅在 Notion 看板管理
- **工作项目**：Notion 看板 + 内网 GitLab issue 双向关联

## 看板结构

Notion Task Database，5列看板：
`Backlog → Sprint Todo → In Progress → In Review → Done`

## 依赖

- Notion MCP（配置指引见 `project-manage:setup`）
- GitLab MCP（内网直连，配置指引见 `project-manage:setup`）
```

- [ ] **Step 3: 验证文件结构**

```bash
ls -la /Users/alex/Workspace/ai-agent/project-manage/
```

期望输出：`plugin.json` 和 `README.md` 存在。

- [ ] **Step 4: 提交**

```bash
cd /Users/alex/Workspace/ai-agent/project-manage
git init
git add plugin.json README.md
git commit -m "feat: init project-manage plugin skeleton"
```

---

## Task 2: 适配器抽象接口

**Files:**
- Create: `adapters/board-interface.md`

- [ ] **Step 1: 创建 adapters/ 目录并编写 board-interface.md**

```markdown
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
- `sprint`, `priority`, `estimate`, `description`, `gitlab_issue`, `gitlab_mr`, `due_date`, `notes`

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
  "gitlab_mr": "http://gitlab.internal/group/repo/-/merge_requests/15",
  "due_date": "2026-05-25",
  "notes": "站会备注"
}
```
```

- [ ] **Step 2: 验证文件已创建**

```bash
cat /Users/alex/Workspace/ai-agent/project-manage/adapters/board-interface.md | head -5
```

期望输出：`# Board Interface — 抽象看板操作规范`

- [ ] **Step 3: 提交**

```bash
cd /Users/alex/Workspace/ai-agent/project-manage
git add adapters/board-interface.md
git commit -m "feat: add board adapter abstract interface"
```

---

## Task 3: Notion 适配器实现指引

**Files:**
- Create: `adapters/notion.md`

- [ ] **Step 1: 创建 adapters/notion.md**

```markdown
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
```

- [ ] **Step 2: 验证文件已创建**

```bash
cat /Users/alex/Workspace/ai-agent/project-manage/adapters/notion.md | head -5
```

期望输出：`# Notion 适配器实现指引`

- [ ] **Step 3: 提交**

```bash
cd /Users/alex/Workspace/ai-agent/project-manage
git add adapters/notion.md
git commit -m "feat: add notion adapter implementation guide"
```

---

## Task 4: Setup Skill

**Files:**
- Create: `skills/setup.md`

- [ ] **Step 1: 创建 skills/setup.md**

```markdown
# project-manage:setup

初次配置引导。运行此 skill 以完成：
1. Notion MCP 安装与配置
2. GitLab MCP 安装与配置
3. Notion 看板结构创建
4. 每日站会定时任务配置

---

## 执行流程

### 第一步：配置 Notion MCP

询问用户是否已安装 Notion MCP：

**若未安装，引导以下步骤：**

1. 访问 https://www.notion.so/profile/integrations，创建一个 Integration
2. 复制 Integration Token（格式：`secret_xxxxxxxx`）
3. 在 Claude Code 配置中添加 Notion MCP：

```json
// 在 ~/.claude/settings.json 的 mcpServers 中添加：
{
  "notion": {
    "command": "npx",
    "args": ["-y", "@notionhq/notion-mcp-server"],
    "env": {
      "OPENAPI_MCP_HEADERS": "{\"Authorization\": \"Bearer secret_xxxxxxxx\", \"Notion-Version\": \"2022-06-28\"}"
    }
  }
}
```

4. 重启 Claude Code 使配置生效
5. 验证：调用 Notion MCP 的 `search` 工具，确认连接成功

**若已安装，** 跳过安装步骤，直接验证连接。

---

### 第二步：配置 GitLab MCP

询问用户是否已配置 GitLab MCP：

**若未配置，询问以下信息：**
- GitLab 内网地址（如 `http://gitlab.internal`）
- Personal Access Token（需要 `api` 权限，在 GitLab → User Settings → Access Tokens 创建）

**提供两种配置路径：**

**路径 A：社区版 gitlab-mcp（推荐先尝试）**

```json
// 在 ~/.claude/settings.json 的 mcpServers 中添加：
{
  "gitlab": {
    "command": "npx",
    "args": ["-y", "gitlab-mcp"],
    "env": {
      "GITLAB_URL": "http://gitlab.internal",
      "GITLAB_TOKEN": "<your-personal-access-token>"
    }
  }
}
```

**路径 B：若路径 A 不支持内网，自建轻量 MCP**

告知用户："社区版 gitlab-mcp 需要验证是否兼容您的内网 GitLab 版本。若连接失败，可在 project-manage 目录下创建一个自建 MCP server（Python FastMCP），运行 `project-manage:setup` 时会提示并提供脚手架。"

验证：调用 GitLab MCP 列出可访问的项目（`list_projects`），确认连接成功。

---

### 第三步：创建 Notion 看板结构

**3.1 创建 Task Database**

使用 Notion MCP 的 `create_database` 在用户指定的 Notion Page 下创建 Task Database：

询问用户：要在 Notion 的哪个 Page 下创建看板？（请提供 Page ID 或 Page URL）

创建 Database，配置以下字段：
- `Name`（Title，默认存在）
- `Status`（Select）：选项 = `["Backlog", "Sprint Todo", "In Progress", "In Review", "Done"]`，颜色建议：灰、蓝、黄、橙、绿
- `Type`（Select）：选项 = `["个人项目", "工作项目"]`
- `Project`（Select）：初始为空，随用户添加项目动态扩展
- `Sprint`（Select）：初始为空
- `Priority`（Select）：选项 = `["P0", "P1", "P2"]`，颜色建议：红、黄、蓝
- `Estimate`（Number）：格式 = 数字
- `Description`（Rich Text）
- `GitLab Issue`（URL）
- `GitLab MR`（URL）
- `Due Date`（Date）
- `Notes`（Rich Text）

**3.2 创建看板视图**

在 Task Database 上创建4个视图：
1. `看板视图`（Board）：按 Status 分组（主视图）
2. `工作项目`（Table）：过滤 Type = "工作项目"
3. `个人项目`（Table）：过滤 Type = "个人项目"
4. `当前Sprint`（Table）：过滤 Sprint = 当前 Sprint 编号（初始为空，每次 Sprint 规划后手动更新）

**3.3 创建 Dashboard Page**

在同一 Notion Page 下创建名为 "📊 项目管理 Dashboard" 的子 Page，写入初始内容：

```
📊 项目管理 Dashboard
（首次运行站会后自动填充）
```

**3.4 记录配置**

将以下配置告知用户，建议记录在 `~/.claude/settings.json` 的 `env` 中或项目本地配置：

```
NOTION_TASK_DB_ID=<刚创建的 Task Database ID>
NOTION_DASHBOARD_PAGE_ID=<刚创建的 Dashboard Page ID>
BOARD_BACKEND=notion
```

---

### 第四步：配置每日站会定时任务

使用 Claude Code 的 `CronCreate` 工具创建定时任务：

```
CronCreate({
  cron: "0 10 * * *",        // 每天 10:00
  prompt: "project-manage:standup",
  durable: true,             // 持久化，重启后保留
  recurring: true
})
```

告知用户：每日 10:00 会自动触发站会。若当日已手动完成站会，定时触发会自动跳过。

---

### 完成确认

Setup 完成后，输出配置摘要：

```
✅ 配置完成

- Notion MCP：已连接
- GitLab MCP：已连接（http://gitlab.internal）
- Task Database：已创建（ID: xxx）
- Dashboard Page：已创建（ID: xxx）
- 每日站会：每天 10:00 自动触发

开始使用：
- 添加任务：project-manage:backlog
- 规划Sprint：project-manage:sprint
- 每日站会：project-manage:standup
```
```

- [ ] **Step 2: 验证文件已创建**

```bash
wc -l /Users/alex/Workspace/ai-agent/project-manage/skills/setup.md
```

期望输出：行数 > 50

- [ ] **Step 3: 提交**

```bash
cd /Users/alex/Workspace/ai-agent/project-manage
git add skills/setup.md
git commit -m "feat: add setup skill - Notion/GitLab MCP config and board init"
```

---

## Task 5: Standup Skill（最高优先级）

**Files:**
- Create: `skills/standup.md`

- [ ] **Step 1: 创建 skills/standup.md**

```markdown
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

使用 Notion MCP 调用 `query_database`，读取以下任务：
- Status = "In Progress"
- Status = "Sprint Todo"（当前 Sprint）

同时读取 `~/.claude/standup-state.json` 中的 `last_snapshot`（上次站会快照）。

### Step 2：差异识别

对比当前数据与 `last_snapshot`：

- **已变更的任务**：用户已自行在 Notion 更新（状态变化或 Notes 有新内容）
  - 直接展示变更，无需再问：
    ```
    ✅ 已检测到您自行更新的任务：
    - [任务名]：In Progress → Done
    - [任务名]：Notes 有更新
    ```

- **未变更的 In Progress 任务**：需要对话引导（见 Step 3）

- **新增任务**（快照中不存在）：直接确认已知晓

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

根据用户选择：
- 选 1：调用 `move_task(id, "Done")`
- 选 2：若用户有补充说明，调用 `update_task(id, { notes: 原notes + "\n" + 日期 + ": " + 说明 })`
- 选 3：调用 `update_task(id, { priority: "P0", notes: 原notes + "\n" + 日期 + ": [blocker] " + 原因 })`

### Step 4：今日计划

展示当前 Sprint 的 "Sprint Todo" 任务列表，询问：

```
📋 Sprint Backlog（待处理）：
1. [任务A]（P1 | 个人项目 | 预估2h）
2. [任务B]（P0 | 工作项目 | 预估4h）
...

今日计划推进哪些任务？（输入序号，多个用逗号分隔，或输入"跳过"）
```

对选中的任务调用 `move_task(id, "In Progress")`。

### Step 5：Blocker 汇总

若有新增 Blocker（Step 3 中用户报告的），输出摘要：

```
🚨 今日 Blocker：
- [任务名]：<阻塞原因>
```

并询问：是否需要在 GitLab 对应 issue 上添加评论说明阻塞情况？（工作项目且有 GitLab Issue 链接时才询问）

若用户确认，调用 GitLab MCP 的 `create_note`（issue comment）。

### Step 6：GitLab 状态同步

对本次站会中状态发生变更的工作项目任务：
- 任务移至 Done → 调用 GitLab MCP `update_issue(project, issue_id, { state: "close" })`
- 任务从 Sprint Todo 移至 In Progress → 调用 GitLab MCP `update_issue(project, issue_id, { state: "reopen" })`（确保 issue 是 opened 状态）

GitLab issue ID 从任务的 `gitlab_issue` URL 中提取（取最后一段数字）。

### Step 7：生成今日站会摘要

自动生成一行摘要（用于 Dashboard 展示），格式：

```
YYYY-MM-DD: 完成 X 项，推进 Y 项，阻塞 Z 项
```

### Step 8：Dashboard 刷新

1. 调用 `get_sprint_stats(当前Sprint名称)` 获取最新统计
2. 读取现有 Dashboard 中的 `recent_standups`（最多保留最近3条）
3. 将今日摘要插入到最前，超出3条则丢弃最旧的
4. 调用 `update_dashboard(sections)` 刷新 Dashboard

### Step 9：持久化状态

将当前状态写入 `~/.claude/standup-state.json`：

```json
{
  "last_done": "今日日期（YYYY-MM-DD）",
  "last_snapshot": {
    "<task-id>": {
      "status": "<当前status>",
      "notes": "<当前notes>"
    }
  }
}
```

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
```

- [ ] **Step 2: 验证文件已创建**

```bash
wc -l /Users/alex/Workspace/ai-agent/project-manage/skills/standup.md
```

期望输出：行数 > 80

- [ ] **Step 3: 提交**

```bash
cd /Users/alex/Workspace/ai-agent/project-manage
git add skills/standup.md
git commit -m "feat: add standup skill - hybrid mode daily standup"
```

---

## Task 6: Sprint Skill

**Files:**
- Create: `skills/sprint.md`

- [ ] **Step 1: 创建 skills/sprint.md**

```markdown
# project-manage:sprint

Sprint 规划。在每个 Sprint 开始时运行，完成 Sprint 创建、任务分配和 GitLab 关联。

---

## 执行流程

### Step 1：确认当前 Sprint 状态

调用 `list_sprints()` 获取 Sprint 列表，展示给用户：

```
当前 Sprint：
- Sprint-01（2026-05-18 ~ 2026-05-31）[进行中]

历史 Sprint：
- Sprint-00（2026-05-01 ~ 2026-05-17）[已归档]
```

询问：要创建新 Sprint，还是查看/调整当前 Sprint 的任务？

- 选择**创建新 Sprint**：继续 Step 2
- 选择**调整当前 Sprint**：跳至 Step 4

### Step 2：创建新 Sprint

收集信息：
- Sprint 名称：自动建议下一个编号（如当前最大是 Sprint-01，建议 Sprint-02），用户可修改
- 开始日期：默认为今天
- 结束日期：默认为开始日期 + 14 天（两周 Sprint），用户可修改

调用 `create_sprint(name, dates)`。

输出：
```
✅ Sprint-02 已创建（2026-06-01 ~ 2026-06-14）
```

### Step 3：从 Backlog 选取任务

调用 `list_tasks({ status: "Backlog" })`，按 Priority 排序展示：

```
📋 Backlog（共 N 项，按优先级排序）：

P0:
1. [任务A]（工作项目 | ssb-tool | 预估4h）- <描述摘要>

P1:
2. [任务B]（个人项目 | daily-market-report | 预估2h）- <描述摘要>
3. [任务C]（工作项目 | spark-deploy | 预估8h）- <描述摘要>

P2:
4. [任务D]（个人项目 | 预估1h）- <描述摘要>

请选择本 Sprint 要纳入的任务（输入序号，多个用逗号分隔）：
```

对选中任务调用 `update_task(id, { sprint: "Sprint-02", status: "Sprint Todo" })`。

### Step 4：检查工作项目的 GitLab 关联

对本 Sprint 内所有 `type = "工作项目"` 且 `gitlab_issue` 为空的任务，逐一提示：

```
⚠️  [任务名] 是工作项目，但未关联 GitLab issue。
选择：
1. 现在创建 GitLab issue（需要提供 GitLab 项目路径）
2. 输入已有 issue 链接
3. 跳过（此任务不需要关联）
```

**选 1：创建 GitLab issue**
- 询问 GitLab 项目路径（如 `group/repo`）
- 调用 GitLab MCP `create_issue`：
  - `title`: 任务名称
  - `description`: 任务的 `description` 字段内容
- 将返回的 issue URL 写入 `update_task(id, { gitlab_issue: url })`

**选 2：** 直接写入 `update_task(id, { gitlab_issue: 用户输入的url })`

### Step 5：生成 Sprint 目标摘要

询问用户：本 Sprint 的核心目标是什么？（一两句话描述）

将目标摘要通过 `update_dashboard` 更新到 Dashboard 的 Sprint 概览模块。

### Step 6：完成输出

```
✅ Sprint-02 规划完成

📊 本 Sprint 计划：
- 总任务数：N 项
- 工作项目：X 项（已关联 GitLab issue Y 项）
- 个人项目：Z 项

Sprint 目标：<用户输入的目标>

Dashboard 已更新。
```
```

- [ ] **Step 2: 提交**

```bash
cd /Users/alex/Workspace/ai-agent/project-manage
git add skills/sprint.md
git commit -m "feat: add sprint skill - sprint planning and GitLab linking"
```

---

## Task 7: Backlog Skill

**Files:**
- Create: `skills/backlog.md`

- [ ] **Step 1: 创建 skills/backlog.md**

```markdown
# project-manage:backlog

Backlog 管理。用于新增任务、查看和调整优先级、工作项目同步 GitLab issue。

---

## 执行流程

启动后，展示操作菜单：

```
📋 Backlog 管理，请选择操作：
1. 新增任务
2. 查看 Backlog（可调整优先级）
3. 退出
```

---

### 操作 1：新增任务

逐步收集信息（每步一个问题）：

**1.1 任务名称**
```
任务名称是什么？
```

**1.2 项目类型**
```
这是哪种类型的项目？
1. 个人项目
2. 工作项目
```

**1.3 所属项目**

调用 `list_tasks({})` 获取已有的 Project 列表，展示供选择：
```
所属项目：
1. ssb-tool
2. daily-market-report
3. spark-deploy
... （已有项目列表）
N. 新项目（输入项目名）
```

**1.4 优先级**
```
优先级：
1. P0（紧急，需立即处理）
2. P1（重要，本 Sprint 内完成）
3. P2（一般，有时间再做）
```

**1.5 详细描述**
```
请描述任务详情（背景、目标、验收标准等，可直接回车跳过）：
```

**1.6 预估工时**
```
预估工时（小时，可直接回车跳过）：
```

**1.7 截止日期**
```
截止日期（格式 YYYY-MM-DD，可直接回车跳过）：
```

**1.8 工作项目额外询问：是否同步创建 GitLab issue？**

仅当 type = "工作项目" 时询问：
```
是否同步在 GitLab 创建对应 issue？
1. 是（需要提供 GitLab 项目路径）
2. 否（稍后手动关联）
```

若选是：
- 询问 GitLab 项目路径（如 `group/repo`）
- 调用 `create_task(fields)` 先创建 Notion 任务
- 调用 GitLab MCP `create_issue(project, title, description)`
- 调用 `update_task(id, { gitlab_issue: 返回的issue_url })`

若选否：直接调用 `create_task(fields)`。

完成后输出：
```
✅ 任务已添加到 Backlog：[任务名]（P1 | 工作项目 | ssb-tool）
```

询问：是否继续添加任务？

---

### 操作 2：查看 Backlog

调用 `list_tasks({ status: "Backlog" })`，按 Priority 排序展示。

展示后询问：是否需要调整某个任务的优先级？

若是，引导用户输入任务序号和新优先级，调用 `update_task(id, { priority: 新优先级 })`。

---

### 操作 3：退出

结束 skill 执行。
```

- [ ] **Step 2: 提交**

```bash
cd /Users/alex/Workspace/ai-agent/project-manage
git add skills/backlog.md
git commit -m "feat: add backlog skill - task creation and priority management"
```

---

## Task 8: Retro Skill

**Files:**
- Create: `skills/retro.md`

- [ ] **Step 1: 创建 skills/retro.md**

```markdown
# project-manage:retro

Sprint 回顾。在 Sprint 结束时运行，统计完成情况、记录复盘总结、归档任务。

---

## 执行流程

### Step 1：选择回顾的 Sprint

调用 `list_sprints()`，展示未归档的 Sprint：

```
请选择要回顾的 Sprint：
1. Sprint-01（2026-05-18 ~ 2026-05-31）
```

### Step 2：统计完成情况

调用 `get_sprint_stats(sprint_name)`，展示统计结果：

```
📊 Sprint-01 完成情况

计划总任务：10 项
✅ 完成（Done）：6 项（60%）
🔄 进行中（In Progress）：2 项
👀 待Review（In Review）：1 项
📋 未开始（Sprint Todo）：1 项

🚨 未完成的 Blocker（P0）：
- [任务名]：<阻塞原因>
```

### Step 3：复盘问题（逐一对话）

**问题 1：**
```
🌟 这个 Sprint 做得好的地方是什么？（可以是多条，用换行分隔）
```

**问题 2：**
```
🔧 哪些地方需要改进？
```

**问题 3：**
```
📌 下个 Sprint 的具体行动项是什么？（可以是多条，每条将自动添加到 Backlog）
```

对问题 3 的每条行动项，调用 `create_task({ name: 行动项, status: "Backlog", priority: "P1", ... })`。

### Step 4：生成回顾总结 Page

在 Notion 中创建新 Page（在主 Page 下），标题为 `Sprint-01 回顾（YYYY-MM-DD）`，内容：

```markdown
# Sprint-01 回顾

**Sprint 周期：** 2026-05-18 ~ 2026-05-31
**完成率：** 60%（6/10）

## 📊 数据统计
- 完成：6 项
- 未完成：4 项（其中 Blocker 1 项）

## 🌟 做得好的地方
<用户输入>

## 🔧 需要改进的地方
<用户输入>

## 📌 下 Sprint 行动项
- <行动项1>（已添加到 Backlog）
- <行动项2>（已添加到 Backlog）
```

### Step 5：归档 Sprint

询问用户：是否现在归档 Sprint-01 的任务？

若确认，调用 `archive_sprint("Sprint-01")`。

输出：
```
✅ Sprint-01 已归档（N 个任务标记为 Sprint-01-archived）
```

### Step 6：Dashboard 刷新

调用 `update_dashboard` 更新 Dashboard，清空当前 Sprint 数据（因为已归档）。

### Step 7：完成输出

```
✅ Sprint-01 回顾完成

📋 摘要：
- 完成率：60%（6/10）
- 行动项已添加到 Backlog：3 项
- 回顾文档：<Notion Page 链接>
- Sprint-01 已归档

下一步：运行 project-manage:sprint 开始规划 Sprint-02。
```
```

- [ ] **Step 2: 提交**

```bash
cd /Users/alex/Workspace/ai-agent/project-manage
git add skills/retro.md
git commit -m "feat: add retro skill - sprint retrospective and archiving"
```

---

## Task 9: Release Skill

**Files:**
- Create: `skills/release.md`

- [ ] **Step 1: 创建 skills/release.md**

```markdown
# project-manage:release

发布管理。检查 GitLab milestone 进度，生成发布 checklist，确认后打 tag。

---

## 执行流程

### Step 1：选择项目和 milestone

询问要发布哪个工作项目（展示有 GitLab 关联的项目列表）：

```
请选择要发布的工作项目：
1. ssb-tool
2. spark-deploy
```

询问 GitLab 项目路径（如 `group/repo`，可从历史记录中自动填充）。

调用 GitLab MCP `list_milestones(project)`，展示可用 milestone：

```
可用 Milestone：
1. v1.0.0（截止：2026-05-31）- 8/10 issues 已关闭
2. v0.9.1（截止：已过期）- 5/5 issues 已关闭
```

用户选择目标 milestone。

### Step 2：检查 milestone 进度

调用 GitLab MCP `list_issues(project, { milestone: milestone_name })`，展示所有 issue 状态：

```
📋 Milestone v1.0.0 进度（8/10 已完成）

✅ 已关闭：
- #42 [任务A]
- #43 [任务B]
...（8项）

🔴 未完成（阻塞发布）：
- #50 [任务C]（assignee: alex | P0）
- #51 [任务D]（assignee: alex | P1）
```

若有未完成 issue，询问：
```
有 2 个 issue 未完成。
1. 等待完成后再发布
2. 关闭这些 issue 并在发布说明中注明（手动跳过）
3. 取消发布
```

### Step 3：生成发布 Checklist

```
📋 发布 Checklist - v1.0.0

代码准备：
[ ] 所有功能分支已合并到主分支
[ ] CI/CD 流水线全部通过
[ ] 代码已 Review（相关 MR 已 merged）

测试：
[ ] 功能测试已完成
[ ] 回归测试通过

文档：
[ ] CHANGELOG 已更新
[ ] README 版本号已更新

发布：
[ ] 发布说明已准备好
```

逐项询问用户确认（输入 y/n 或直接回车跳过），记录未完成项。

### Step 4：确认发布

展示发布摘要：

```
🚀 准备发布 v1.0.0

GitLab 项目：group/repo
Tag 名称：v1.0.0（可修改）
Tag 基于：main 分支（可修改）

发布说明：
<从用户输入或自动从 milestone issue 生成>

确认发布？（y/n）
```

### Step 5：执行发布

用户确认后：

1. 调用 GitLab MCP `create_tag(project, "v1.0.0", "main")`
2. 在 Notion 中更新关联任务状态为 Done（若存在关联任务）
3. 在 Notion Dashboard 记录发布事件

输出：
```
✅ v1.0.0 已发布

- GitLab Tag：已创建（v1.0.0 @ main）
- 关联 Notion 任务：已更新为 Done
- Dashboard：已记录发布事件

GitLab Release 页面：http://gitlab.internal/group/repo/-/releases/v1.0.0
```
```

- [ ] **Step 2: 提交**

```bash
cd /Users/alex/Workspace/ai-agent/project-manage
git add skills/release.md
git commit -m "feat: add release skill - milestone check and tag creation"
```

---

## Task 10: 集成验证

**Files:** 无新文件，验证所有组件协同工作。

- [ ] **Step 1: 验证文件结构完整**

```bash
find /Users/alex/Workspace/ai-agent/project-manage -type f | sort
```

期望输出（至少包含以下文件）：
```
.../README.md
.../plugin.json
.../adapters/board-interface.md
.../adapters/notion.md
.../skills/setup.md
.../skills/standup.md
.../skills/sprint.md
.../skills/backlog.md
.../skills/retro.md
.../skills/release.md
```

- [ ] **Step 2: 验证 plugin.json 格式正确**

```bash
python3 -c "import json; json.load(open('/Users/alex/Workspace/ai-agent/project-manage/plugin.json')); print('JSON valid')"
```

期望输出：`JSON valid`

- [ ] **Step 3: 验证 plugin 注册到 Claude Code**

在 Claude Code 设置中确认 plugin 路径已配置。在对话中检查 skill 是否出现在可用列表：

```
project-manage:setup
project-manage:standup
project-manage:sprint
project-manage:backlog
project-manage:retro
project-manage:release
```

- [ ] **Step 4: 运行 setup skill 端到端测试**

调用 `project-manage:setup`，验证：
- Notion MCP 连接成功
- GitLab MCP 连接成功
- Task Database 和4个视图创建成功
- Dashboard Page 创建成功
- `~/.claude/scheduled_tasks.json` 中出现 10:00 的 cron 任务

- [ ] **Step 5: 运行 standup skill 端到端测试**

手动调用 `project-manage:standup`，验证：
- 前置检查逻辑正确（首次运行无 `standup-state.json` 时正常执行）
- Notion 任务正确拉取
- 交互引导正常
- `~/.claude/standup-state.json` 在完成后正确写入 `last_done` 和 `last_snapshot`
- 再次调用时出现"今日已完成"提示

- [ ] **Step 6: 最终提交**

```bash
cd /Users/alex/Workspace/ai-agent/project-manage
git log --oneline
```

期望：看到10次提交历史，每个 Task 对应一次。

---

## 验证顺序

1. **Task 10 Step 4**：setup 端到端（依赖 Notion/GitLab MCP 已配置）
2. **Task 10 Step 5**：standup 端到端（依赖 setup 已完成）
3. **sprint/backlog/retro/release**：在有真实数据后逐一验证
