# High Pressure Answer Templates Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add detailed high-pressure interview answer templates to the existing NL2DSL interview answer guide.

**Architecture:** This is a documentation-only change. The existing answer guide remains intact, and a new chapter is appended at the end so the original answer structure is not disrupted.

**Tech Stack:** Markdown, Git, PowerShell text inspection.

---

### Task 1: Append Detailed Answer Templates

**Files:**
- Modify: `docs/高宇-NL2DSL简历面试参考答案.md`

- [x] **Step 1: Read the target document ending**

Run:

```powershell
Get-Content -Encoding UTF8 -LiteralPath 'D:\claude_work\work\jianli\docs\高宇-NL2DSL简历面试参考答案.md' -Tail 80
```

Expected: The current ending section is visible so the new chapter can be appended without rewriting existing content.

- [x] **Step 2: Append the new chapter**

Add a final chapter titled `## 十一、高压追问详细回答模板`.

The chapter must include these groups:

- 项目真实性与边界
- Vibe Coding 掌握度
- 性能指标可信度
- RAG 与向量库选型
- 大数据与 Spark 真实性
- 代码现场讲解
- 生产落地边界
- Java 后端基本盘

Each group must contain:

- `### 总回答口径`
- `### 详细回答模板`
- `### 不要这么说`

- [x] **Step 3: Verify required headings exist**

Run:

```powershell
Select-String -Path 'D:\claude_work\work\jianli\docs\高宇-NL2DSL简历面试参考答案.md' -Pattern '十一、高压追问详细回答模板|项目真实性与边界|Vibe Coding 掌握度|性能指标可信度|RAG 与向量库选型|大数据与 Spark 真实性|代码现场讲解|生产落地边界|Java 后端基本盘'
```

Expected: Every required chapter or group heading appears at least once.

- [x] **Step 4: Verify content does not include unresolved placeholders**

Run:

```powershell
Select-String -Path 'D:\claude_work\work\jianli\docs\高宇-NL2DSL简历面试参考答案.md' -Pattern 'TODO|TBD|待定|占位'
```

Expected: No matches.

- [x] **Step 5: Review the diff**

Run:

```powershell
git diff -- 'docs/高宇-NL2DSL简历面试参考答案.md'
```

Expected: The diff only appends the new detailed answer chapter and does not rewrite unrelated existing sections.

- [x] **Step 6: Commit the documentation change**

Run:

```powershell
git add -- 'docs/高宇-NL2DSL简历面试参考答案.md' 'docs/superpowers/plans/2026-07-03-high-pressure-answer-templates.md'
git commit -m "docs: add high pressure answer templates"
```

Expected: Git creates one commit containing the plan and appended answer templates.
