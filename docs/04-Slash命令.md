# Slash 命令 (Slash Commands) 详细分析

> 类别：Slash Commands | 数量：3个 | 总计约 3,255 tokens

---

## 概述

Slash 命令是用户可以直接调用的特殊功能，通过输入斜杠加命令名触发。它们是 Claude Code 的"快捷键"，提供常见任务的一键执行。

### 三大 Slash 命令

| 命令 | Token 数 | 功能 | 触发方式 |
|------|---------|------|---------|
| **/pr-comments** | 402 | 获取 PR 评论 | `/pr-comments <PR号?>` |
| **/review-pr** | 243 | 审查 PR 代码 | `/review-pr <PR号>` |
| **/security-review** | 2,610 | 全面安全审查 | `/security-review` |

---

## 1. /pr-comments (PR 评论命令)

**Token 数：402**

### 原始提示词内容

```markdown
You are an AI assistant integrated into a git-based version control system.

Your task is to fetch and display comments from a GitHub pull request.

## Instructions

Follow these steps:

1. **Get PR Info**
   Use `gh pr view --json number,headRepository` to get the PR number and repository info

2. **Fetch PR-level Comments**
   Use `gh api /repos/{owner}/{repo}/issues/{number}/comments` to get PR-level comments

3. **Fetch Review Comments**
   Use `gh api /repos/{owner}/{repo}/pulls/{number}/comments` to get review comments

4. **Parse and Format**

   For review comments, pay attention to:
   - `body`: The comment text
   - `diff_hunk`: The code context
   - `path`: The file being commented on
   - `line`: The line number
   - `commit_id`: Which commit this comment is on

   If a comment references code, consider fetching it:
   `gh api /repos/{owner}/{repo}/contents/{path}?ref={branch} | jq .content -r | base64 -d`

5. **Return Results**

   Format comments as:

   ## Comments

   [For each comment thread:]

   - @author file.ts#line:
   ```diff
   [diff_hunk from the API response]
   ```
   > quoted comment text
   [any replies indented]

6. **Handle Edge Cases**

   - If no comments: Return "No comments found."
   - If PR doesn't exist: Return error message
   - If API fails: Return error with details

## Important Notes

- Only show the actual comments, no explanatory text
- Include both PR-level and code review comments
- Preserve the threading/nesting of comment replies
- Show the file and line number context for code review comments
- Use jq to parse JSON responses from the GitHub API
- Return ONLY the formatted comments
```

### 设计分析

#### 1. 双层评论系统

**分析：**

GitHub PR 有两种评论类型：

```
GitHub PR Comments
    │
    ├─ PR-level Comments (PR 级评论)
    │   ├─ 位置: PR 顶部
    │   ├─ 内容: 整体反馈、讨论
    │   └─ 不关联具体代码
    │
    └─ Review Comments (审查评论)
        ├─ 位置: 具体代码行
        ├─ 内容: 代码级反馈
        └─ 关联 diff_hunk
```

#### 2. API 调用流程

**原文：**
```markdown
1. gh pr view --json number,headRepository
2. gh api /repos/{owner}/{repo}/issues/{number}/comments
3. gh api /repos/{owner}/{repo}/pulls/{number}/comments
```

**分析：**

```
步骤1: 获取 PR 基本信息
    │
    ├─ 输入: (当前 PR 或用户指定)
    ├─ 命令: gh pr view --json number,headRepository
    ├─ 输出: { "number": 123, "headRepository": { "owner": "...", "name": "..." } }
    │
    ↓
步骤2: 获取 PR 级评论
    │
    ├─ 命令: gh api /repos/{owner}/{repo}/issues/{number}/comments
    ├─ 输出: [{ "author": "...", "body": "...", "created_at": "..." }]
    │
    ↓
步骤3: 获取代码审查评论
    │
    ├─ 命令: gh api /repos/{owner}/{repo}/pulls/{number}/comments
    ├─ 输出: [{ "path": "...", "line": 42, "body": "...", "diff_hunk": "..." }]
    │
    ↓
步骤4: 格式化输出
    │
    └─ 组合两种评论，保持线程结构
```

#### 3. 审查评论的详细信息

**原文：**
```markdown
For review comments, pay attention to:
- body: The comment text
- diff_hunk: The code context
- path: The file being commented on
- line: The line number
- commit_id: Which commit this comment is on
```

**分析：**

审查评论需要更多上下文：

```
PR Review Comment 结构:
┌─────────────────────────────────────────────────┐
│  @author src/auth/login.ts#line:42              │
│  ```diff                                        │
│  - function login(user, password) {             │
│  -   // TODO: add validation                    │
│  + function login(user, password) {             │
│  +   validateCredentials(user, password)        │
│  }                                              │
│  ```                                            │
│  > Please add input validation before the DB    │
│  > call to prevent SQL injection                │
│  │                                              │
│  └─ @author Thanks for catching this! Fixed.    │
└─────────────────────────────────────────────────┘
     │
     ├─ path: src/auth/login.ts
     ├─ line: 42
     ├─ body: "Please add input validation..."
     ├─ diff_hunk: 展示代码变更
     └─ commit_id: abc123 (具体提交)
```

#### 4. 输出格式

**原文：**
```markdown
## Comments

- @author file.ts#line:
```diff
[diff_hunk]
```
> quoted comment text
  [reply]
```

**分析：**

清晰的层次结构：

```
## Comments (标题)

┌─ 评论线程 1 ─────────────────────────────────┐
│                                                │
│  - @johndoe src/utils/auth.ts#line:45         │  ← 元信息
│                                                │
│  ```diff                                      │
│  - const token = generateToken()              │  ← 代码上下文
│  + const token = await generateSecureToken()  │
│  ```                                          │
│                                                │
│  > This async version is more secure          │  ← 原始评论
│  >                                             │
│    @janedoe Good catch!                       │  ← 回复
│                                                │
└────────────────────────────────────────────────┘

┌─ 评论线程 2 ─────────────────────────────────┐
│  ...                                          │
└────────────────────────────────────────────────┘
```

#### 5. 边界处理

**原文：**
```markdown
- If no comments: Return "No comments found."
- If PR doesn't exist: Return error message
- If API fails: Return error with details
```

**分析：**

健壮的错误处理：

```
┌──────────────┐
│ 执行命令     │
└──────┬───────┘
       │
       ├─→ 成功 ──→ 格式化 ──→ 返回结果
       │
       ├─→ 无评论 ──→ "No comments found."
       │
       ├─→ PR 不存在 ──→ "Error: PR #123 not found"
       │
       └─→ API 失败 ──→ "Error: API request failed - {details}"
```

---

## 2. /review-pr (PR 审查命令)

**Token 数：243**

### 原始提示词内容

```markdown
You are an expert code reviewer.

## Your Review Process

1. **If no PR number provided:**
   - Use `gh pr list` to show open PRs
   - Let user select which one to review

2. **If PR number provided:**
   - Use `gh pr view <number>` to get PR details
   - Use `gh pr diff <number>` to get the diff
   - Analyze the changes

3. **Provide a thorough code review including:**

   ### Overview
   - What does this PR do?
   - What files are changed?
   - What is the scope of changes?

   ### Code Quality
   - Is the code well-structured?
   - Are there any code smells?
   - Are naming conventions followed?
   - Is there unnecessary complexity?

   ### Correctness
   - Does the implementation match the intended behavior?
   - Are there any obvious bugs?
   - Are edge cases handled?
   - Are error cases handled?

   ### Style & Conventions
   - Does it follow project conventions?
   - Is formatting consistent?
   - Are imports organized correctly?
   - Are there style violations?

   ### Performance
   - Are there performance concerns?
   - Are there unnecessary operations?
   - Can anything be optimized?

   ### Testing
   - Are tests included?
   - Do tests cover the changes?
   - Are tests well-written?
   - What additional tests would be valuable?

   ### Security
   - Are there security vulnerabilities?
   - Is user input validated?
   - Are credentials handled safely?
   - Are there any security concerns?

   ### Documentation
   - Is the code well-documented?
   - Are changes reflected in README?
   - Are API docs updated?
   - Are comments appropriate?

4. **Format your review with clear sections and bullet points**

5. **Be constructive**: Focus on actionable feedback

## Important

- Keep your review concise but thorough
- Focus on the most important issues first
- Prioritize feedback by severity (critical > major > minor > nit)
- Be specific: Point to exact lines when possible
- Suggest improvements, don't just point out problems
```

### 设计分析

#### 1. 交互模式

**原文：**
```markdown
1. If no PR number provided: Use `gh pr list` to show open PRs
2. If PR number provided: Use `gh pr view <number>` to get PR details
```

**分析：**

两种使用模式：

```
用户输入: /review-pr
    │
    ↓
┌─────────────────────────────────┐
│  gh pr list                     │
│  #1  Add dark mode    feature   │
│  #2  Fix login bug    fix       │
│  #3  Update docs       docs     │
└─────────────────────────────────┘
    │
    用户选择: #2
    │
    ↓
审查 PR #2
═══════════════════════════════════

用户输入: /review-pr 2
    │
    ↓
直接审查 PR #2
```

#### 2. 审查维度

**原文：**
```markdown
### Overview, Code Quality, Correctness, Style & Conventions,
### Performance, Testing, Security, Documentation
```

**分析：**

8 个维度覆盖全面：

```
┌─────────────────────────────────────────────────┐
│              PR Review Framework                │
├─────────────────────────────────────────────────┤
│                                                 │
│  Overview        → 理解变更目的                 │
│  Code Quality    → 评估代码结构                 │
│  Correctness     → 验证功能正确性               │
│  Style           → 检查代码风格                 │
│  Performance     → 分析性能影响                 │
│  Testing         → 评估测试覆盖                 │
│  Security        → 识别安全风险                 │
│  Documentation   → 确认文档更新                 │
│                                                 │
└─────────────────────────────────────────────────┘
```

每个维度的具体检查项：

| 维度 | 关键问题 |
|------|---------|
| Overview | 做了什么？改了哪些文件？范围多大？ |
| Code Quality | 结构良好？有坏味道？命名规范？过度复杂？ |
| Correctness | 实现正确？有 bug？边界情况？错误处理？ |
| Style | 遵循约定？格式一致？导入有序？ |
| Performance | 性能问题？冗余操作？可优化？ |
| Testing | 包含测试？覆盖充分？测试质量？ |
| Security | 有漏洞？输入验证？凭证安全？ |
| Documentation | 文档完善？README 更新？API 文档？ |

#### 3. 优先级系统

**原文：**
```markdown
Prioritize feedback by severity (critical > major > minor > nit)
```

**分析：**

四级严重度分类：

```
Critical (关键)
    │
    ├─ 安全漏洞
    ├─ 数据丢失风险
    ├─ 系统崩溃
    └─ 必须修复
    │
Major (重要)
    │
    ├─ 功能缺陷
    ├─ 性能问题
    ├─ 错误处理缺失
    └─ 应该修复
    │
Minor (次要)
    │
    ├─ 代码异味
    ├─ 不一致性
    ├─ 缺少边界情况
    └─ 建议修复
    │
Nit (吹毛求疵)
    │
    ├─ 空格问题
    ├─ 变量命名
    ├─ 注释风格
    └─ 可选修复
```

#### 4. 建设性反馈

**原文：**
```markdown
Be constructive: Focus on actionable feedback
Suggest improvements, don't just point out problems
```

**分析：**

从"指出问题"到"提供解决方案"：

```
❌ 不好的反馈:
"This code is messy."

✅ 好的反馈:
"The function `processUserData` is doing too many things.
Consider splitting it into:
- `validateUserData()`
- `sanitizeUserData()`
- `saveUserData()"

This will improve testability and make the code easier to understand.
```

建设性反馈的特点：
- 具体位置
- 清晰问题
- 具体建议
- 解释原因

---

## 3. /security-review (安全审查命令)

**Token 数：2,610**

### 原始提示词内容

```markdown
You are an expert security analyst specializing in code security review.

Your task is to conduct a comprehensive security review of code changes.

## Review Focus Areas

### 1. Injection Vulnerabilities
- **SQL Injection**: Check for raw SQL queries with user input
- **Command Injection**: Look for shell command execution with user input
- **LDAP Injection**: Check LDAP queries with untrusted input
- **NoSQL Injection**: Check NoSQL queries with user input

### 2. Authentication & Authorization
- **Hardcoded Credentials**: Check for passwords, API keys, tokens
- **Weak Authentication**: Check for weak password policies
- **Authorization Bypass**: Check permission checks
- **Session Management**: Check session token handling

### 3. Cross-Site Scripting (XSS)
- **Reflected XSS**: Check for reflected user input
- **Stored XSS**: Check for stored user input
- **DOM XSS**: Check for dangerous JavaScript APIs
- **Content Security Policy**: Check CSP implementation

### 4. Sensitive Data Exposure
- **Logging**: Check if sensitive data is logged
- **Error Messages**: Check if error messages leak information
- **Debug Information**: Check for debug info in production
- **Comments**: Check for sensitive info in comments

### 5. Cryptography Issues
- **Weak Algorithms**: Check for MD5, SHA1, etc.
- **Hardcoded Keys**: Check for hardcoded encryption keys
- **Key Management**: Check key storage and rotation
- **Random Generation**: Check for weak random generation

### 6. Authorization & Access Control
- **Horizontal Privilege Escalation**: Check if user can access others' data
- **Vertical Privilege Escalation**: Check if user can elevate privileges
- **Missing Authorization**: Check for missing permission checks
- **Insecure Direct Object References**: Check IDOR vulnerabilities

### 7. Input Validation
- **Type Validation**: Check for type enforcement
- **Length Validation**: Check for length limits
- **Format Validation**: Check for format enforcement
- **Sanitization**: Check for proper input sanitization

### 8. API Security
- **Authentication**: Check API authentication
- **Rate Limiting**: Check for rate limiting
- **Input Validation**: Check API input validation
- **Output Encoding**: Check API output encoding

## Review Process

1. **Understand the Change**
   - What is the purpose of this change?
   - What data flows are introduced or modified?
   - What external inputs are involved?

2. **Analyze Each Focus Area**
   - Go through each security area systematically
   - Look for patterns that indicate vulnerabilities
   - Consider both obvious and subtle security issues

3. **Document Findings**
   For each vulnerability found:
   - **Severity**: Critical/High/Medium/Low
   - **Category**: Which focus area
   - **Location**: Specific file and line
   - **Description**: What is the vulnerability
   - **Impact**: What could an attacker do
   - **Recommendation**: How to fix it
   - **Code Example**: Show vulnerable and fixed code

4. **Prioritize Findings**
   - Critical: Immediate fix required
   - High: Should fix soon
   - Medium: Should fix in next cycle
   - Low: Nice to have

5. **Generate Report**
   ```
   ## Security Review Report

   ### Summary
   [X] Critical, [Y] High, [Z] Medium, [W] Low findings

   ### Critical Findings
   [... detailed findings ...]

   ### High Severity Findings
   [... detailed findings ...]

   ### Recommendations
   1. [Most important action items]
   2. [Additional improvements]
   ```

## Common Vulnerability Patterns

### SQL Injection Pattern
```javascript
// VULNERABLE:
const query = `SELECT * FROM users WHERE id = ${userId}`;

// SECURE:
const query = 'SELECT * FROM users WHERE id = ?';
await db.query(query, [userId]);
```

### Command Injection Pattern
```javascript
// VULNERABLE:
exec(`grep ${searchTerm} file.txt`);

// SECURE:
execFile('grep', [searchTerm, 'file.txt']);
```

### XSS Pattern
```javascript
// VULNERABLE:
div.innerHTML = userInput;

// SECURE:
div.textContent = userInput;
// or with sanitization:
div.innerHTML = DOMPurify.sanitize(userInput);
```

## Check Your Analysis

Before finalizing your review:
- [ ] Have I covered all 8 focus areas?
- [ ] Have I provided actionable recommendations?
- [ ] Have I included code examples for fixes?
- [ ] Have I prioritized by severity?
- [ ] Is the report clear and actionable?
```

### 设计分析

#### 1. OWASP Top 10 对齐

**分析：**

安全审查覆盖 OWASP Top 10：

| /security-review 聚焦领域 | OWASP Top 10 (2021) |
|-------------------------|---------------------|
| Injection Vulnerabilities | A01:2021 – Broken Access Control |
| Authentication & Authorization | A02:2021 – Cryptographic Failures |
| Cross-Site Scripting (XSS) | A03:2021 – Injection |
| Sensitive Data Exposure | A04:2021 – Insecure Design |
| Cryptography Issues | A05:2021 – Security Misconfiguration |
| Authorization & Access Control | A06:2021 – Vulnerable Components |
| Input Validation | A07:2021 – Authentication Failures |
| API Security | A08:2021 – Software/Data Integrity Failures |

#### 2. 系统化审查流程

**原文：**
```markdown
1. Understand the Change
2. Analyze Each Focus Area
3. Document Findings
4. Prioritize Findings
5. Generate Report
```

**分析：**

```
阶段1: 理解变更
    │
    ├─ 变更目的是什么?
    ├─ 引入了什么数据流?
    └─ 涉及哪些外部输入?
    │
    ↓
阶段2: 分析聚焦领域
    │
    ├─ Injection?
    ├─ Auth/Authz?
    ├─ XSS?
    ├─ Data Exposure?
    ├─ Crypto?
    ├─ Access Control?
    ├─ Input Validation?
    └─ API Security?
    │
    ↓
阶段3: 记录发现
    │
    ├─ Severity: Critical/High/Medium/Low
    ├─ Category: 聚焦领域
    ├─ Location: 文件:行号
    ├─ Description: 漏洞描述
    ├─ Impact: 攻击影响
    ├─ Recommendation: 修复建议
    └─ Code Example: 漏洞和修复代码
    │
    ↓
阶段4: 优先级排序
    │
    ├─ Critical: 立即修复
    ├─ High: 尽快修复
    ├─ Medium: 下周期修复
    └─ Low: 有时间修复
    │
    ↓
阶段5: 生成报告
    │
    └─ 汇总 → 优先展示 → 行动建议
```

#### 3. 漏洞模式教学

**原文：**
```markdown
## Common Vulnerability Patterns
### SQL Injection Pattern
### Command Injection Pattern
### XSS Pattern
```

**分析：**

通过示例教学，提高审查质量：

```
SQL Injection 示例:
┌─────────────────────────────────────────────┐
│  // VULNERABLE:                             │
│  const query = `SELECT * FROM users         │
│                WHERE id = ${userId}`;       │
│                                             │
│  // SECURE:                                 │
│  const query = 'SELECT * FROM users         │
│                WHERE id = ?';               │
│  await db.query(query, [userId]);           │
└─────────────────────────────────────────────┘
     │
     ├─ 展示错误模式
     ├─ 展示正确模式
     └─ 解释为什么安全
```

这种模式覆盖了：
- **可见性**: 一眼看出问题
- **对比性**: 错误 vs 正确
- **具体性**: 实际代码示例
- **可操作性**: 可以直接应用

#### 4. 结构化发现报告

**原文：**
```markdown
For each vulnerability found:
- Severity: Critical/High/Medium/Low
- Category: Which focus area
- Location: Specific file and line
- Description: What is the vulnerability
- Impact: What could an attacker do
- Recommendation: How to fix it
- Code Example: Show vulnerable and fixed code
```

**分析：**

每个漏洞发现都包含完整上下文：

```
┌───────────────────────────────────────────────────────────┐
│  Vulnerability Finding                                     │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  Severity:        Critical 🔴                              │
│  Category:        SQL Injection                           │
│  Location:        src/auth/login.ts:42                    │
│                                                           │
│  Description:                                                 │
│  User input is directly interpolated into SQL query        │
│  without parameterization, allowing SQL injection.         │
│                                                           │
│  Impact:                                                  │
│  An attacker could:                                         │
│  - Bypass authentication                                   │
│  - Extract all user data                                   │
│  - Modify or delete database records                      │
│                                                           │
│  Recommendation:                                           │
│  Use parameterized queries with prepared statements        │
│                                                           │
│  Code Example:                                              │
│  // VULNERABLE:                                            │
│  const query = `SELECT * FROM users WHERE id = '${id}'`;   │
│                                                           │
│  // SECURE:                                                │
│  const query = 'SELECT * FROM users WHERE id = $1';        │
│  await client.query(query, [id]);                          │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

#### 5. 自检清单

**原文：**
```markdown
Check Your Analysis:
- [ ] Have I covered all 8 focus areas?
- [ ] Have I provided actionable recommendations?
- [ ] Have I included code examples for fixes?
- [ ] Have I prioritized by severity?
- [ ] Is the report clear and actionable?
```

**分析：**

确保审查质量的自检机制：

```
完成性检查
    │
    ├─ 覆盖所有 8 个聚焦领域?
    ├─ 提供可操作建议?
    ├─ 包含代码示例?
    ├─ 按严重度优先级排序?
    └─ 报告清晰可操作?
    │
    全部 Yes → 可以提交
    │
    有 No → 补充/改进
```

---

## 三大 Slash 命令对比

| 维度 | /pr-comments | /review-pr | /security-review |
|------|-------------|-----------|-----------------|
| **Token 数** | 402 | 243 | 2,610 |
| **复杂度** | 低 | 中 | 高 |
| **主要功能** | 获取评论 | 全面审查 | 安全专项 |
| **输出形式** | 格式化评论列表 | 结构化审查报告 | 安全漏洞报告 |
| **专业深度** | 浅层（展示） | 中等（多维度） | 深度（安全专家） |
| **用途** | 快速查看评论 | 一般性代码审查 | 安全合规 |
| **严重度分级** | 无 | Critical > Nit | Critical > Low |

---

## 总结：Slash 命令的设计哲学

### 1. 快捷访问模式

```
传统方式:
    请求
    ↓
Claude 分析
    ↓
Claude 执行
    ↓
结果

Slash 命令:
    命令
    ↓
直接执行
    ↓
结果
```

### 2. 专业化 Agent

每个 Slash 命令都是一个专业化 Agent：

```
/pr-comments      →  评论聚合专家
                     (获取、解析、格式化)

/review-pr        →  代码审查专家
                     (多维度分析、建设性反馈)

/security-review  →  安全专家
                     (漏洞扫描、威胁分析、修复建议)
```

### 3. 标准化输出

所有 Slash 命令都产生标准化、结构化输出：

```
┌─────────────────────────────────────────┐
│           标准化输出结构                 │
├─────────────────────────────────────────┤
│                                         │
│  标题                                   │
│  ├─ 清晰标识                            │
│  ├─ 汇总信息                            │
│  └─ 严重度概览（如适用）                │
│                                         │
│  主体                                   │
│  ├─ 结构化章节                          │
│  ├─ 具体条目                            │
│  └─ 代码示例（如适用）                  │
│                                         │
│  可选：建议/行动项                      │
│                                         │
└─────────────────────────────────────────┘
```

---

*文档生成时间：2025年12月*
*基于 Claude Code v2.0.76*
