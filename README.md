# Codex App + Claude Code + Hermes Agent 三端 Git 分支串联流程

本文档用于把三类 agent 串成一个稳定的本地工作流：

- **Codex App**：任务规划、拆分、阶段性 handoff，不直接长期接管代码修改。
- **Claude Code**：在独立 Git 分支里继续执行代码修改。
- **Hermes Agent**：在独立 Git 分支里只做 review，检查 diff、逻辑、统计或生物学合理性、边界条件，不直接改代码。
- **Git**：负责隔离三者修改，避免 main 被半成品污染。
- **`.ai_bus/`**：作为三者之间的任务中转区。

---

## 1. 总体逻辑

三者不是直接互相调用，而是通过 **Git 分支 + `.ai_bus` 文件夹** 串联。

```text
Codex App 规划任务或整理中途进度
        ↓
.ai_bus/inbox/*.md 任务文件
.ai_bus/done/*.md 交接说明
        ↓
Claude Code 在 agent/claude/<task_id> 分支执行代码修改
        ↓
导出 Claude 或 Codex-Claude 合并 diff
        ↓
Hermes Agent 在 agent/hermes-review/<task_id> 分支审查 diff
        ↓
integrate/<task_id> 分支整合
        ↓
验证通过后合并回 main
```

核心原则：

1. **Codex 不直接合并 main。**
2. **Claude 只在自己的分支修改代码。**
3. **Hermes 默认只审查，不改代码。**
4. **main 只接受通过 review 和验证后的结果。**
5. **results、logs、原始数据、大型中间文件默认不进 Git。**

---

## 2. 推荐目录结构

```text
TB_host_pathogen_drug_network_results/
├─ .ai_bus/
│  ├─ inbox/       # 给 Claude 或 Hermes 的任务文件
│  ├─ done/        # Codex 或 Claude 的交接总结
│  ├─ logs/        # Claude 或 Hermes 执行日志，不建议进 Git
│  ├─ patches/     # 给 Hermes 审查的 diff
│  └─ reviews/     # Hermes 审查报告
├─ scripts/
├─ results/        # 运行结果，不建议进 Git
├─ logs/           # 运行日志，不建议进 Git
├─ README.md
├─ requirements.txt
├─ run_all.py
└─ .gitignore
```

推荐 `.gitignore`：

```gitignore
logs/
results/
__pycache__/
*.pyc
*.log
*.tmp
```

如果有原始数据目录，也加入 `.gitignore`：

```gitignore
raw/
data/
原始数据/
```

---

## 3. 分支命名规则

```text
main
├─ agent/codex/<task_id>_wip
├─ agent/claude/<task_id>_continue
├─ agent/hermes-review/<task_id>
├─ agent/hermes-fix/<task_id>      # 可选，只有明确需要修复时使用
└─ integrate/<task_id>
```

示例：

```text
agent/codex/task001_wip
agent/claude/task001_continue
agent/hermes-review/task001
integrate/task001
```

---

## 4. 一次性准备

进入项目目录：

```powershell
cd "E:\多模态数据\TB_host_pathogen_drug_network_results"
```

如果 Git 报 `dubious ownership`：

```powershell
git config --global --add safe.directory "E:/多模态数据/TB_host_pathogen_drug_network_results"
```

初始化 Git：

```powershell
git init
git branch -M main
git commit --allow-empty -m "initial empty baseline before agent workflow"
```

如果项目已经有 Git，只需要：

```powershell
git status
git branch --show-current
```

创建 `.gitignore`：

```powershell
@"
logs/
results/
__pycache__/
*.pyc
*.log
*.tmp
"@ | Set-Content .gitignore -Encoding UTF8
```

创建 `.ai_bus`：

```powershell
mkdir .ai_bus -Force
mkdir .ai_bus\inbox -Force
mkdir .ai_bus\done -Force
mkdir .ai_bus\logs -Force
mkdir .ai_bus\patches -Force
mkdir .ai_bus\reviews -Force
```

提交基础文件：

```powershell
git add .gitignore
git commit -m "chore: add gitignore and agent bus policy"
```

---

## 5. 标准新任务流程

### 5.1 Codex App 新建任务规划分支

```powershell
git switch main
git switch -c agent/codex/task001_wip
```

在 Codex App 中输入：

```text
Use the multi-agent-git-branch skill.

请只做任务规划，不要修改正式代码。请把当前问题拆成任务文件，保存到 .ai_bus/inbox/。

至少创建：
1. task001_claude_continue.md：给 Claude Code 执行，负责实际代码修改。
2. task001_hermes_review.md：给 Hermes Agent 审查，负责 review only。

每个任务文件必须写清楚：
- 背景
- 当前问题
- 允许修改的文件
- 禁止修改的文件
- 完成标准
- 验证命令
- 如果出错，必须说明为什么错、怎么修、下一次怎么检查

分工规则：
1. 需要实际修改代码的任务交给 Claude Code。
2. 需要复核统计、代码逻辑、生物学解释、边界条件的任务交给 Hermes Agent。
3. 不允许两个 agent 同时修改同一个文件。
4. Hermes 默认 review only，不直接修改代码。
5. 不要修改 results、logs、原始数据或大型中间文件。
```

Codex 创建任务文件后，提交：

```powershell
git add .ai_bus\inbox
git commit -m "codex: add task001 task specs"
```

---

### 5.2 Claude Code 接手执行

基于 Codex WIP 分支创建 Claude 分支：

```powershell
git switch agent/codex/task001_wip
git switch -c agent/claude/task001_continue
```

#### 方式 A：交互模式，推荐

```powershell
claude
```

进入 Claude Code 后输入：

```text
Use the multi-agent-git-branch skill.

你现在处于 Git 分支 agent/claude/task001_continue。请读取：

.ai_bus/inbox/task001_claude_continue.md

并接着当前 Codex WIP 状态继续完成任务。

规则：
1. 只修改任务文件允许修改的文件。
2. 不要修改原始数据。
3. 不要修改 results/ 和 logs/。
4. 不要删除 Codex 已经完成的内容。
5. 不要合并 main。
6. 完成后请说明：
   - 修改了哪些文件
   - 为什么原来会错
   - 这次怎么修
   - 下一次遇到类似问题怎么检查
   - 我下一步应该运行哪些验证命令

开始前请先读取任务文件，并告诉我你理解的任务边界。
```

#### 方式 B：非交互模式，适合明确小任务

```powershell
mkdir .ai_bus\logs -Force

claude -p "请读取 .ai_bus\inbox\task001_claude_continue.md，并接着当前 Codex WIP 状态继续完成任务。只修改任务允许修改的文件，不要修改原始数据，不要修改 results 和 logs。完成后说明：改了什么、为什么原来会错、下一次怎么检查。" --allowedTools "Read,Edit,Bash" --output-format text *> ".ai_bus\logs\task001_claude_continue.log"
```

查看日志：

```powershell
Get-Content ".ai_bus\logs\task001_claude_continue.log" -Tail 80 -Wait
```

Claude 完成后检查：

```powershell
git status
git diff --stat
git diff
```

确认无误后提交：

```powershell
git add -A
git commit -m "claude: continue task001 from Codex WIP"
```

---

### 5.3 导出 diff 给 Hermes 审查

回到 main：

```powershell
git switch main
mkdir .ai_bus\patches -Force
```

导出完整 diff：

```powershell
git diff main...agent/claude/task001_continue > .ai_bus\patches\task001_codex_claude.diff
```

提交 diff：

```powershell
git add .ai_bus\patches\task001_codex_claude.diff
git commit -m "chore: add Codex-Claude diff for Hermes review task001"
```

---

### 5.4 Hermes Agent 审查

创建 Hermes review 分支：

```powershell
git switch -c agent/hermes-review/task001
mkdir .ai_bus\reviews -Force
```

运行 Hermes：

```powershell
wsl bash -lc "cd '/mnt/e/多模态数据/TB_host_pathogen_drug_network_results' && hermes -z '请审查 .ai_bus/patches/task001_codex_claude.diff、.ai_bus/done/task001_codex_handoff.md 和 .ai_bus/inbox/task001_hermes_review.md。你只能输出审查报告，不要修改代码。请判断：1 是否符合任务；2 Codex 半成品是否引入风险；3 Claude 接续修改是否正确；4 是否建议合并；5 如不建议合并，列出必须修复的问题。' > .ai_bus/reviews/task001_hermes_review.md"
```

提交 Hermes 审查：

```powershell
git add .ai_bus\reviews\task001_hermes_review.md
git commit -m "hermes: review task001 Codex-Claude changes"
```

阅读审查报告：

```powershell
Get-Content .ai_bus\reviews\task001_hermes_review.md -Raw
```

---

### 5.5 整合分支

如果 Hermes 认为可以合并：

```powershell
git switch main
git switch -c integrate/task001
git merge --no-ff agent/claude/task001_continue -m "merge Claude implementation for task001"
git merge --no-ff agent/hermes-review/task001 -m "merge Hermes review for task001"
```

验证：

```powershell
python run_all.py
```

或按任务文件里的验证命令运行，例如：

```powershell
python scripts\01e_summarize_patient_level_dst.py
python scripts\09_validate_regional_resistance.py
python scripts\10_visualization.py
```

验证通过后合并回 main：

```powershell
git switch main
git merge --no-ff integrate/task001 -m "final merge task001 after Codex-Claude-Hermes review"
```

---

## 6. Codex 已经做了一半时的流程

如果 Codex 已经在项目中改了一半，先不要让 Claude 或 Hermes 接手。先固定 Codex 当前状态。

### 6.1 保存 Codex WIP

```powershell
cd "E:\多模态数据\TB_host_pathogen_drug_network_results"
git status
git branch --show-current
```

如果还没有 Git：

```powershell
git init
git config --global --add safe.directory "E:/多模态数据/TB_host_pathogen_drug_network_results"
git branch -M main
git commit --allow-empty -m "initial empty baseline before Codex WIP"
git switch -c agent/codex/task001_wip
```

如果已经在 `agent/codex/task001_wip`：

```powershell
git status
```

不要提交 `results/`、`logs/`、`__pycache__/`。推荐：

```powershell
git reset

@"
logs/
results/
__pycache__/
*.pyc
*.log
*.tmp
"@ | Set-Content .gitignore -Encoding UTF8

git add README.md requirements.txt run_all.py scripts .gitignore
git commit -m "codex: save current WIP code state for task001"
```

### 6.2 让 Codex 只创建交接文件

在 Codex App 中输入：

```text
Use the multi-agent-git-branch skill.

当前任务已经执行到一半。请停止继续修改正式代码，不要再运行脚本，不要再扩展新功能，不要清理或重构无关文件。

请只创建交接文件，不要修改 scripts、results、logs、README、requirements 或 run_all.py。

请实际创建以下文件：

1. .ai_bus/done/task001_codex_handoff.md
内容包括：
- 当前任务目标
- 已完成的修改
- 已修改文件列表
- 尚未完成的部分
- 可能存在风险的地方
- 建议交给 Claude Code 继续处理的内容
- 建议交给 Hermes 审查的内容
- 当前不建议继续由 Codex 修改的原因

2. .ai_bus/inbox/task001_claude_continue.md
交给 Claude Code 继续完成。必须写清楚：
- 背景
- 当前 Codex 已经做了什么
- Claude 允许修改哪些文件
- Claude 禁止修改哪些文件
- 完成标准
- 必须输出为什么错、怎么修、下一次怎么检查

3. .ai_bus/inbox/task001_hermes_review.md
交给 Hermes review only。Hermes 只审查 diff、代码逻辑、统计/生物学逻辑和边界条件，不直接修改代码。

要求：
1. 不要删除当前已完成的修改。
2. 不要提交 main。
3. 不要继续运行后续脚本。
4. 不要修改正式代码文件。
5. 最后只告诉我这三个文件是否已经创建成功。
```

交接文件创建后提交：

```powershell
git add .ai_bus
git commit -m "codex: add handoff files for task001"
```

确认：

```powershell
git status
dir .ai_bus\done
dir .ai_bus\inbox
```

然后再进入 Claude 流程。

---

## 7. 当前 TB 项目的建议执行顺序

你的当前项目路径：

```powershell
cd "E:\多模态数据\TB_host_pathogen_drug_network_results"
```

当前已完成状态示例：

```text
agent/codex/task001_wip
800c0a7 codex: save current WIP code state for task001
2de1393 codex: add handoff files for task001
```

下一步：

```powershell
git switch -c agent/claude/task001_continue
mkdir .ai_bus\logs -Force
claude
```

Claude Code 里输入：

```text
Use the multi-agent-git-branch skill.

请读取 .ai_bus/inbox/task001_claude_continue.md。
你现在在 agent/claude/task001_continue 分支。
请接着当前 Codex WIP 完成任务。

只修改任务文件允许修改的文件。
不要修改原始数据、results/、logs/。
不要合并 main。
完成后说明修改了什么、为什么错、怎么修、下次怎么检查，以及我应该运行哪些验证命令。
```

Claude 完成后：

```powershell
git status
git diff --stat
git diff
git add -A
git commit -m "claude: continue task001 from Codex WIP"
```

然后 Hermes：

```powershell
git switch main
mkdir .ai_bus\patches -Force
git diff main...agent/claude/task001_continue > .ai_bus\patches\task001_codex_claude.diff
git add .ai_bus\patches\task001_codex_claude.diff
git commit -m "chore: add Codex-Claude diff for Hermes review task001"

git switch -c agent/hermes-review/task001
mkdir .ai_bus\reviews -Force

wsl bash -lc "cd '/mnt/e/多模态数据/TB_host_pathogen_drug_network_results' && hermes -z '请审查 .ai_bus/patches/task001_codex_claude.diff、.ai_bus/done/task001_codex_handoff.md 和 .ai_bus/inbox/task001_hermes_review.md。你只能输出审查报告，不要修改代码。请判断是否建议合并，并列出必须修复的问题。' > .ai_bus/reviews/task001_hermes_review.md"

git add .ai_bus\reviews\task001_hermes_review.md
git commit -m "hermes: review task001 Codex-Claude changes"
```

---

## 8. 常见问题和处理

### 8.1 `fatal: detected dubious ownership`

原因：Git 认为该目录所在文件系统不能记录或确认所有权。

处理：

```powershell
git config --global --add safe.directory "E:/多模态数据/TB_host_pathogen_drug_network_results"
```

下次遇到移动硬盘、E 盘、WSL 挂载路径、网络盘时，先加 safe.directory。

---

### 8.2 `LF will be replaced by CRLF`

这是换行符警告，不是错误。意思是 Git 可能把 LF 换成 Windows 的 CRLF。

如果想减少该警告，可以在仓库中建立 `.gitattributes`：

```gitattributes
* text=auto
*.py text eol=lf
*.md text eol=lf
*.txt text eol=lf
*.csv text eol=lf
```

然后提交：

```powershell
git add .gitattributes
git commit -m "chore: normalize line endings"
```

---

### 8.3 `git add -A` 把 results 也加进去了

先取消暂存：

```powershell
git reset
```

再写 `.gitignore`：

```powershell
@"
logs/
results/
__pycache__/
*.pyc
*.log
*.tmp
"@ | Set-Content .gitignore -Encoding UTF8
```

然后只加代码：

```powershell
git add README.md requirements.txt run_all.py scripts .gitignore
```

---

### 8.4 Claude Code 没有实时输出

如果用了：

```powershell
*> ".ai_bus\logs\task001_claude_continue.log"
```

输出会被写进日志，所以终端看起来像卡住。

另开一个窗口查看：

```powershell
Get-Content ".ai_bus\logs\task001_claude_continue.log" -Tail 80 -Wait
```

如果想直接对话，使用：

```powershell
claude
```

不要用 `claude -p`。

---

### 8.5 WSL Hermes 提示 localhost 代理没有镜像

如果 Hermes 在 WSL 里能正常访问 API，可以忽略。

如果 Hermes 不能联网，需要把 Windows 代理改成 WSL 可访问的主机 IP，而不是 `localhost`。先查 Windows 网关：

```bash
ip route | grep default
```

然后把代理配置为类似：

```bash
export HTTPS_PROXY=http://<windows_gateway_ip>:<port>
export HTTP_PROXY=http://<windows_gateway_ip>:<port>
```

---

## 9. 三端职责边界

| Agent | 主要职责 | 是否改代码 | 是否合并 main |
|---|---|---:|---:|
| Codex App | 拆任务、整理 handoff、最终审阅 | 少量，最好只写 `.ai_bus` | 否 |
| Claude Code | 按任务文件执行代码修改 | 是 | 否 |
| Hermes Agent | 审查 diff、统计逻辑、边界条件、生物学合理性 | 默认否 | 否 |
| 人 | 检查 diff、运行验证、最终合并 | 是 | 是 |

---

## 10. 每次任务的最短操作清单

```powershell
# 1. Codex WIP 分支
git switch main
git switch -c agent/codex/taskXXX_wip

# 2. Codex 创建 .ai_bus/inbox 任务文件后
git add .ai_bus
git commit -m "codex: add taskXXX specs"

# 3. Claude 执行
git switch -c agent/claude/taskXXX_continue
claude

# 4. Claude 完成后
git status
git diff --stat
git add -A
git commit -m "claude: complete taskXXX"

# 5. 导出 diff
git switch main
mkdir .ai_bus\patches -Force
git diff main...agent/claude/taskXXX_continue > .ai_bus\patches\taskXXX_codex_claude.diff
git add .ai_bus\patches\taskXXX_codex_claude.diff
git commit -m "chore: add diff for Hermes review taskXXX"

# 6. Hermes review
git switch -c agent/hermes-review/taskXXX
mkdir .ai_bus\reviews -Force
wsl bash -lc "cd '/mnt/e/多模态数据/TB_host_pathogen_drug_network_results' && hermes -z '请审查 .ai_bus/patches/taskXXX_codex_claude.diff 和 .ai_bus/inbox/taskXXX_hermes_review.md。只输出审查报告，不要修改代码。' > .ai_bus/reviews/taskXXX_hermes_review.md"
git add .ai_bus\reviews\taskXXX_hermes_review.md
git commit -m "hermes: review taskXXX"

# 7. 整合
git switch main
git switch -c integrate/taskXXX
git merge --no-ff agent/claude/taskXXX_continue -m "merge Claude implementation taskXXX"
git merge --no-ff agent/hermes-review/taskXXX -m "merge Hermes review taskXXX"

# 8. 验证后合并 main
python run_all.py
git switch main
git merge --no-ff integrate/taskXXX -m "final merge taskXXX"
```

---

## 11. 不建议做的事

不要：

```text
1. 让 Codex、Claude、Hermes 同时在同一个分支改代码。
2. 在 main 上直接让 agent 自动改。
3. 把 results/、logs/、原始数据、大型 csv/xlsx 全部提交进 Git。
4. 没看 git diff 就 commit。
5. Hermes review 还没看就 merge main。
6. Claude 非交互跑很久但不看日志。
7. Codex 处理一半时直接让 Claude 在 main 上接着改。
```

应该：

```text
1. 每个 agent 一个分支。
2. 每个阶段一个 commit。
3. 每次交接都写 .ai_bus 文件。
4. 每次提交前看 git status 和 git diff --stat。
5. 最后在 integrate 分支验证。
6. 只有验证通过才合并 main。
```

---

## 12. 官方依据和本地约定

本流程基于以下事实和本地约定：

1. Codex App 可以作为任务规划者，但没有 Codex CLI 时，不适合被 PowerShell 直接调度。
2. Claude Code 支持交互模式，也支持 `claude -p` 这种一次性非交互执行方式。
3. Hermes Agent 在 WSL 里可以用 `hermes -z` 输出单次结果，适合做 review-only。
4. Agent Skills 的核心入口是 `SKILL.md`，你已安装 `multi-agent-git-branch` skill 到三端：
   - Codex App: `C:\Users\cbl02\.agents\skills\multi-agent-git-branch`
   - Claude Code: `C:\Users\cbl02\.claude\skills\multi-agent-git-branch`
   - Hermes Agent: `/home/cbl02/.hermes/skills/multi-agent-git-branch`

本项目推荐继续使用 `.ai_bus + Git branches`，不要试图让三个 agent 直接互相控制。
