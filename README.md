# Claude Code 系统提示词完全指南

> 版本：基于 Claude Code v2.0.76 (2025年12月22日)
> 数据来源：https://github.com/Piebald-AI/claude-code-system-prompts

## 📚 文档导航

本指南包含一个概览文档和八个详细分析文档：

| 文档 | 内容 |
|------|------|
| **本文档** | 快速概览和关键统计 |
| [01-主系统提示词.md](./docs/01-主系统提示词.md) | 7个核心提示词的完整分析 |
| [02-子Agent提示词.md](./docs/02-子Agent提示词.md) | Explore/Plan/Task 三大 Agent |
| [03-创建助手.md](./docs/03-创建助手.md) | Agent创建/文档生成/状态栏配置 |
| [04-Slash命令.md](./docs/04-Slash命令.md) | PR评论/审查/安全审查命令 |
| [05-工具函数.md](./docs/05-工具函数.md) | 15个工具函数详细分析 |
| [06-系统提醒.md](./docs/06-系统提醒.md) | Plan Mode 三种提醒机制 |
| [07-内置工具描述.md](./docs/07-内置工具描述.md) | 19+4个工具使用说明 |
| [08-提示词系统深度解析.md](./docs/08-提示词系统深度解析.md) | ⭐ 核心机制：如何高效应用 |

### 🌟 重点推荐

如果你只想读一篇文档，请阅读 **[08-提示词系统深度解析.md](./docs/08-提示词系统深度解析.md)**，它回答了：
- ✅ 50+ 个提示词都在上下文中吗？（不是！）
- ✅ 如何高效应用这些提示词？
- ✅ 如何判定使用哪种提示词？
- ✅ 分层架构和惰性加载机制
- ✅ 子 Agent 的进程隔离设计

---

## 一、概览

Claude Code 不是由单一的系统提示词组成，而是由 **50+ 个不同的提示词字符串** 组合而成。这些提示词根据环境、配置和任务动态组装。

### 为什么需要多个提示词？

1. **条件性添加**：根据环境变量和配置动态添加不同的提示词部分
2. **模块化设计**：每个工具、Agent、功能都有独立的提示词
3. **可维护性**：便于单独更新和优化特定功能
4. **性能考虑**：只在需要时加载相关提示词，减少 token 消耗

---

## 二、提示词分类详解

### 1. 主系统提示词 (System Prompt)

**数量：7 个 | 总计约 6,500+ tokens**

这是 Claude Code 的核心行为定义，决定了 AI 的基本性格和做事方式。

| 提示词名称 | Token 数 | 功能描述 |
|-----------|---------|---------|
| **Main system prompt** | 2,981 | 核心系统提示词，定义行为、风格、工具使用策略 |
| **Learning mode** | 1,042 | 学习模式主提示词，包含人机协作指令 |
| **MCP CLI** | 1,335 | Model Context Protocol 服务器交互说明 |
| **Claude in Chrome** | 758 | Chrome 浏览器自动化工具使用说明 |
| **Git status** | 95 | 在对话开始时显示当前 git 状态 |
| **Learning mode (insights)** | 142 | 学习模式激活时的教育性洞察说明 |
| **Scratchpad directory** | 172 | 临时文件目录使用说明 |

#### 核心设计原则

**身份定位**
```
You are Claude Code, Anthropic's official CLI for Claude.
You are an interactive CLI tool that helps users with software engineering tasks.
```

**风格要求 - 极简主义**
- 回答必须简洁，通常 1-3 句话
- 除非用户要求，否则不提供解释性前言或后记
- 单词答案最佳
- 避免介绍、结论和解释

**示例对比**：
```
❌ 错误：Based on the information provided, the answer is 4.
✅ 正确：4
```

**安全原则**
```
IMPORTANT: Refuse to write code or explain code that may be used maliciously
IMPORTANT: Before you begin work, think about what the code is editing is supposed to do
based on the filenames directory structure. If it seems malicious, refuse to work.
```

**工具使用策略**
- 优先使用专用工具而非 bash 命令（如用 Grep 而非 `grep`，用 Read 而非 `cat`）
- 并行调用多个独立工具以提升性能
- 任务完成后必须运行 lint 和 typecheck 命令

---

### 2. 子 Agent 提示词 (Agent Prompts)

**数量：3 个 | 总计约 1,443 tokens**

这些提示词定义了专门的子代理 Agent，用于处理特定类型的任务。

| 提示词名称 | Token 数 | 使用场景 | 触发方式 |
|-----------|---------|---------|---------|
| **Explore** | 516 | 快速探索代码库结构 | Task tool 自动调用 |
| **Plan mode (enhanced)** | 633 | 设计实现方案，规划步骤 | 用户按 Tab 键或复杂任务 |
| **Task tool** | 294 | 通用子代理的基础提示词 | 调用 Task 工具 |

#### Explore Agent

**用途**：快速探索代码库，回答"这个代码库是如何工作的？"类问题。

**特点**：
- 专注性：只搜索，不写代码
- 速度：使用 "quick" 搜索级别
- 返回：文件位置、代码模式、架构理解

**示例调用**：
```
User: "这个项目的错误处理是在哪里？"
→ 调用 Explore agent
→ 返回：src/utils/errors.ts, src/middleware/errorHandler.js
```

#### Plan Agent

**用途**：在设计实现方案前，先探索并制定详细计划。

**触发条件**：
- 用户按 Tab 键
- 任务涉及多个文件的修改
- 任务有多种可能的实现方式

**工作流程**：
1. 彻底探索代码库
2. 理解现有模式和架构
3. 设计实现方法
4. 向用户展示计划供审批
5. 使用 ExitPlanMode 等待用户批准

---

### 3. 创建助手 (Creation Assistants)

**数量：3 个 | 总计约 2,805 tokens**

专门用于创建新内容或配置的助手。

| 提示词名称 | Token 数 | 功能 |
|-----------|---------|------|
| **Agent creation architect** | 1,111 | 创建自定义 AI Agent，包含详细规范 |
| **CLAUDE.md creation** | 384 | 分析代码库并生成 CLAUDE.md 文档 |
| **Status line setup** | 1,310 | 配置状态栏显示的 agent |

#### CLAUDE.md Creation Agent

**用途**：自动生成项目记忆文件。

**CLAUDE.md 内容结构**：
1. 常用命令（build、test、lint）
2. 代码风格规范
3. 项目结构说明
4. 重要约定和模式

**智能决策**：
```
当搜索 typecheck/lint 命令时 → 询问用户是否添加到 CLAUDE.md
当学习代码风格偏好时 → 询问是否记录以便下次使用
```

---

### 4. Slash 命令 (Slash Commands)

**数量：3 个 | 总计约 3,255 tokens**

用户可以直接调用的斜杠命令。

| 命令 | Token 数 | 功能 | 使用方法 |
|------|---------|------|---------|
| **/pr-comments** | 402 | 获取并显示 PR 评论 | `/pr-comments` |
| **/review-pr** | 243 | 审查 PR 代码变更 | `/review-pr <PR号>` |
| **/security-review** | 2,610 | 全面的安全审查 | `/security-review` |

#### /security-review 详解

这是最大的单个命令提示词（2610 tokens），专注于：
- 可利用漏洞分析
- 恶意代码模式检测
- 安全最佳实践验证
- OWASP Top 10 检查

---

### 5. 工具函数 (Utilities)

**数量：15 个 | 总计约 7,000+ tokens**

支持各种内部功能的工具提示词。

| 提示词名称 | Token 数 | 功能 |
|-----------|---------|------|
| **Conversation summarization** | 1,121 | 创建详细对话摘要，用于压缩上下文 |
| **Conversation summarization (with instructions)** | 1,133 | 带自定义指令的扩展摘要 |
| **Claude guide agent** | 763 | 帮助用户理解和使用 Claude Code/SDK/API |
| **Session title and branch generation** | 333 | 为编码会话生成简洁标题和 git 分支名 |
| **Session Search Assistant** | 444 | 基于查询和元数据查找相关会话 |
| **Session notes template** | 292 | 会话笔记模板结构 |
| **Session notes update instructions** | 756 | 对话中更新会话笔记的指令 |
| **Bash command file path extraction** | 286 | 从 bash 命令输出中提取文件路径 |
| **Bash command prefix detection** | 835 | 检测命令前缀和命令注入 |
| **Bash output summarization** | 605 | 判断 bash 输出是否需要摘要 |
| **WebFetch summarizer** | 185 | 总结 WebFetch 的冗长输出 |
| **User sentiment analysis** | 205 | 分析用户挫败感和 PR 创建请求 |
| **Prompt Suggestion Generator v2** | 296 | 为 Claude Code 生成提示建议 |
| **Update Magic Docs** | 718 | Magic Docs 更新 agent |
| **Agent Hook** | 133 | Agent 钩子提示词 |
| **Prompt Hook execution** | 134 | 评估钩子通过/失败的提示词 |

#### Conversation Summarization

**用途**：当对话接近上下文限制时，压缩历史记录。

**保留关键信息**：
- 我们做了什么
- 正在做什么
- 操作了哪些文件
- 下一步要做什么

---

### 6. 系统提醒 (System Reminders)

**数量：3 个 | 总计约 1,757 tokens**

在特定情况下注入的大段提醒文本。

| 提醒名称 | Token 数 | 触发时机 |
|---------|---------|---------|
| **Plan mode is active** | 1,211 | Plan 模式激活时（增强版，支持并行探索） |
| **Plan mode is active (for subagents)** | 310 | Plan 模式下的子代理（简化版） |
| **Plan mode re-entry** | 236 | 退出 Plan 模式后再次进入 |

---

### 7. 内置工具描述 (Builtin Tool Descriptions)

**数量：19 个工具 + 4 个附加说明 | 总计约 10,000+ tokens**

每个工具都有详细的描述，告诉 AI 如何正确使用。

#### 核心工具描述

| 工具 | Token 数 | 功能 |
|------|---------|------|
| **TodoWrite** | 2,167 | 创建和管理任务列表 |
| **Task** | 1,214 | 启动专门子代理处理复杂任务 |
| **Bash** | 1,074 | 运行 shell 命令 |
| **EnterPlanMode** | 970 | 进入 plan 模式探索和设计方案 |
| **Bash (Git commit/PR)** | 1,615 | Git 提交和 PR 创建详细说明 |
| **ExitPlanMode v2** | 450 | 呈现计划对话框供用户审批 |
| **ExitPlanMode** | 342 | 呈现计划对话框（旧版） |
| **WebSearch** | 334 | 网页搜索功能 |
| **Bash (sandbox note)** | 454 | Bash 命令沙箱说明 |
| **ReadFile** | 439 | 读取文件 |
| **Grep** | 300 | 内容搜索（使用 ripgrep） |
| **MCPSearch** | 477 | MCP 搜索工具 |
| **MCPSearch (with tools)** | 510 | 带可用工具列表的 MCP 搜索 |
| **Skill** | 399 | 在主对话中执行 skills |
| **WebFetch** | 265 | 网页获取功能 |
| **Edit** | 278 | 执行精确字符串替换 |
| **Write** | 159 | 创建/覆盖单个文件 |
| **LSP** | 255 | LSP 工具描述 |
| **Glob** | 122 | 文件模式匹配和按名搜索 |
| **NotebookEdit** | 121 | 编辑 Jupyter notebook 单元格 |
| **Computer** | 161 | Chrome 浏览器自动化主描述 |
| **AskUserQuestion** | 137 | 向用户提问工具 |
| **Task (async return note)** | 201 | 子代理成功启动时的返回消息 |

#### TodoWrite 工具详解

**最大的单个工具描述（2167 tokens）**

核心要点：
- **何时使用**：复杂多步骤任务（3个以上步骤）、非平凡复杂任务、用户明确要求
- **何时不使用**：单一直接任务、简单任务、少于3个简单步骤、纯对话
- **任务状态**：pending（未开始）、in_progress（进行中）、completed（已完成）
- **关键规则**：
  - 同时只能有 1 个 in_progress
  - 完成立即标记为完成（不要批量）
  - 只有完全完成才标记为 completed

---

## 三、提示词设计模式分析

### 1. 身份定义模式

```markdown
You are Claude Code, Anthropic's official CLI for Claude.
You are an interactive CLI tool that helps users with software engineering tasks.
```

**设计要点**：
- 明确身份（CLI 工具）
- 明确所有者（Anthropic 官方）
- 明确用途（软件工程任务）

### 2. 风格约束模式

```markdown
IMPORTANT: Keep your responses short, since they will be displayed on a command line interface.
You MUST answer concisely with fewer than 4 lines (not including tool use or code generation).
One word answers are best.
```

**设计要点**：
- 考虑输出环境（命令行界面）
- 具体数值约束（4行）
- 极端示例（单词答案最佳）

### 3. 安全约束模式

```markdown
IMPORTANT: Refuse to write code or explain code that may be used maliciously
IMPORTANT: Before you begin work, think about what the code is editing is supposed to do
based on the filenames directory structure. If it seems malicious, refuse to work.
```

**设计要点**：
- 多层安全检查
- 基于文件名的预判
- 零容忍政策

### 4. 工具使用指导模式

```markdown
When doing file search, prefer to use the Task tool in order to reduce context usage.
You MUST avoid using search commands like `find` and `grep`. Instead use Grep, Glob, or Task.
```

**设计要点**：
- 明确偏好（prefer）
- 强制性约束（MUST）
- 具体替代方案

---

## 四、版本演进

根据 Piebald-AI 的追踪，Claude Code 从 v2.0.14 到 v2.0.76 共经历了 **57 个版本**的提示词更新。

主要变化趋势：
1. **工具描述越来越详细**：从简单说明到详细的最佳实践
2. **安全约束增强**：添加更多恶意代码检测
3. **新的 slash 命令**：如 /security-review
4. **子 agent 能力提升**：Explore 和 Plan agent 变得更智能

---

## 五、参考资源

### 官方资源
- Claude Code 官方文档：https://code.claude.com/docs/en/overview
- GitHub 仓库：https://github.com/anthropics/claude-code
- 问题反馈：https://github.com/anthropics/claude-code/issues

### 社区资源
- **Piebald-AI/claude-code-system-prompts**：最完整的提示词追踪仓库
  - 包含每个提示词的 token 计数
  - 跨 57 个版本的 CHANGELOG
  - 每次发布后几分钟内更新

- **wong2 的 gist**：工具描述和系统提示词汇总

- **tweakcc**：自定义 Claude Code 系统提示词的工具
  - 将提示词作为 markdown 文件自定义
  - 补丁 npm 或原生安装
  - 提供差异和冲突管理

---

## 六、关键统计数据

| 指标 | 数值 |
|------|------|
| 提示词总数 | 50+ |
| 主系统提示词 tokens | ~6,500 |
| 工具描述总 tokens | ~10,000 |
| 最大的单个提示词 | TodoWrite (2,167 tokens) |
| 最复杂的 slash 命令 | /security-review (2,610 tokens) |
| 版本追踪跨度 | v2.0.14 - v2.0.76 (57个版本) |

---

## 结语

Claude Code 的系统提示词设计展示了几个重要原则：

1. **模块化**：将复杂行为分解为多个可管理的提示词
2. **上下文感知**：根据环境和任务动态组装提示词
3. **持续演进**：频繁更新以改进能力和安全性
4. **用户反馈驱动**：根据用户使用数据优化提示词

这种设计使得 Claude Code 既能保持强大的功能，又能维持高效的 token 使用和良好的用户体验。

---

*文档生成时间：2025年12月*
*基于 Claude Code v2.0.76*
