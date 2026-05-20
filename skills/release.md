# project-manage:release

发布管理。检查 GitLab milestone 进度，生成发布 checklist，确认后打 tag。

---

## 执行流程

### Step 1：选择项目和 milestone

询问要发布哪个工作项目：

```
请选择要发布的工作项目：
1. ssb-tool
2. spark-deploy
（从 Notion 任务的 Project 字段中获取 type="工作项目" 的项目列表）
```

询问 GitLab 项目路径（如 `group/repo`，格式为 `命名空间/仓库名`）。

调用 GitLab MCP 的 `list_milestones`，展示可用 milestone：

```
可用 Milestone：
1. v1.0.0（截止：2026-05-31）- 8/10 issues 已关闭
2. v0.9.1（截止：已过期）- 5/5 issues 已关闭
```

用户选择目标 milestone。

### Step 2：检查 milestone 进度

调用 GitLab MCP 的 `list_issues`，查询该项目下属于此 milestone 的所有 issue，分别统计已关闭和未完成：

```
📋 Milestone v1.0.0 进度（8/10 已完成）

✅ 已关闭（8项）：
- #42 [任务A]
- #43 [任务B]
...

🔴 未完成（2项，阻塞发布）：
- #50 [任务C]
- #51 [任务D]
```

若有未完成 issue，询问：
```
有 2 个 issue 未完成，如何处理？
1. 等待完成后再发布（退出，稍后重新运行）
2. 忽略这些 issue，继续发布（将在发布说明中注明）
3. 取消发布
```

选 1 或 3 → 退出 skill。
选 2 → 继续。

### Step 3：生成发布 Checklist

逐项确认（每项用户输入 y 确认或 n 标注未完成）：

```
📋 发布 Checklist - v1.0.0

代码准备：
[ ] 所有功能分支已合并到主分支（y/n）
[ ] CI/CD 流水线全部通过（y/n）
[ ] 代码已 Review（相关 MR 已 merged）（y/n）

测试：
[ ] 功能测试已完成（y/n）
[ ] 回归测试通过（y/n）

文档：
[ ] CHANGELOG 已更新（y/n）
[ ] README 版本号已更新（y/n）

发布：
[ ] 发布说明已准备好（y/n）
```

记录所有未完成项（n 的项目）。若有未完成项，展示汇总后询问：是否仍要继续发布？

### Step 4：确认发布

收集发布信息：

```
🚀 准备发布

Tag 名称：v1.0.0（可修改）
Tag 基于的分支/commit：main（可修改）
发布说明：（请输入，或回车使用 milestone issue 列表自动生成）
```

若用户回车跳过发布说明，自动从已关闭的 milestone issues 列表生成摘要作为发布说明。

展示发布确认摘要：

```
GitLab 项目：group/repo
Tag：v1.0.0 @ main
发布说明：<内容预览前100字>
未完成 Checklist 项：<数量>

确认发布？（y/n）
```

### Step 5：执行发布

用户确认（y）后：

1. 调用 GitLab MCP 的 `create_tag`，参数：project_path、tag_name、ref（分支或 commit SHA）

2. 在 Notion 中，调用 `query_database` 查找与此 GitLab 项目关联（`gitlab_issue` URL 包含该项目路径）且 Status ≠ "Done" 的任务，逐一调用 `update_page` 更新 Status = "Done"

3. 调用 Notion MCP 更新 Dashboard Page，在"最近站会记录"模块追加一条发布记录：`YYYY-MM-DD: 🚀 发布 v1.0.0（group/repo）`

输出：
```
✅ v1.0.0 已发布

- GitLab Tag：已创建（v1.0.0 @ main）
- 关联 Notion 任务：已更新为 Done（N 项）
- Dashboard：已记录发布事件

GitLab Tag 页面：http://gitlab.internal/group/repo/-/tags/v1.0.0
```
