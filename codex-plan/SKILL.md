---
name: codex-plan
description: 通过 Codex CLI 生成/更新 planning-with-files-zh 规划文件。当用户说"用 codex 规划"、"codex plan"、"更新规划"、"刷新计划"等时触发。
---

# Codex Plan - 委托 Codex 管理规划文件

Codex 只负责生成/更新三个规划文件，不执行任何实际任务。所有代码编写和修改由 Claude Code 完成。

## 首次调用：环境检查

**每次调用此技能时，先检查初始化标记文件是否存在：**

```
~/.claude/skills/codex-plan/.initialized
```

### 如果 `.initialized` 不存在 → 执行完整环境检查

1. **检查 Codex CLI 是否安装：**

```bash
which codex && codex --version
```

如果未安装（命令不存在或报错），停止执行并提示用户：

> Codex CLI 未安装。请先安装 Codex CLI：
>
> ```bash
> npm install -g @openai/codex
> ```
>
> 安装完成后请重新运行此任务。

2. **检查 planning-with-files-zh 是否安装：**

```bash
test -f ~/.agents/skills/planning-with-files-zh/SKILL.md && echo "已安装" || echo "未安装"
```

如果未安装（文件不存在），提示用户：

> planning-with-files-zh 技能未在 Codex 中安装。正在自动安装...
>
> 然后执行安装：
> ```bash
> mkdir -p ~/.agents/skills/planning-with-files-zh/templates &&
> mkdir -p ~/.agents/skills/planning-with-files-zh/scripts &&
> cp ~/.claude/skills/planning-with-files-zh/SKILL.md ~/.agents/skills/planning-with-files-zh/SKILL.md &&
> cp ~/.claude/skills/planning-with-files-zh/templates/* ~/.agents/skills/planning-with-files-zh/templates/ &&
> cp ~/.claude/skills/planning-with-files-zh/scripts/* ~/.agents/skills/planning-with-files-zh/scripts/
> ```

3. **两项检查都通过后，创建初始化标记：**

```bash
mkdir -p ~/.claude/skills/codex-plan && echo "$(date)" > ~/.claude/skills/codex-plan/.initialized
```

### 如果 `.initialized` 已存在 → 跳过检查，直接执行

---

## 两种模式

根据用户意图选择对应模式：

| 用户说 | 模式 | 行为 |
|--------|------|------|
| "用 codex 规划"、"codex plan"、新任务 | **新建** | 从零生成三个规划文件 |
| "更新规划"、"刷新计划"、"sync 规划" | **更新** | 基于当前进度刷新规划文件 |

---

## 模式 1：新建规划

从零开始，Codex 分析任务并生成三个规划文件。

**执行要求：** Bash 调用时设置 `timeout: 900000`（15分钟），将 codex exec 的 stdout 实时展示给用户，让用户看到文件生成进度。

```bash
codex exec --skip-git-repo-check --color never '$planning-with-files-zh 帮我分析以下任务，只创建 task_plan.md、findings.md、progress.md 三个规划文件。不要执行任何代码，不要修改任何项目文件，只生成这三个规划文件。任务描述：<用户的完整需求>'
```

### codex exec 完成后

**无论成功或失败，立即结束本次技能调用，不继续执行任何后续操作。**

1. 检查 codex exec 的退出码和输出
2. 如果失败（退出码非0），向用户报告错误信息，结束
3. 如果成功，验证三个文件是否已生成：
   ```bash
   ls -la task_plan.md findings.md progress.md
   ```
4. 打印文件摘要：
   ```bash
   echo "=== task_plan.md 阶段概览 ===" && grep -E "阶段|状态" task_plan.md && echo "" && echo "=== findings.md 关键发现 ===" && head -20 findings.md && echo "" && echo "=== progress.md ===" && head -10 progress.md
   ```
5. 向用户报告生成结果，结束。**不要自动进入执行阶段。**

## 模式 2：更新规划

当前项目已有规划文件，但 Claude Code 执行了一段时间后进度发生变化。让 Codex 读取当前规划文件，刷新阶段状态、更新 findings、同步 progress。

### 更新步骤

1. 先读取当前三个规划文件的内容：

```bash
cat task_plan.md && echo "===FINDINGS===" && cat findings.md && echo "===PROGRESS===" && cat progress.md
```

2. 将当前规划内容 + 用户最新需求一起发给 Codex（**Bash 调用设置 `timeout: 900000`**，实时展示 stdout）：

```bash
codex exec --skip-git-repo-check --color never '$planning-with-files-zh 只更新 task_plan.md、findings.md、progress.md 三个规划文件，不要执行任何代码。以下是当前的规划文件内容和最新进展，请根据实际进度刷新这三个文件。用户最新反馈：<用户的新需求或进展>。当前规划文件内容如下：<第一步 cat 命令的输出>'
```

3. 等待 Codex 覆盖写入三个文件
4. **codex exec 完成后，立即结束本次技能调用：**
   - 成功 → 验证文件已更新，打印阶段摘要，报告用户
   - 失败 → 报告错误信息给用户
   - **不要自动进入执行阶段**

---

## 重要

- 首次调用时务必执行环境检查，不要跳过
- **Codex 只生成/更新规划文件，不执行任务** — 所有实际工作在 Claude Code 中完成
- 新建模式下，把用户的任务完整、准确地传递给 Codex
- 更新模式下，务必将当前规划文件内容和最新进展一起传给 Codex
- **调用 codex exec 时，Bash 工具必须设置 `timeout: 900000`（15分钟）**
- **将 codex exec 的 stdout 实时展示给用户**，让用户看到文件生成进度
- **codex exec 完成后（成功或失败），立即结束技能调用，打印输出结果，不要继续执行任何后续操作**
