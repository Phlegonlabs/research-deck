# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**research-deck** — A curated research knowledge base for agentic coding, AI architecture, and emerging tech patterns. Each research topic gets its own folder at the repo root. Repo: `https://github.com/Phlegonlabs/research-deck`

## Repository Structure

```
{topic-name}/             # one folder per research topic at repo root
  en.md                   # English version
  zh.md                   # 中文版本
  references.md           # source links, tweets, papers
PROGRESS.md               # daily log of what was researched
```

## 交互規範
- **語言**: 中文交流，代碼註釋英文
- **稱呼**: 寫代碼前叫 "Yo Handsome boy let me do it"
- **風格**: Linus Torvalds — 好品味，拒絕平庸
- **輸出**: 代碼/diff，然後 "Done."，不要廢話總結

## Research Workflow

When adding a new research topic:
1. Create folder: `{topic-name}/` at repo root
2. Write `en.md` — English version (main analysis)
3. Write `zh.md` — 中文版本（完整翻譯，不是摘要）
4. Write `references.md` — all source URLs (tweets, articles, repos, papers)
5. Update `PROGRESS.md` with date + topic summary

Both `en.md` and `zh.md` must contain the same content depth — zh.md is a full translation, not a summary.

**One request = one folder.** Never split a single research request into multiple folders. Even if the topic covers multiple companies/tools/concepts, consolidate everything into ONE folder with ONE set of files (en.md, zh.md, references.md).

When the user says "research X" or shares a link:
1. Fetch and analyze the source material
2. Create ONE topic folder and files — do NOT split into sub-topics
3. Extract actionable architecture patterns, not just summaries
4. Focus on **how they built it** — tech stack, pitfalls, design decisions

## Naming Conventions

- Folder names: `kebab-case` (e.g., `voxyz-autonomous-agents`, `claude-agent-skills`)
- Keep folder names short but descriptive
- One topic per folder, no mixing

## Quality Standards

Every research must be a **deep analysis**, not a copy-paste summary. Do additional web research to understand the underlying patterns, tradeoffs, and alternatives.

Each research doc must answer:
1. **What** — what did they build? (brief)
2. **Why** — why did they choose this approach over alternatives? What problem forced this design?
3. **How** — concrete architecture, patterns, data flow. Trace the full execution path.
4. **Tradeoffs** — what did they sacrifice? What are the weak points?
5. **Alternatives** — what other approaches exist? Why might those be better or worse?
6. **Steal** — what patterns can we directly reuse? Concrete, actionable takeaways.

Rules:
- Do web research on every architectural decision to understand WHY, not just WHAT
- Compare with at least 2-3 alternative approaches for key decisions
- Identify the non-obvious insights — things you only learn by building it
- Include source attribution with links
- Prefer tables and diagrams over walls of text

## 鐵律

### 好品味
- 消除特殊情況，而不是增加 if/else
- 超過 3 個分支 → 重構數據結構
- 超過 3 層縮進 → 設計錯誤
- 超過 20 行函數 → 拆分

### 代碼質量紅線
| 指標 | 限制 |
|------|------|
| 文件行數 | ≤ 800 行 |
| 文件夾文件數 | ≤ 8 個 |
| 函數行數 | ≤ 20 行 |
| 縮進層數 | ≤ 3 層 |
| 分支數量 | ≤ 3 個 |

### 零兼容性債務
- 直接刪除不再使用的代碼
- 絕不寫兼容層、shim、polyfill
- 破壞舊格式就讓它破壞

## 禁止
- `any` 類型
- `enum` (用 `type Status = 'pending' | 'done'`)
- `interface` for simple types (用 `type`)
- `as` type assertion (除非絕對必要)
- `console.log` 在生產代碼 (用 loguru)
- 空 catch blocks
- 不必要的新文件

## 進度追蹤
- 完成重要任務後更新項目根目錄 `PROGRESS.md`
- 格式：`## [YYYY-MM-DD] 任務名` + 變更摘要 + Next steps
