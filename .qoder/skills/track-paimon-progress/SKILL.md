---
name: track-paimon-progress
description: 追踪 Apache Paimon 项目研发进展：拉取最新代码合并到本地，分析每次新增的 commit 变更内容，并将分析结果写入 track 文件夹内的 changelog-日期时间.md 文件。当用户说"追踪 paimon 进展"、"拉取最新变更"、"分析新增 commit"或"生成 changelog"时使用此技能。
---

# Track Apache Paimon Progress

追踪 Apache Paimon 项目的研发进展，自动拉取最新代码、分析新增 commit、生成结构化 changelog 文件。

## 工作流程

### Step 1：确保 upstream 已配置，记录当前 HEAD，拉取最新代码

```powershell
# 检查是否存在 upstream remote
git remote -v
```

若没有 `upstream`，先添加 Apache 官方仓库：

```powershell
git remote add upstream https://github.com/apache/paimon.git
```

然后拉取并合并：

```powershell
# 记录拉取前的 HEAD commit
$oldHead = git rev-parse HEAD

# 从 Apache 官方仓库拉取最新代码，冲突以官方为准
git pull -X theirs upstream master
```

若 `git pull` 仍然失败（如非内容冲突的结构性错误），提示用户具体错误信息，不继续执行。

### Step 2：找出新增的 commit 列表

```powershell
# 列出 oldHead..HEAD 之间所有新增的 commit（从旧到新排列）
git log $oldHead..HEAD --oneline --reverse
```

若输出为空，说明本地已是最新，提示用户"已是最新版本，无新增 commit"，终止流程。

### Step 3：逐个分析每个 commit

对每个新增 commit，执行：

```powershell
git show <commit_hash> --stat
git show <commit_hash>
```

分析时关注以下维度：
- **类型判断**：新功能（Feature）/ Bug 修复（Fix）/ 性能优化（Perf）/ 重构（Refactor）/ 文档注释（Docs）/ 测试（Test）/ 其他
- **影响模块**：从提交信息的 `[模块]` 前缀和改动文件路径推断（如 `[core]`、`[flink]`、`[spark]`、`[python]` 等）
- **核心变更**：用 1-3 句话描述主要改动内容
- **关键文件**：列出最核心的改动文件（最多 5 个）

### Step 4：生成 changelog 文件

文件命名格式：`changelog-{YYYY-MM-DD-HHmm}.md`
文件存放路径：项目根目录下的 `track/` 文件夹

**必须使用 PowerShell 命令动态获取当前时间生成文件名**，不得手动填写固定日期：

```powershell
$projectRoot = git rev-parse --show-toplevel
$now = Get-Date -Format "yyyy-MM-dd-HHmm"
$filePath = "$projectRoot/track/changelog-$now.md"
```

- 若 `track` 文件夹不存在，先创建该文件夹，再写入文件

## changelog 文件模板

```markdown
# Apache Paimon 研发进展 - {YYYY-MM-DD HH:mm}

> 拉取时间：{YYYY-MM-DD HH:mm}
> 新增 commit 数量：{N}
> commit 范围：{oldHead_short}..{newHead_short}

---

## 变更列表

### 1. {commit_type} | {module} | {commit_subject}

- **commit**：`{hash_short}`（作者：{author}，时间：{date}）
- **类型**：{Feature / Fix / Perf / Refactor / Docs / Test}
- **摘要**：{1-3句核心描述}
- **关键文件**：
  - `{file_path_1}`：{一句话说明}
  - `{file_path_2}`：{一句话说明}

---

### 2. {commit_type} | {module} | {commit_subject}

...（以此类推）

---

## 模块统计

| 模块 | 新增 commit 数 | 类型分布 |
|------|--------------|---------|
| core | N | Feature×a, Fix×b |
| flink | N | Feature×a |
| ...  | N | ... |
```

### Step 5：提交并推送到远程仓库

将新生成的 changelog 文件和上游拉取的代码一起提交并推送：

```powershell
# 添加 changelog 文件
git add track/changelog-*.md

# 提交
git commit -m "track: add changelog $now"

# 推送到 origin
git push origin master
```

若推送失败（如远程有新提交），提示用户具体错误信息。

## 注意事项

- commit 描述使用**中文**，保持简洁专业
- 若某个 commit 的 diff 过大（超过 500 行），只分析 `--stat` 部分，不逐行阅读全量 diff
- 文件路径引用使用 Markdown 链接格式指向实际文件
- 若新增 commit 超过 20 个，优先分析功能类（Feature/Fix/Perf）commit，测试和文档类可简略处理
- changelog 文件创建后，告知用户文件的完整路径
