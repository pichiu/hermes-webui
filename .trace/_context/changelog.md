# Changelog 分析

**Base Commit 範圍**: `e23ba59..fcc8328`  
**日期**: 2026-05-05  
**版本變更**: v0.50.245 → v0.51.2

---

## Commit 歷史

```
fcc8328 Merge pull request #1682 from nesquena/stage-299
e095ed9 chore(release): stamp v0.51.2 — 3-PR follow-up + #1669 scroll hotfix
e2748fe Apply Opus pre-release SHOULD-FIX (absorbed in stage-299)
4e9ec6f fix(sidebar): scroll jumps back to 0 on small lists (≤80 sessions) — #1669 follow-up
d76ef2a Cover CLI compression lineage filtering
8981d33 Fix CLI session CI compatibility
79d0762 Filter low-value CLI agent sessions
af1c628 feat: add logs tab MVP
2684d6f feat: add LLM Wiki status panel
```

---

## 變更檔案清單（排除 tests/ 和 docs/pr-media/）

| 狀態 | 檔案 | 類別 | 行數變更 |
|------|------|------|---------|
| M | `api/routes.py` | API 介面 / 核心邏輯 | +349 行 |
| M | `api/agent_sessions.py` | 核心邏輯 | +123 行 |
| M | `api/models.py` | 資料模型 | +18 行 |
| M | `static/panels.js` | Extension / 前端 | +184 行 |
| M | `static/sessions.js` | Extension / 前端 | +42 行 |
| M | `static/index.html` | Extension / 前端 | +49 行 |
| M | `static/i18n.js` | Extension / 前端 | +123 行 |
| M | `static/style.css` | Extension / 前端 | +45 行 |
| M | `CHANGELOG.md` | 無影響 | +47 行 |
| M | `ROADMAP.md` | 無影響 | +2 行 |
| M | `TESTING.md` | 無影響 | +4 行 |

**有影響的變更檔案**：8 個  
**總檔案數**：484  
**變更幅度**：8/484 ≈ **1.7%**（輕量更新）

---

## 功能變更摘要

### 1. 新 Feature：Logs Tab（`feat: add logs tab MVP`，`af1c628`）

**後端**（`api/routes.py`）：
- 新函式 `_normalize_logs_tail(raw_tail)` — 限制 tail 值為 [100, 200, 500, 1000]
- 新函式 `_handle_logs(handler, parsed)` — 處理 GET `/api/logs`
- 新端點 `GET /api/logs?file=agent|errors|gateway&tail=200` — 回傳指定 log 檔的最後 N 行

**前端**（`static/panels.js`, `static/index.html`）：
- 新 panel：`panelLogs`（sidebar 新增 "Logs" tab）
- Log 檔案選擇器（agent / errors / gateway）
- Tail 行數選擇器（100 / 200 / 500 / 1000）
- Auto-refresh 每 5 秒（`_logsAutoRefreshTimer`）
- 行級別顏色（WARNING=黃、ERROR/CRITICAL=紅）
- Wrap lines toggle
- Copy all 按鈕

### 2. 新 Feature：LLM Wiki Status Panel（`feat: add LLM Wiki status panel`，`2684d6f`）

**後端**（`api/routes.py`）：
- 多個新函式：`_llm_wiki_*`（路徑解析、config 讀取、檔案計數）
- 新函式 `_build_llm_wiki_status()` — 回傳 LLM Wiki 整合狀態
- 新函式 `_handle_llm_wiki_status(handler, parsed)` — 處理 GET `/api/wiki/status`
- 新端點 `GET /api/wiki/status`

**前端**（`static/panels.js`, `static/index.html`）：
- Insights panel 中新增 `wiki-status-card`
- 顯示 LLM Wiki 的啟用狀態、路徑、頁面數量等

### 3. CLI Session Filtering（PR #1587，`79d0762`-`d76ef2a`）

**後端**（`api/agent_sessions.py`）：
- 新常數 `CLI_MIN_UNTITLED_MESSAGE_COUNT = 6`
- 新常數 `CLI_MIN_UNTITLED_USER_MESSAGE_COUNT = 2`
- 新函式 `_looks_like_default_cli_title(row)` — 偵測低價值的 CLI session（標題如 "Untitled"、"cli"、"cli session"）
- 新函式 `_normalize_source_name(value)` — 正規化 source 名稱
- 過濾邏輯：untitled CLI sessions 需有 ≥6 messages 且 ≥2 user messages 才顯示

**後端**（`api/models.py`）：
- 新常數 `CLI_VISIBLE_SESSION_LIMIT = 20` — CLI sessions 上限（sidebar 最多顯示 20 個 CLI sessions）
- 新函式 `_message_role(message)` — 安全取得訊息 role
- Session.compact() 新增欄位 `user_message_count`
- `all_sessions()` 的 CLI session 載入邏輯傳入 `limit=CLI_VISIBLE_SESSION_LIMIT`

**前端**（`static/sessions.js`）：
- 新函式 `_isCliSession(session)` — 更健壯的 CLI session 偵測（多 source 欄位、避免誤判 messaging sessions）

### 4. Sidebar Scroll Fix（#1669，`4e9ec6f`）

- 修復：session 列表 ≤80 個時 scroll 跳回 0 的 bug
- 影響 `static/sessions.js`

---

## 版本資訊

**發版版本**：`v0.51.2`（stamp commit: `e095ed9`）
