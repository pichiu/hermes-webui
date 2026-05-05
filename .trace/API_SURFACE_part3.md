# Hermes Web UI — API 參考文件（第三部分）

> **← 第二部分**：[API_SURFACE_part2.md](API_SURFACE_part2.md)（Core Session/Chat、Workspace）  
> 版本：v0.51.2（May 2026，增量更新：2026-05-05）

## 7. Skills / Cron / Memory

### Skills（技能）

#### `GET /api/skills`

列出所有 skills（來自 hermes-agent `tools.skills_tool`）。

---

#### `GET /api/skills/content?name=<skill>`

取得指定 skill 的完整內容。

---

#### `POST /api/skills/save`

儲存（新增或更新）skill。

**Request Body**：`{"name": "my_skill", "content": "..."}`

---

#### `POST /api/skills/delete`

刪除 skill。

**Request Body**：`{"name": "my_skill"}`

---

### Cron Jobs

#### `GET /api/crons`

列出所有 cron jobs（來自 hermes-agent `cron.jobs`）。

---

#### `GET /api/crons/output?job_id=<id>`

取得 cron job 的輸出內容。

---

#### `GET /api/crons/history`

取得 cron job 執行歷史。

---

#### `GET /api/crons/recent`

取得最近執行的 cron job 清單。

---

#### `GET /api/crons/status`

取得 cron 系統狀態。

---

#### `POST /api/crons/create`

建立新 cron job。

**Request Body**：
```json
{
  "name": "daily_report",
  "schedule": "0 9 * * *",
  "task": "生成每日報告"
}
```

---

#### `POST /api/crons/update`

更新 cron job 設定。

---

#### `POST /api/crons/delete`

刪除 cron job。

**Request Body**：`{"job_id": "cron_abc"}`

---

#### `POST /api/crons/run` （GET 亦支援）

立即手動執行 cron job。

---

#### `POST /api/crons/pause`

暫停 cron job。

---

#### `POST /api/crons/resume`

恢復暫停的 cron job。

---

### Memory

#### `GET /api/memory`

讀取 agent memory（`MEMORY.md` + `USER.md`）。

---

#### `POST /api/memory/write`

寫入 agent memory。

**Request Body**：`{"content": "使用者偏好...", "file": "MEMORY"}`

---

## 8. Auth — 認證

#### `GET /api/auth/status`

查詢認證狀態（不需要認證即可存取）。

**Response 200**：
```json
{
  "auth_enabled": true,
  "logged_in": true
}
```

---

#### `POST /api/auth/login`

登入（驗證密碼，建立 session）。

**Request Body**：`{"password": "mypassword"}`

**Response 200**（成功）：
```json
{"ok": true}
```
同時設定 `Set-Cookie: hermes_session=<token>; HttpOnly; SameSite=Lax; Max-Age=2592000`

**Response 401**（失敗）：`{"error": "Invalid password"}`

**Response 429**（限流）：`{"error": "Too many attempts. Try again in a minute."}`

---

#### `POST /api/auth/logout`

登出（清除 cookie，invalidate session token）。

**Response 200**：`{"ok": true}`

---

### Profile 管理

#### `GET /api/profiles`

列出所有 profiles 及當前 active profile。

**Response 200**：
```json
{
  "profiles": [...],
  "active": "default"
}
```

---

#### `GET /api/profile/active`

取得當前 active profile 名稱及路徑。

**Response 200**：
```json
{
  "name": "default",
  "path": "/home/user/.hermes"
}
```

---

#### `POST /api/profile/switch`

切換 active profile（設定 Cookie，不影響其他並行 tab）。

**Request Body**：`{"name": "work"}`

---

#### `POST /api/profile/create`

建立新 profile。

**Request Body**：`{"name": "work"}`

---

#### `POST /api/profile/delete`

刪除 profile。

---

#### `POST /api/personality/set`

設定 AI 個性（persona）。

---

### Providers

#### `GET /api/providers`

列出已設定的 AI provider 及 API key 狀態。

---

#### `POST /api/providers`

新增或更新 provider 設定（API key、base URL 等）。

---

#### `POST /api/providers/delete`

刪除 provider 設定。

---

### Gateway

#### `GET /api/gateway/status`

取得 messaging gateway（Telegram/Discord/Slack 等）連線狀態。

**Response 200**：
```json
{
  "running": true,
  "platforms": [
    {"name": "telegram", "label": "Telegram"},
    {"name": "discord", "label": "Discord"}
  ],
  "last_active": "2026-05-05T10:30:00",
  "session_count": 12
}
```

---

#### `GET /api/sessions/gateway/stream`

SSE 串流：gateway session 即時更新推送（sidebar badge 更新）。

---

### MCP Servers

#### `GET /api/mcp/servers`

列出已設定的 MCP（Model Context Protocol）server。

---

#### `GET /api/mcp/tools`

列出可用的 MCP tools。

---

### 其他管理

#### `GET /api/dashboard/status`

Dashboard 伺服器狀態（若有部署）。

---

#### `GET /api/dashboard/config` / `POST /api/dashboard/config`

讀取/更新 dashboard 設定。

---

#### `GET /api/plugins`

列出已安裝的 WebUI 擴充套件（extensions）。

---

#### `GET /api/background/status`

背景任務狀態。

---

#### `POST /api/background`

觸發背景任務。

---

### Terminal

#### `POST /api/terminal/start`

在 workspace 目錄中啟動終端機 session。

**Request Body**：
```json
{
  "session_id": "abc123",
  "rows": 24,
  "cols": 80,
  "restart": false
}
```

---

#### `POST /api/terminal/input`

傳送輸入到終端機（限 8192 bytes）。

**Request Body**：`{"session_id": "abc123", "data": "ls -la\r"}`

---

#### `POST /api/terminal/resize`

調整終端機視窗大小（SIGWINCH）。

**Request Body**：`{"session_id": "abc123", "rows": 40, "cols": 120}`

---

#### `POST /api/terminal/close`

關閉終端機 session。

---

#### `GET /api/terminal/output?session_id=<sid>`

SSE 串流：終端機輸出即時推送。

---

## 9. Onboarding — 初始設定精靈

Onboarding 端點在**未啟用認證**時限制只允許本地網路（loopback 或 private IP）存取，避免遠端使用者竊取 API keys。環境變數 `HERMES_WEBUI_ONBOARDING_OPEN=1` 可放寬此限制（用於有防火牆保護的遠端伺服器）。

---

#### `GET /api/onboarding/status`

查詢 onboarding 是否已完成。

**Response 200**：
```json
{
  "completed": false,
  "providers_configured": [],
  "step": "welcome"
}
```

---

#### `POST /api/onboarding/setup`

套用 onboarding 設定（寫入 API keys 到 `~/.hermes/`）。

**Request Body**：
```json
{
  "provider": "anthropic",
  "api_key": "sk-ant-...",
  "model": "claude-sonnet-4.6"
}
```

**限制**：僅本地網路（或 `HERMES_WEBUI_ONBOARDING_OPEN=1`）

---

#### `POST /api/onboarding/complete`

標記 onboarding 已完成。

---

#### `POST /api/onboarding/probe`

探測自建 provider endpoint 是否可連接（讀取 `/models` 清單）。

**Request Body**：
```json
{
  "base_url": "http://localhost:11434",
  "api_key": "optional"
}
```

---

#### `POST /api/onboarding/oauth/start`

啟動 OAuth 授權流程（用於 provider OAuth 認證）。

**限制**：同 `/api/onboarding/setup`

---

#### `GET /api/onboarding/oauth/poll`

輪詢 OAuth 授權狀態。

---

#### `POST /api/onboarding/oauth/cancel`

取消 OAuth 授權流程。

---

#### `GET /api/commands`

取得可用的 slash commands 清單（用於前端 autocomplete）。

---

#### `GET /api/personalities`

取得可用的 personality 清單。

---

## 10. Logs & Wiki Status（v0.51.2 新增）

### `GET /api/logs`

取得指定 log 檔案的最後 N 行內容。

**Query Parameters**：

| 參數 | 必填 | 說明 | 範例 |
|------|------|------|------|
| `file` | 是 | Log 檔案類型；允許值：`agent`、`errors`、`gateway` | `?file=agent` |
| `tail` | 否 | 回傳行數；允許值：`100`、`200`、`500`、`1000`，預設 `200` | `?tail=500` |

**回應（200 OK）**：

```json
{
  "lines": [
    "2026-05-05 12:00:01 INFO  Agent started",
    "2026-05-05 12:00:02 WARNING Low memory",
    "2026-05-05 12:00:03 ERROR Connection failed"
  ],
  "file": "agent",
  "tail": 200
}
```

**錯誤回應**：

| 狀態碼 | 條件 |
|--------|------|
| 400 | `file` 參數不在白名單（`agent` / `errors` / `gateway`）或 `tail` 不在允許值內 |
| 404 | Log 檔案不存在（agent 尚未啟動過） |

**實作細節**：
- 白名單：`{"agent": "agent.log", "errors": "errors.log", "gateway": "gateway.log"}`（`api/routes.py`）
- `tail` 正規化函式 `_normalize_logs_tail(raw_tail)` — 若無效值則 fallback 至 200
- 最大讀取 4 MB（保護超大 log 檔）
- 前端 `panelLogs`（`static/panels.js`）每 5 秒自動 refresh（`_logsAutoRefreshTimer`）
- 行級別顏色渲染：WARNING=黃、ERROR/CRITICAL=紅

---

### `GET /api/wiki/status`

取得 LLM Wiki 整合狀態。

**回應（200 OK）**：

```json
{
  "enabled": true,
  "path": "/home/user/.hermes/wiki",
  "page_count": 42,
  "config_source": "config.yaml",
  "last_checked": 1746446400.0
}
```

| 欄位 | 型別 | 說明 |
|------|------|------|
| `enabled` | `bool` | LLM Wiki 功能是否啟用 |
| `path` | `str \| null` | Wiki 目錄的絕對路徑（`null` 表示未設定） |
| `page_count` | `int` | Wiki 目錄下的 Markdown 頁面數量 |
| `config_source` | `str` | 設定來源（`"config.yaml"` 或 `"env"`） |
| `last_checked` | `float` | 最後一次檢查的 Unix timestamp |

**實作細節**：
- 後端函式：`_build_llm_wiki_status()`（`api/routes.py`）
- 多個 `_llm_wiki_*` 輔助函式負責路徑解析、config 讀取、檔案計數
- 前端顯示在 Insights panel 的 `wiki-status-card`（`static/panels.js`、`static/index.html`）

---

## 11. 完整 Endpoint 清單

### GET 端點

| Path | 說明 |
|------|------|
| `/health` | 健康檢查 |
| `/api/auth/status` | 認證狀態 |
| `/api/models` | 可用 model 清單 |
| `/api/models/live` | 即時 model 清單 |
| `/api/settings` | 使用者設定 |
| `/api/reasoning` | Reasoning 設定 |
| `/api/sessions` | Session 清單 |
| `/api/session` | 單一 session 詳情 |
| `/api/session/status` | Session stream 狀態 |
| `/api/session/yolo` | YOLO 模式狀態 |
| `/api/session/usage` | Token 使用量 |
| `/api/session/export` | 匯出 session |
| `/api/session/conversation-rounds` | 對話輪次 |
| `/api/session/handoff-summary` | Session 摘要 |
| `/api/sessions/search` | 搜尋 sessions |
| `/api/projects` | Project 清單 |
| `/api/workspaces` | Workspace 清單 |
| `/api/workspaces/suggest` | Workspace 建議 |
| `/api/list` | 目錄內容 |
| `/api/file` | 讀取檔案 |
| `/api/file/raw` | 下載檔案原始資料 |
| `/api/git-info` | Git 狀態 |
| `/api/media` | 媒體檔案 |
| `/api/chat/stream` | SSE Chat 串流 |
| `/api/chat/stream/status` | Stream 狀態 |
| `/api/chat/cancel` | 取消 Stream |
| `/api/approval/pending` | 待核准項目 |
| `/api/approval/stream` | Approval SSE |
| `/api/clarify/pending` | 待澄清問題 |
| `/api/clarify/stream` | Clarify SSE |
| `/api/terminal/output` | 終端機輸出 SSE |
| `/api/sessions/gateway/stream` | Gateway SSE |
| `/api/skills` | Skills 清單 |
| `/api/skills/content` | Skill 內容 |
| `/api/crons` | Cron jobs 清單 |
| `/api/crons/output` | Cron job 輸出 |
| `/api/crons/history` | Cron 歷史 |
| `/api/crons/recent` | 最近 Cron |
| `/api/crons/status` | Cron 狀態 |
| `/api/memory` | Agent memory |
| `/api/profiles` | Profile 清單 |
| `/api/profile/active` | Active profile |
| `/api/providers` | Provider 設定 |
| `/api/plugins` | 擴充套件 |
| `/api/gateway/status` | Gateway 狀態 |
| `/api/mcp/servers` | MCP servers |
| `/api/mcp/tools` | MCP tools |
| `/api/rollback/list` | Checkpoint 清單 |
| `/api/rollback/diff` | Checkpoint diff |
| `/api/onboarding/status` | Onboarding 狀態 |
| `/api/onboarding/oauth/poll` | OAuth 輪詢 |
| `/api/updates/check` | 更新檢查 |
| `/api/commands` | Slash commands |
| `/api/personalities` | Personality 清單 |
| `/api/logs` | Log 檔案內容（`?file=agent\|errors\|gateway&tail=200`） |
| `/api/wiki/status` | LLM Wiki 整合狀態 |
| `/api/background/status` | 背景任務狀態 |
| `/api/dashboard/status` | Dashboard 狀態 |
| `/api/dashboard/config` | Dashboard 設定 |
| `/api/insights` | 統計分析 |

### POST 端點

| Path | 說明 |
|------|------|
| `/api/auth/login` | 登入 |
| `/api/auth/logout` | 登出 |
| `/api/session/new` | 建立 session |
| `/api/session/duplicate` | 複製 session |
| `/api/session/rename` | 重新命名 |
| `/api/session/delete` | 刪除 session |
| `/api/session/clear` | 清空 messages |
| `/api/session/branch` | 建立分支 |
| `/api/session/update` | 更新 metadata |
| `/api/session/truncate` | 截斷 session |
| `/api/session/compress` | 壓縮 context |
| `/api/session/retry` | 重試最後訊息 |
| `/api/session/undo` | 撤銷回覆 |
| `/api/session/yolo` | 切換 YOLO |
| `/api/session/toolsets` | 設定 toolsets |
| `/api/session/pin` | 釘選 session |
| `/api/session/archive` | 封存 session |
| `/api/session/move` | 移動至 project |
| `/api/session/import` | 匯入 session |
| `/api/session/import_cli` | 匯入 CLI session |
| `/api/sessions/cleanup` | 清理空 sessions |
| `/api/sessions/cleanup_zero_message` | 清理零訊息 sessions |
| `/api/projects/create` | 建立 project |
| `/api/projects/rename` | 重新命名 project |
| `/api/projects/delete` | 刪除 project |
| `/api/chat/start` | 啟動 chat stream |
| `/api/chat` | 同步 chat |
| `/api/chat/steer` | Mid-stream steering |
| `/api/btw` | Ephemeral 對話 |
| `/api/background` | 背景任務 |
| `/api/approval/respond` | 核准/拒絕工具 |
| `/api/clarify/respond` | 回答澄清問題 |
| `/api/terminal/start` | 啟動終端機 |
| `/api/terminal/input` | 終端機輸入 |
| `/api/terminal/resize` | 調整終端機大小 |
| `/api/terminal/close` | 關閉終端機 |
| `/api/skills/save` | 儲存 skill |
| `/api/skills/delete` | 刪除 skill |
| `/api/crons/create` | 建立 cron job |
| `/api/crons/update` | 更新 cron job |
| `/api/crons/delete` | 刪除 cron job |
| `/api/crons/run` | 立即執行 cron |
| `/api/crons/pause` | 暫停 cron |
| `/api/crons/resume` | 恢復 cron |
| `/api/memory/write` | 寫入 memory |
| `/api/profile/switch` | 切換 profile |
| `/api/profile/create` | 建立 profile |
| `/api/profile/delete` | 刪除 profile |
| `/api/personality/set` | 設定 personality |
| `/api/providers` | 儲存 provider |
| `/api/providers/delete` | 刪除 provider |
| `/api/settings` | 儲存設定 |
| `/api/reasoning` | 更新 reasoning |
| `/api/default-model` | 設定預設 model |
| `/api/workspaces/add` | 新增 workspace |
| `/api/workspaces/remove` | 移除 workspace |
| `/api/workspaces/rename` | 重新命名 workspace |
| `/api/workspaces/reorder` | 重新排序 workspace |
| `/api/file/save` | 儲存檔案 |
| `/api/file/create` | 建立檔案 |
| `/api/file/create-dir` | 建立目錄 |
| `/api/file/rename` | 重新命名檔案 |
| `/api/file/delete` | 刪除檔案 |
| `/api/file/reveal` | 在 Finder/Explorer 顯示 |
| `/api/rollback/restore` | 還原 checkpoint |
| `/api/upload` | 上傳檔案 |
| `/api/upload/extract` | 解壓縮檔案 |
| `/api/transcribe` | 語音轉文字 |
| `/api/onboarding/setup` | 套用 onboarding |
| `/api/onboarding/complete` | 完成 onboarding |
| `/api/onboarding/probe` | 探測 provider |
| `/api/onboarding/oauth/start` | 啟動 OAuth |
| `/api/onboarding/oauth/cancel` | 取消 OAuth |
| `/api/updates/apply` | 套用更新 |
| `/api/updates/force` | 強制更新 |
| `/api/admin/reload` | 重新載入設定 |
| `/api/dashboard/config` | 更新 Dashboard 設定 |

### PATCH / DELETE 端點

| Method | Path Pattern | 說明 |
|--------|-------------|------|
| PATCH | `/api/kanban/*` | Kanban board 操作（由 `api/kanban_bridge.py` 處理） |
| DELETE | `/api/kanban/*` | 刪除 Kanban 項目 |

---

*文件最後更新：2026-05-05*  
*程式碼來源：`api/routes.py`（handle_get L2033、handle_post L2878、handle_patch L4062、handle_delete L4074）、`api/streaming.py`、`api/auth.py`、`api/helpers.py`*
