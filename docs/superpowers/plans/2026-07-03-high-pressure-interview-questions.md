# High Pressure Interview Questions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a high-pressure interview question chapter to the existing NL2DSL interviewer follow-up checklist.

**Architecture:** This is a documentation-only change. The existing follow-up checklist remains the source document, and the new chapter is appended at the end to avoid disrupting current sections.

**Tech Stack:** Markdown, Git, PowerShell text inspection.

---

### Task 1: Append High-Pressure Follow-Up Chapter

**Files:**
- Modify: `docs/高宇-NL2DSL简历面试官追问清单.md`

- [x] **Step 1: Read the target document ending**

Run:

```powershell
Get-Content -Encoding UTF8 -LiteralPath 'D:\claude_work\work\jianli\docs\高宇-NL2DSL简历面试官追问清单.md' -Tail 80
```

Expected: The output shows the current ending section so the new chapter can be appended without overwriting existing content.

- [x] **Step 2: Append the new chapter**

Add a final chapter titled `## 十一、高压追问与防穿帮问题`.

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

- `### 高频高压问题`
- `### 连环追问链`
- `### 回答底线`

- [x] **Step 3: Verify required headings exist**

Run:

```powershell
Select-String -Path 'D:\claude_work\work\jianli\docs\高宇-NL2DSL简历面试官追问清单.md' -Pattern '十一、高压追问与防穿帮问题|项目真实性与边界|Vibe Coding 掌握度|性能指标可信度|RAG 与向量库选型|大数据与 Spark 真实性|代码现场讲解|生产落地边界|Java 后端基本盘'
```

Expected: Every required chapter or group heading appears at least once.

- [x] **Step 4: Verify content does not include unresolved placeholders**

Run:

```powershell
Select-String -Path 'D:\claude_work\work\jianli\docs\高宇-NL2DSL简历面试官追问清单.md' -Pattern 'TODO|TBD|待定|占位'
```

Expected: No matches.

- [x] **Step 5: Review the diff**

Run:

```powershell
git diff -- 'docs/高宇-NL2DSL简历面试官追问清单.md'
```

Expected: The diff only appends the new high-pressure chapter and does not rewrite unrelated existing sections.

- [x] **Step 6: Commit the documentation change**

Run:

```powershell
git add -- 'docs/高宇-NL2DSL简历面试官追问清单.md' 'docs/superpowers/plans/2026-07-03-high-pressure-interview-questions.md'
git commit -m "docs: add high pressure interview questions"
```

Expected: Git creates one commit containing the plan and appended interview question chapter.
