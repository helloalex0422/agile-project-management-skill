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

逐步收集信息（每步一个问题，等待用户回答后再问下一步）：

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

调用 Notion MCP 的 `query_database` 获取所有任务，提取唯一的 Project 值列表，展示供选择：
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

**若选是：**
- 询问 GitLab 项目路径（如 `group/repo`）
- 调用 Notion MCP 的 `create_page` 先创建 Notion 任务（Status="Backlog"）
- 调用 GitLab MCP 的 `create_issue`，title = 任务名称，description = 用户输入的描述
- 调用 Notion MCP 的 `update_page`，将返回的 issue URL 写入 `GitLab Issue` 字段

**若选否：**
- 直接调用 Notion MCP 的 `create_page` 创建 Notion 任务（Status="Backlog"）

完成后输出：
```
✅ 任务已添加到 Backlog：[任务名]（P1 | 工作项目 | ssb-tool）
```

询问：是否继续添加任务？（是→重复操作1流程，否→返回主菜单）

---

### 操作 2：查看 Backlog

调用 Notion MCP 的 `query_database`，查询 Status = "Backlog" 的任务，按 Priority 排序展示：

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

若用户输入序号，询问新优先级（P0/P1/P2），调用 Notion MCP 的 `update_page` 更新 `Priority` 字段。

可重复调整，直到用户输入"否"。

---

### 操作 3：退出

结束 skill 执行，不做任何操作。
