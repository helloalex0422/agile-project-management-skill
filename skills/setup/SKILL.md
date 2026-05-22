---
name: setup
description: "首次配置向导。初始化敏捷项目管理环境：配置 Notion API Key、创建 Notion 看板数据库和 Dashboard、设置每日站会定时任务。首次使用时运行。"
---

# agile-project-management:setup

初次配置引导。运行此 skill 以完成：
1. Notion API Key 配置
2. Notion 看板结构创建
3. 每日站会定时任务配置

---

## 执行流程

### 第一步：配置 Notion API Key

询问用户是否已有 Notion Integration Token：

**若没有，引导以下步骤：**

1. 访问 https://www.notion.so/profile/integrations，创建一个 Integration
2. 复制 Integration Token（格式：`ntn_xxxxxxxx` 或 `secret_xxxxxxxx`）
3. 在 Notion 中，将 Integration 添加到目标工作区页面（页面右上角 → 连接 → 选择刚创建的 Integration）

**获取到 Token 后：**

1. 创建配置目录和文件：

```bash
mkdir -p ~/.claude/agile-pm
```

2. 将 Token 临时记录（后续第三步写入完整配置文件时一并保存）
3. 验证连接：调用 notion-tool 的 `search` 命令确认 Token 有效

```bash
NOTION_API_KEY=<用户提供的Token> uv run tools/notion_tool.py search
```

若返回结果正常（包含 `results` 数组），说明连接成功。若报错，提示用户检查 Token 和 Integration 权限。

---

### 第二步：创建 Notion 看板结构

**2.1 创建 Task Database**

询问用户：要在 Notion 的哪个 Page 下创建看板？（请提供 Page ID 或 Page URL）

调用 notion-tool 的 `create-database` 创建 Task Database：

```bash
uv run tools/notion_tool.py create-database '{
  "parent": { "type": "page_id", "page_id": "<用户指定的 Page ID>" },
  "title": [{ "text": { "content": "Task Board" } }],
  "properties": {
    "Name": { "title": {} },
    "Status": {
      "select": {
        "options": [
          { "name": "Draft", "color": "gray" },
          { "name": "Backlog", "color": "default" },
          { "name": "Sprint Todo", "color": "blue" },
          { "name": "In Progress", "color": "yellow" },
          { "name": "In Review", "color": "orange" },
          { "name": "Done", "color": "green" }
        ]
      }
    },
    "Type": {
      "select": {
        "options": [
          { "name": "个人项目" },
          { "name": "工作项目" }
        ]
      }
    },
    "Project": { "select": { "options": [] } },
    "Sprint": { "select": { "options": [] } },
    "Priority": {
      "select": {
        "options": [
          { "name": "P0", "color": "red" },
          { "name": "P1", "color": "yellow" },
          { "name": "P2", "color": "blue" }
        ]
      }
    },
    "Estimate": { "number": {} },
    "Description": { "rich_text": {} },
    "GitLab Issue": { "url": {} },
    "Due Date": { "date": {} },
    "Notes": { "rich_text": {} }
  }
}'
```

从返回结果中获取 `id` 即为 Task Database ID。

**2.2 创建看板视图**

在 Task Database 上创建5个视图：
1. `看板视图`（Board）：按 Status 分组（主视图）
2. `工作项目`（Table）：过滤 Type = "工作项目"
3. `个人项目`（Table）：过滤 Type = "个人项目"
4. `当前Sprint`（Table）：过滤 Sprint = 当前 Sprint 编号（初始为空，每次 Sprint 规划后手动更新）
5. `草稿箱`（Table）：过滤 Status = "Draft"，用于快速批量录入任务

提示用户手动创建「草稿箱」视图：

```
请在 Notion 的 Task Board 数据库中创建一个新视图：
1. 点击数据库左上角的 "+" 添加视图
2. 视图名称：草稿箱
3. 视图类型：Table
4. 添加 Filter：Status is Draft
5. 创建后，复制该视图的 URL（点击视图标签 → 在浏览器地址栏复制完整 URL）
```

询问用户粘贴草稿箱视图的 URL，保存到配置中（见 3.4）。

**2.3 创建 Dashboard Page（完整模板）**

在同一 Notion Page 下创建名为 "📊 项目管理 Dashboard" 的子 Page，并写入完整的 Dashboard 结构：

```bash
uv run tools/notion_tool.py create-page '{
  "parent": { "type": "page_id", "page_id": "<用户指定的 Parent Page ID>" },
  "properties": {
    "title": { "title": [{ "text": { "content": "📊 项目管理 Dashboard" } }] }
  },
  "children": [
    { "heading_2": { "rich_text": [{ "text": { "content": "🏃 当前 Sprint 概览" } }] } },
    { "paragraph": { "rich_text": [{ "text": { "content": "尚未创建 Sprint，请运行 agile-project-management:sprint 开始规划" } }] } },
    { "heading_2": { "rich_text": [{ "text": { "content": "📋 任务状态分布" } }] } },
    { "bulleted_list_item": { "rich_text": [{ "text": { "content": "Backlog: 0" } }] } },
    { "bulleted_list_item": { "rich_text": [{ "text": { "content": "Sprint Todo: 0" } }] } },
    { "bulleted_list_item": { "rich_text": [{ "text": { "content": "In Progress: 0" } }] } },
    { "bulleted_list_item": { "rich_text": [{ "text": { "content": "In Review: 0" } }] } },
    { "bulleted_list_item": { "rich_text": [{ "text": { "content": "Done: 0" } }] } },
    { "heading_2": { "rich_text": [{ "text": { "content": "🚨 Blocker 列表" } }] } },
    { "paragraph": { "rich_text": [{ "text": { "content": "暂无阻塞项" } }] } },
    { "heading_2": { "rich_text": [{ "text": { "content": "📁 各项目进度" } }] } },
    { "paragraph": { "rich_text": [{ "text": { "content": "暂无项目数据" } }] } },
    { "heading_2": { "rich_text": [{ "text": { "content": "📅 最近站会记录" } }] } },
    { "paragraph": { "rich_text": [{ "text": { "content": "尚未进行站会，请运行 agile-project-management:standup" } }] } }
  ]
}'
```

从返回结果中获取 `id` 即为 Dashboard Page ID。

**2.4 写入配置文件**

将以下配置写入 `~/.claude/agile-pm/config.json`：

```json
{
  "board_backend": "notion",
  "notion": {
    "api_key": "<用户提供的 Notion Integration Token>",
    "task_db_id": "<刚创建的 Task Database ID>",
    "dashboard_page_id": "<刚创建的 Dashboard Page ID>",
    "parent_page_id": "<用户指定的 Parent Page ID>",
    "draft_view_url": "<用户提供的草稿箱视图 URL>"
  },
  "standup_cron_id": "",
  "pulse_cron_ids": [],
  "current_sprint": null
}
```

告知用户：配置已保存至 `~/.claude/agile-pm/config.json`，后续所有 skill 将自动从此文件读取配置。

---

### 第三步：配置每日站会定时任务

使用 Claude Code 的 `CronCreate` 工具创建定时任务：

```
CronCreate({
  cron: "0 10 * * *",
  prompt: "agile-project-management:standup",
  durable: true,
  recurring: true
})
```

创建成功后，将返回的 Job ID 写入 `~/.claude/agile-pm/config.json` 的 `standup_cron_id` 字段。

告知用户：每日 10:00 会自动触发站会。若当日已手动完成站会，定时触发会自动跳过。

---

### 第三步 (b)：配置脉搏检查定时任务

使用 CronCreate 创建两个定时任务：

**早间检查：**
```
CronCreate({
  cron: "57 9 * * *",
  prompt: "agile-project-management:pulse",
  durable: true,
  recurring: true
})
```

**午间检查：**
```
CronCreate({
  cron: "7 14 * * *",
  prompt: "agile-project-management:pulse",
  durable: true,
  recurring: true
})
```

将两个返回的 Job ID 写入 `~/.claude/agile-pm/config.json` 的 `pulse_cron_ids` 数组。

告知用户：每日约 10:00 和 14:00 会自动进行项目健康检查（脉搏检查），如有异常会主动提醒。

---

### 完成确认

Setup 完成后，输出配置摘要：

```
✅ 配置完成

- Notion API：已连接
- Task Database：已创建（ID: xxx）
- Dashboard Page：已创建（完整模板已初始化）
- 配置文件：~/.claude/agile-pm/config.json
- 每日站会：每天 10:00 自动触发
- 脉搏检查：每日 ~10:00 和 ~14:00 自动检查

开始使用：
- 添加任务：agile-project-management:backlog
- 规划Sprint：agile-project-management:sprint
- 每日站会：agile-project-management:standup
```
