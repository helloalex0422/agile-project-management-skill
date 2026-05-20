# project-manage

Claude Code Plugin，提供敏捷项目管理能力。作为独立 Scrum Master 使用，支持个人项目和工作项目管理。

## 快速上手

首次使用前，运行 `project-manage:setup` 进行初次配置：

- 配置 Notion MCP 连接
- 配置 GitLab MCP 连接（可选）
- 初始化看板结构
- 设置每日站会定时任务

配置完成后，日常工作流包括：

1. **每日站会** — `project-manage:standup` 生成站会报告，支持手动触发和定时触发
2. **Backlog 管理** — `project-manage:backlog` 新增任务、调整优先级、同步 GitLab issue
3. **Sprint 规划** — `project-manage:sprint` 创建 Sprint、从 Backlog 选任务、关联 GitLab issue
4. **Sprint 回顾** — `project-manage:retro` 统计完成率、复盘总结、归档任务
5. **发布管理** — `project-manage:release` 检查 GitLab milestone、执行发布 checklist、打 tag

## 项目类型

### 个人项目

仅使用 Notion 看板，不涉及 GitLab：

- 看板存储在 Notion 数据库中
- 任务状态流转：Backlog → Sprint Todo → In Progress → In Review → Done
- 适合个人学习项目、开源贡献跟踪

### 工作项目

同时使用 Notion 看板和 GitLab issue 管理：

- Notion 看板作为关键信息汇总中心
- GitLab issue 作为详细工作记录和代码关联
- 任务双向同步：Notion ↔ GitLab
- 适合团队协作、正式项目发布

## 看板结构

标准 Kanban 看板，5 列流转：

| 列名 | 说明 |
|------|------|
| Backlog | 未排期的任务池 |
| Sprint Todo | 当前 Sprint 的待做任务 |
| In Progress | 正在进行中 |
| In Review | 待审核/验收 |
| Done | 已完成 |

每个任务卡片包含以下信息：

- 标题
- 优先级（P0 / P1 / P2）
- 估点（1-13）
- GitLab issue 链接（工作项目）

## 依赖

本 Plugin 依赖以下 MCP 服务：

- **Notion MCP** — 读写 Notion 看板数据库
- **GitLab MCP** — 创建/更新 GitLab issue、查询 milestone、管理 tag（工作项目可选）

在 `project-manage:setup` 中配置 MCP 连接的认证凭证。
