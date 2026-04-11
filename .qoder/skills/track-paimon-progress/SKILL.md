---
name: track-paimon-progress
description: 追踪 Apache Paimon 项目研发进展：拉取最新代码合并到本地，分析每次新增的 commit 变更内容，并将分析结果写入 track 文件夹内的 changelog-日期时间.md 文件。当用户说"追踪 paimon 进展"、"拉取最新变更"、"分析新增 commit"或"生成 changelog"时使用此技能。
---

# Track Apache Paimon Progress

追踪 Apache Paimon 项目的研发进展，自动拉取最新代码、分析新增 commit、生成结构化 changelog 文件。

## 平台适配说明

AI agent 在执行本技能前，应先根据用户的操作系统信息选择对应的命令语法：

- **macOS / Linux**：使用 bash 语法
- **Windows**：使用 PowerShell 语法

对于 `git` 原生命令（如 `git remote -v`、`git pull`、`git log`、`git show`、`git add`、`git commit`、`git push` 等），所有平台语法完全一致，无需区分。

对于**变量赋值、时间获取、目录创建、字符串拼接**等操作，不同平台语法存在差异，相关步骤中会同时提供两种写法，agent 应按实际系统选择其一执行。

---

## 工作流程

### Step 1：确保 upstream 已配置，记录当前 HEAD，拉取最新代码

```bash
# 检查是否存在 upstream remote
git remote -v
```

若没有 `upstream`，先添加 Apache 官方仓库：

```bash
git remote add upstream https://github.com/apache/paimon.git
```

然后记录当前 HEAD，再拉取并合并：

**macOS / Linux (bash)：**
```bash
# 记录拉取前的 HEAD commit
oldHead=$(git rev-parse HEAD)

# 从 Apache 官方仓库拉取最新代码，冲突以官方为准
git pull -X theirs upstream master
```

**Windows (PowerShell)：**
```powershell
# 记录拉取前的 HEAD commit
$oldHead = git rev-parse HEAD

# 从 Apache 官方仓库拉取最新代码，冲突以官方为准
git pull -X theirs upstream master
```

若 `git pull` 仍然失败（如非内容冲突的结构性错误），提示用户具体错误信息，不继续执行。

### Step 2：找出新增的 commit 列表

**macOS / Linux (bash)：**
```bash
# 列出 oldHead..HEAD 之间所有新增的 commit（从旧到新排列）
git log $oldHead..HEAD --oneline --reverse
```

**Windows (PowerShell)：**
```powershell
# 列出 oldHead..HEAD 之间所有新增的 commit（从旧到新排列）
git log $oldHead..HEAD --oneline --reverse
```

若输出为空，说明本地已是最新，提示用户"已是最新版本，无新增 commit"，终止流程。

### Step 3：逐个分析每个 commit

对每个新增 commit，执行（所有平台一致）：

```bash
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

**必须使用命令动态获取当前时间生成文件名**，不得手动填写固定日期：

**macOS / Linux (bash)：**
```bash
projectRoot=$(git rev-parse --show-toplevel)
now=$(date +"%Y-%m-%d-%H%M")
filePath="$projectRoot/track/changelog-$now.md"

# 若 track 文件夹不存在，先创建
mkdir -p "$projectRoot/track"
```

**Windows (PowerShell)：**
```powershell
$projectRoot = git rev-parse --show-toplevel
$now = Get-Date -Format "yyyy-MM-dd-HHmm"
$filePath = "$projectRoot/track/changelog-$now.md"

# 若 track 文件夹不存在，先创建
New-Item -ItemType Directory -Force -Path "$projectRoot/track"
```

- 若 `track` 文件夹不存在，对应命令会自动创建，再写入文件

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

---

## 本次进展亮点

> 从本次变更中挑选 2-10 个最值得关注的功能/修复，重点说明其使用场景和方式。

### 1. {亮点标题}

{2-3句背景说明，解释此功能解决了什么问题或带来了什么改进}

**使用示例**：
根据功能类型自由选择最直观的代码语言，优先顺序如下：
- SQL 操作类→ 优先用 Flink SQL 或 Spark SQL
- 纯 Java/Scala 内部 API → 用 Java 示例
- Python 客户端功能 → 用 Python 示例
- 命令行工具 → 用 Shell 示例

```sql
-- 示例（可替换为 java/python/bash 代码块）
```

### 2. {亮点标题}

{2-3句背景说明}

**使用示例**：
```python
# 示例代码
```

...（以此类推，每个亮点必须包含可运行的示例代码）
```

### Step 5：提交并推送到远程仓库

将新生成的 changelog 文件和上游拉取的代码一起提交并推送：

```bash
# 添加 changelog 文件（所有平台一致）
git add track/changelog-*.md
```

提交时引用前面获取的时间变量：

**macOS / Linux (bash)：**
```bash
git commit -m "track: add changelog $now"
```

**Windows (PowerShell)：**
```powershell
git commit -m "track: add changelog $now"
```

推送到远程（所有平台一致）：

```bash
git push origin master
```

若推送失败（如远程有新提交），提示用户具体错误信息。

## 注意事项

- commit 描述使用**中文**，保持简洁专业
- 若某个 commit 的 diff 过大（超过 500 行），只分析 `--stat` 部分，不逐行阅读全量 diff
- 文件路径引用使用 Markdown 链接格式指向实际文件
- 若新增 commit 超过 20 个，优先分析功能类（Feature/Fix/Perf）commit，测试和文档类可简略处理
- changelog 文件创建后，告知用户文件的完整路径
- **本次进展亮点**章节必须为每个亮点提供可运行的示例代码，代码语言根据功能类型自由选择：SQL 操作类优先 Flink/Spark SQL，纯 Java 内部 API 类使用 Java，Python 客户端功能使用 Python，命令行工具使用 Shell；**示例代码必须严格来源于以下渠道，禁止凭空推断或虚构**：
  1. commit diff 中新增/修改的代码片段（测试代码中的用法也算）
  2. commit message 中作者提供的示例
  3. PR 描述中的用法说明
  若以上渠道均无法提供准确示例，**只写文字描述，不写代码块**，避免给出错误演示
