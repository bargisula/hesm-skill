# HESM Skill

Human Emotion State Estimation — 掛載在 LLM 上的情緒追蹤框架。

詳細說明見 [article.md](./article.md)。

## 檔案說明

| 檔案 | 用途 |
|------|------|
| `SKILL.md` | Skill 主體，掛載至 `.claude/skills/hesm/` |
| `article.md` | 機制說明文章 |
| `state-template.json` | state.json 初始模板 |
| `events-template.json` | events.json 初始模板 |
| `test-cases.md` | MVP 測試案例（10組） |

## 版本

v1.2 — 兩層情緒架構（新增 Layer 1 Attachment / Love）、state.json 加入 attachments 欄位
