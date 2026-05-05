# Hermes Web UI — API 參考文件 Part 2（Workspace、Skills/Cron/Memory、Auth、Onboarding、完整清單）

> **← Part 1**：[API_SURFACE_part1.md](API_SURFACE_part1.md)（認證、CSRF、Error Handling、SSE 協定、Core）  
> 版本：v0.50.245（May 2026）  
> 來源程式碼：`api/routes.py`（handle_post L2878、handle_patch L4062、handle_delete L4074）

---

## 目錄

6. [Workspace — 檔案與工作目錄](#6-workspace--檔案與工作目錄)
7. [Skills / Cron / Memory](#7-skills--cron--memory)
8. [Auth — 認證 Endpoints](#8-auth--認證-endpoints)
9. [Onboarding — 初始設定精靈](#9-onboarding--初始設定精靈)
10. [完整 Endpoint 清單](#10-完整-endpoint-清單)

---

## 6. Workspace — 檔案與工作目錄

### Workspaces

| Endpoint | Method | 說明 | Request |
|----------|--------|------|---------|
| `/api/workspaces` | GET | 列出已登錄的 workspace 清單 | — |
| `/api/workspaces/suggest` | GET | 自動建議可用的 workspace 目錄 | — |
| `/api/workspaces/add` | POST | 新增 workspace | `{"path": "/home/user/project"}` |
| `/api/workspaces/remove` | POST | 移除 workspace | `{"path": "/home/user/project"}` |
| `/api/workspaces/rename` | POST | 重新命名顯示名稱 | `{"path": "...", "name": "..."}` |
| `/api/workspaces/reorder` | POST | 調整排列順序 | `{"paths": [...]}` |

---

### 檔案操作

所有檔案操作都限制在 workspace sandbox 內（path traversal 保護）。

#### `GET /api/list?path=<dir>&session_id=<sid>`

列出目錄內容（檔案清單、類型、大小）。

---

#### `GET /api/file?path=<file>&session_id=<sid>`

讀取檔案內容（返回 metadata + 文字內容）。

---

#### `GET /api/file/raw?path=<file>&session_id=<sid>`

讀取檔案原始位元組（用於下載）。

---

#### `POST /api/file/save`

儲存檔案內容。

**Request Body**：`{"path": "src/main.py", "content": "...", "session_id": "abc123"}`

---

#### 檔案操作快速參考

| Endpoint | Method | 說明 | Request 必要欄位 |
|----------|--------|------|-----------------|
| `/api/file/create` | POST | 建立新檔案 | `path`, `session_id` |
| `/api/file/create-dir` | POST | 建立新目錄 | `path`, `session_id` |
| `/api/file/rename` | POST | 重新命名 | `path`, `new_path`, `session_id` |
| `/api/file/delete` | POST | 刪除檔案 | `path`, `session_id` |
| `/api/file/reveal` | POST | 在 Finder/Explorer 顯示 | `path`, `session_id` |

---

#### `GET /api/git-info?session_id=<sid>`

取得 workspace 的 git 狀態（branch、uncommitted changes 等）。

---

#### `GET /api/media?path=<file>&session_id=<sid>`

提供圖片/媒體檔案（用於 chat 中的圖片預覽）。

---

### Rollback（版本回滾）

| Endpoint | Method | 說明 | Params |
|----------|--------|------|--------|
| `/api/rollback/list` | GET | 列出 checkpoints | `?workspace=` |
| `/api/rollback/diff` | GET | 預覽 diff | `?workspace=&checkpoint=` |
| `/api/rollback/restore` | POST | 還原到 checkpoint | `{"workspace":"...","checkpoint":"..."}` |

---

### Upload & Transcribe

| Endpoint | Method | 說明 |
|----------|--------|------|
| `/api/upload` | POST | 上傳檔案附件（multipart/form-data） |
| `/api/upload/extract` | POST | 解壓縮上傳的 ZIP 或 tar 檔案 |
| `/api/transcribe` | POST | 語音轉文字（Audio → text） |

---

### Projects

| Endpoint | Method | 說明 | Request |
|----------|--------|------|---------|
| `/api/projects` | GET | 列出 projects（`all_profiles=1`） | — |
| `/api/projects/create` | POST | 建立 project | `{"name": "My Project"}` |
| `/api/projects/rename` | POST | 重新命名 | `{"project_id":"...","name":"..."}` |
| `/api/projects/delete` | POST | 刪除 project（sessions 不受影響） | `{"project_id":"..."}` |

---

### Terminal

#### `POST /api/terminal/start`

在 workspace 目錄中啟動終端機 session（`api/routes.py:4277`）。

**Request Body**：
```json
{
  "session_id": "abc123",
  "rows": 24,
  "cols": 80,
  "restart": false
}
```

**Response 200**：`{"ok": true, "session_id": "abc123", "workspace": "...", "running": true}`

---

#### Terminal 操作快速參考

| Endpoint | Method | 說明 | Request |
|----------|--------|------|---------|
| `/api/terminal/input` | POST | 傳送輸入（限 8192 bytes） | `{"session_id":"...","data":"ls\r"}` |
| `/api/terminal/resize` | POST | 調整視窗大小 | `{"session_id":"...","rows":40,"cols":120}` |
| `/api/terminal/close` | POST | 關閉終端機 | `{"session_id":"..."}` |
| `/api/terminal/output` | GET | SSE：終端機輸出推送 | `?session_id=` |

---

## 7. Skills / Cron / Memory

### Skills（技能）

| Endpoint | Method | 說明 | Request/Params |
|----------|--------|------|----------------|
| `/api/skills` | GET | 列出所有 skills | — |
| `/api/skills/content` | GET | 取得 skill 內容 | `?name=<skill>` |
| `/api/skills/save` | POST | 儲存（新增或更新） | `{"name":"my_skill","content":"..."}` |
| `/api/skills/delete` | POST | 刪除 skill | `{"name":"my_skill"}` |

---

### Cron Jobs

Cron 操作均透過 `cron_profile_context()` 確保寫入正確 profile 的 `jobs.json`（`api/routes.py:3450-3480`）。

#### `GET /api/crons`

列出所有 cron jobs（來自 hermes-agent `cron.jobs`）。

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

#### Cron 操作快速參考

| Endpoint | Method | 說明 | Request/Params |
|----------|--------|------|----------------|
| `/api/crons/output` | GET | Cron job 輸出 | `?job_id=` |
| `/api/crons/history` | GET | 執行歷史 | — |
| `/api/crons/recent` | GET | 最近執行清單 | — |
| `/api/crons/status` | GET | Cron 系統狀態 | — |
| `/api/crons/update` | POST | 更新 cron 設定 | `{"job_id":"...","schedule":"..."}` |
| `/api/crons/delete` | POST | 刪除 cron job | `{"job_id":"..."}` |
| `/api/crons/run` | GET/POST | 立即執行 | `job_id` |
| `/api/crons/pause` | POST | 暫停 | `{"job_id":"..."}` |
| `/api/crons/resume` | POST | 恢復 | `{"job_id":"..."}` |

---

### Memory

| Endpoint | Method | 說明 | Request |
|----------|--------|------|---------|
| `/api/memory` | GET | 讀取 `MEMORY.md` + `USER.md` | — |
| `/api/memory/write` | POST | 寫入 memory | `{"content":"...","file":"MEMORY"}` |

---

## 8. Auth — 認證 Endpoints

### `GET /api/auth/status`

查詢認證狀態（在 PUBLIC_PATHS 中，不需認證）。

**Response 200**：`{"auth_enabled": true, "logged_in": true}`

---

### `POST /api/auth/login`

登入（`api/routes.py:3995`）。

**Request Body**：`{"password": "mypassword"}`

**Response 200**（成功）：`{"ok": true}`
- 同時設定 `Set-Cookie: hermes_session=<token>; HttpOnly; SameSite=Lax; Max-Age=2592000`

**Response 401**（失敗）：`{"error": "Invalid password"}`

**Response 429**（限流）：`{"error": "Too many attempts. Try again in a minute."}`

---

### `POST /api/auth/logout`

登出，清除 cookie 並 invalidate session token（`api/routes.py:4027`）。

**Response 200**：`{"ok": true}`

---

### Profile 管理

#### `GET /api/profiles`

列出所有 profiles 及當前 active profile。

**Response 200**：`{"profiles": [...], "active": "default"}`

---

#### `GET /api/profile/active`

取得當前 active profile 名稱及路徑。

**Response 200**：`{"name": "default", "path": "/home/user/.hermes"}`

---

#### Profile 操作快速參考

| Endpoint | Method | 說明 | Request |
|----------|--------|------|---------|
| `/api/profile/switch` | POST | 切換 profile（設定 Cookie） | `{"name":"work"}` |
| `/api/profile/create` | POST | 建立新 profile | `{"name":"work"}` |
| `/api/profile/delete` | POST | 刪除 profile | `{"name":"work"}` |
| `/api/personality/set` | POST | 設定 AI 個性（persona） | `{"name":"..."}` |

---

### Providers

| Endpoint | Method | 說明 |
|----------|--------|------|
| `/api/providers` | GET | 列出已設定的 AI provider 及 API key 狀態 |
| `/api/providers` | POST | 新增或更新 provider 設定（API key、base URL） |
| `/api/providers/delete` | POST | 刪除 provider 設定 |

---

### Gateway

#### `GET /api/gateway/status`

取得 messaging gateway（Telegram/Discord/Slack 等）連線狀態（`api/routes.py:2798`）。

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

### MCP Servers

| Endpoint | Method | 說明 |
|----------|--------|------|
| `/api/mcp/servers` | GET | 列出已設定的 MCP server |
| `/api/mcp/tools` | GET | 列出可用的 MCP tools |

---

### 其他管理

| Endpoint | Method | 說明 |
|----------|--------|------|
| `/api/plugins` | GET | 列出已安裝的 WebUI 擴充套件 |
| `/api/background/status` | GET | 背景任務狀態 |
| `/api/background` | POST | 觸發背景任務 |
| `/api/dashboard/status` | GET | Dashboard 伺服器狀態 |
| `/api/dashboard/config` | GET/POST | 讀取/更新 dashboard 設定 |
| `/api/insights` | GET | 統計分析（message count 等） |
| `/api/admin/reload` | POST | 重新載入設定（無需重啟） |
| `/api/updates/apply` | POST | 套用更新 |
| `/api/updates/force` | POST | 強制更新 |
| `/api/commands` | GET | Slash commands 清單（autocomplete） |
| `/api/personalities` | GET | Personality 清單 |

---

## 9. Onboarding — 初始設定精靈

Onboarding 端點在**未啟用認證**時限制只允許本地網路（loopback 或 private IP）存取，避免遠端使用者竊取 API keys（`api/routes.py:3680-3731`）。

環境變數 `HERMES_WEBUI_ONBOARDING_OPEN=1` 可放寬此限制（用於有防火牆保護的遠端伺服器）。

---

#### `GET /api/onboarding/status`

查詢 onboarding 是否已完成（不需認證）。

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

套用 onboarding 設定（寫入 API keys 到 `~/.hermes/`）。**僅本地網路**。

**Request Body**：
```json
{
  "provider": "anthropic",
  "api_key": "sk-ant-...",
  "model": "claude-sonnet-4.6"
}
```

---

#### `POST /api/onboarding/probe`

探測自建 provider endpoint 是否可連接（`api/routes.py:3742`）。

**Request Body**：`{"base_url": "http://localhost:11434", "api_key": "optional"}`

---

#### Onboarding 操作快速參考

| Endpoint | Method | 說明 |
|----------|--------|------|
| `/api/onboarding/complete` | POST | 標記 onboarding 已完成 |
| `/api/onboarding/oauth/start` | POST | 啟動 OAuth 授權流程（**僅本地網路**） |
| `/api/onboarding/oauth/poll` | GET | 輪詢 OAuth 授權狀態 |
| `/api/onboarding/oauth/cancel` | POST | 取消 OAuth 授權流程 |

---

## 10. 完整 Endpoint 清單

### GET 端點（53 個）

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
| `/api/background/status` | 背景任務狀態 |
| `/api/dashboard/status` | Dashboard 狀態 |
| `/api/dashboard/config` | Dashboard 設定 |
| `/api/insights` | 統計分析 |

### POST 端點（65 個）

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
| PATCH | `/api/kanban/*` | Kanban board 操作（`api/kanban_bridge.py`） |
| DELETE | `/api/kanban/*` | 刪除 Kanban 項目 |

---

*文件最後更新：2026-05-05*  
*程式碼來源：`api/routes.py`（handle_get L2033、handle_post L2878、handle_patch L4062、handle_delete L4074）*
