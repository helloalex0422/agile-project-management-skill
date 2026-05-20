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
3. 在 Claude Code 配置中添加 Notion MCP（在 `~/.claude/settings.json` 的 `mcpServers` 中添加）：

```json
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

在 `~/.claude/settings.json` 的 `mcpServers` 中添加：

```json
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
  cron: "0 10 * * *",
  prompt: "project-manage:standup",
  durable: true,
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
