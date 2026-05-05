# Hermes Web UI — API 參考文件（第二部分）

> **← 第一部分**：[API_SURFACE_part1.md](API_SURFACE_part1.md)（認證、CSRF、Error Handling、SSE 協定）  
> **→ 第三部分**：[API_SURFACE_part3.md](API_SURFACE_part3.md)（Skills/Cron/Memory、Auth、Onboarding、完整清單）

## 5. Core — 健康、Session 和 Chat

### Health

#### `GET /health`

健康檢查，不需要認證。

**Response 200**：
```json
{"status": "ok"}
```

---

### Session 管理

#### `GET /api/session?session_id=<sid>`

取得單一 session 的完整資料（含 messages）。

**Query Params**：
- `session_id`（必要）
- `messages=0`：略過 messages payload，加快 sidebar 切換速度

**Response 200**：
```json
{
  "session": {
    "session_id": "abc123def456",
    "title": "My Conversation",
    "model": "anthropic/claude-sonnet-4.6",
    "workspace": "/home/user/project",
    "messages": [...],
    "created_at": 1714000000,
    "updated_at": 1714001000
  }
}
```

---

#### `GET /api/sessions`

列出所有 session（含 CLI sessions、按 `last_message_at` 倒序排列）。

**Query Params**：
- `all_profiles=1`：顯示所有 profile 的 sessions（預設只顯示當前 profile）

**Response 200**：
```json
{
  "sessions": [...],
  "cli_count": 5,
  "all_profiles": false,
  "active_profile": "default",
  "other_profile_count": 3,
  "server_time": 1714000000.0,
  "server_tz": "+0800"
}
```

---

#### `GET /api/session/status?session_id=<sid>`

查詢 session 是否有 active stream。

**Response 200**：
```json
{"active": true, "stream_id": "abc123"}
```

---

#### `POST /api/session/new`

建立新 session。

**Request Body**：
```json
{
  "workspace": "/home/user/project",
  "model": "anthropic/claude-sonnet-4.6",
  "model_provider": "anthropic",
  "profile": "default",
  "project_id": "proj_abc"
}
```

**Response 200**：
```json
{
  "session": {
    "session_id": "abc123",
    "title": "Untitled",
    "messages": [],
    ...
  }
}
```

---

#### `POST /api/session/duplicate`

複製現有 session（深度複製 messages）。

**Request Body**：`{"session_id": "abc123"}`

**Response 200**：`{"session": {...}}`

---

#### `POST /api/session/rename`

重新命名 session。

**Request Body**：`{"session_id": "abc123", "title": "新標題"}`

---

#### `POST /api/session/delete`

刪除 session（及其對應的 JSON 檔案）。

**Request Body**：`{"session_id": "abc123"}`

---

#### `POST /api/session/clear`

清空 session 中所有 messages。

**Request Body**：`{"session_id": "abc123"}`

---

#### `POST /api/session/branch`

從 session 建立分支（branch），可指定 message index。

**Request Body**：`{"session_id": "abc123", "message_index": 5}`

---

#### `POST /api/session/pin`

釘選/取消釘選 session。

**Request Body**：`{"session_id": "abc123", "pinned": true}`

---

#### `POST /api/session/archive`

封存/取消封存 session。

**Request Body**：`{"session_id": "abc123", "archived": true}`

---

#### `POST /api/session/move`

移動 session 至不同 project。

**Request Body**：`{"session_id": "abc123", "project_id": "proj_xyz"}`

---

#### `POST /api/session/update`

更新 session 的 metadata（如 workspace、model）。

---

#### `POST /api/session/compress`

壓縮 session context（去除多餘 token，縮短 context window）。

---

#### `POST /api/session/truncate`

截斷 session 到指定 message 數量。

---

#### `POST /api/session/retry`

重試最後一條 user message。

---

#### `POST /api/session/undo`

撤銷最後一條 assistant 回覆。

---

#### `POST /api/session/yolo`

切換 YOLO 模式（自動核准所有工具呼叫）。

`GET /api/session/yolo?session_id=<sid>`：查詢目前 YOLO 狀態。

---

#### `POST /api/session/toolsets`

設定 session 啟用的 toolsets 清單。

**Request Body**：`{"session_id": "abc123", "toolsets": ["file", "terminal", "web"]}`

---

#### `GET /api/session/export?session_id=<sid>`

匯出 session 為 JSON 檔案（下載）。

---

#### `POST /api/session/import`

匯入 session JSON 檔案。

---

#### `POST /api/session/import_cli`

從 hermes-agent CLI 的 SQLite DB 匯入 session。

---

#### `GET /api/sessions/search?q=<query>`

全文搜尋 session 標題。

---

#### `POST /api/sessions/cleanup`

清理空 session。

---

#### `GET /api/session/usage?session_id=<sid>`

取得 session 的 token 使用量統計。

---

#### `GET /api/session/conversation-rounds?session_id=<sid>`

取得 session 的對話輪次數。

---

#### `GET /api/session/handoff-summary?session_id=<sid>`

取得 session 的摘要（用於 context handoff）。

---

### Chat（核心對話）

#### `POST /api/chat/start`

啟動一次 agent 對話，建立 SSE stream。

**Request Body**：
```json
{
  "session_id": "abc123",
  "message": "請幫我分析這段程式碼",
  "model": "anthropic/claude-sonnet-4.6",
  "model_provider": "anthropic",
  "workspace": "/home/user/project",
  "attachments": []
}
```

**Response 200**：
```json
{
  "stream_id": "xyz789",
  "session_id": "abc123"
}
```

接著用 `stream_id` 建立 SSE 連線：`GET /api/chat/stream?stream_id=xyz789`

---

#### `GET /api/chat/stream?stream_id=<id>`

SSE 串流端點。保持長連線，逐一推送 SSE events（詳見第 4 節）。

**Headers**：
```
Content-Type: text/event-stream; charset=utf-8
Cache-Control: no-cache
X-Accel-Buffering: no
Connection: keep-alive
```

串流結束條件：收到 `stream_end`、`error`、或 `cancel` event。

---

#### `GET /api/chat/stream/status?stream_id=<id>`

查詢指定 stream 是否仍在執行中。

**Response 200**：`{"active": true, "stream_id": "xyz789"}`

---

#### `GET /api/chat/cancel?stream_id=<id>`

取消正在執行的 agent stream。

**Response 200**：`{"ok": true, "cancelled": true, "stream_id": "xyz789"}`

---

#### `POST /api/chat`

同步（非串流）chat，等待完整回覆後才回應（較少用）。

---

#### `POST /api/chat/steer`

在 agent 執行中途注入額外指示（mid-stream steering）。

**Request Body**：`{"session_id": "abc123", "text": "請用繁體中文回覆"}`

---

#### `POST /api/btw`

Ephemeral（短暫）對話，不儲存到 session history。

---

### Approval（人工核准）

#### `GET /api/approval/pending?session_id=<sid>`

查詢等待核准的危險命令。

---

#### `GET /api/approval/stream?session_id=<sid>`

SSE 串流：等待核准事件推送。

---

#### `POST /api/approval/respond`

回覆核准請求。

**Request Body**：`{"session_id": "abc123", "approved": true}`

---

### Clarify（澄清問題）

#### `GET /api/clarify/pending?session_id=<sid>`

查詢等待回答的澄清問題。

---

#### `GET /api/clarify/stream?session_id=<sid>`

SSE 串流：等待澄清問題推送。

---

#### `POST /api/clarify/respond`

回答澄清問題。

**Request Body**：`{"session_id": "abc123", "choice": "選項A"}`

---

### Settings & Models

#### `GET /api/settings`

取得使用者設定（不含 `password_hash`）。

**Response 200**：
```json
{
  "model": "anthropic/claude-sonnet-4.6",
  "send_key": "enter",
  "show_cli_sessions": false,
  "password_env_var": false,
  "webui_version": "v0.50.245",
  "agent_version": "..."
}
```

---

#### `POST /api/settings`

儲存使用者設定。特殊欄位：
- `_set_password`：設定登入密碼
- `_clear_password`：清除密碼

若 `HERMES_WEBUI_PASSWORD` 環境變數已設定，修改密碼操作 → HTTP 409。

---

#### `GET /api/models`

取得可用的 AI model 清單（含 provider 資訊）。

---

#### `GET /api/models/live`

即時從 AI provider 取得最新 model 清單。

---

#### `POST /api/default-model`

設定預設 model。

---

#### `GET /api/reasoning`

取得 reasoning（Chain-of-Thought）設定狀態。

---

#### `POST /api/reasoning`

更新 reasoning 設定（`show_reasoning`、`reasoning_effort`）。

---

#### `GET /api/updates/check`

查詢是否有新版本可用（向 GitHub Releases API 查詢）。

**Response 200**：
```json
{
  "available": true,
  "current": "v0.50.245",
  "latest": "v0.50.291",
  "url": "https://github.com/nesquena/hermes-webui/releases/..."
}
```

---

#### `POST /api/updates/apply` / `POST /api/updates/force`

觸發自我更新。

---

#### `GET /api/insights`

取得統計分析資料（message count 等，來自 `state_sync.py`）。

---

#### `POST /api/admin/reload`

重新載入設定（無需重啟伺服器）。

---

## 6. Workspace — 檔案與工作目錄

### Workspaces

#### `GET /api/workspaces`

列出已登錄的 workspace 清單。

---

#### `GET /api/workspaces/suggest`

自動建議可用的 workspace 目錄。

---

#### `POST /api/workspaces/add`

新增 workspace 到清單。

**Request Body**：`{"path": "/home/user/project"}`

---

#### `POST /api/workspaces/remove`

移除 workspace。

**Request Body**：`{"path": "/home/user/project"}`

---

#### `POST /api/workspaces/rename`

重新命名 workspace 顯示名稱。

---

#### `POST /api/workspaces/reorder`

調整 workspace 排列順序。

---

### 檔案操作

#### `GET /api/list?path=<dir>&session_id=<sid>`

列出目錄內容（workspace sandbox 內）。

---

#### `GET /api/file?path=<file>&session_id=<sid>`

讀取檔案內容（metadata + 內容）。

---

#### `GET /api/file/raw?path=<file>&session_id=<sid>`

讀取檔案原始位元組（用於下載）。

---

#### `POST /api/file/save`

儲存檔案內容。

**Request Body**：`{"path": "src/main.py", "content": "...", "session_id": "abc123"}`

---

#### `POST /api/file/create`

建立新檔案。

---

#### `POST /api/file/create-dir`

建立新目錄。

---

#### `POST /api/file/rename`

重新命名檔案/目錄。

---

#### `POST /api/file/delete`

刪除檔案（限 workspace sandbox 內）。

---

#### `POST /api/file/reveal`

在系統檔案管理員中顯示檔案位置（桌面環境）。

---

#### `GET /api/git-info?session_id=<sid>`

取得 workspace 的 git 狀態（branch、uncommitted changes 等）。

---

#### `GET /api/media?path=<file>&session_id=<sid>`

提供圖片/媒體檔案（用於 chat 中的圖片預覽）。

---

### Rollback（版本回滾）

#### `GET /api/rollback/list?workspace=<path>`

列出 workspace 可用的 checkpoint 清單。

---

#### `GET /api/rollback/diff?workspace=<path>&checkpoint=<id>`

預覽 checkpoint 與目前狀態的差異（diff）。

---

#### `POST /api/rollback/restore`

還原到指定 checkpoint。

**Request Body**：`{"workspace": "/home/user/project", "checkpoint": "ckpt_abc"}`

---

### Upload & Transcribe

#### `POST /api/upload`

上傳檔案附件（multipart/form-data）。

---

#### `POST /api/upload/extract`

解壓縮上傳的 ZIP 或 tar 檔案。

---

#### `POST /api/transcribe`

語音轉文字（Audio → text，Web Speech API 輔助）。

---

### Projects

#### `GET /api/projects`

列出所有 project（session 分組）。

**Query Params**：`all_profiles=1`（顯示所有 profile 的 projects）

---

#### `POST /api/projects/create`

建立新 project。

**Request Body**：`{"name": "My Project"}`

---

#### `POST /api/projects/rename`

重新命名 project。

---

#### `POST /api/projects/delete`

刪除 project（不刪除其中的 sessions）。

---

