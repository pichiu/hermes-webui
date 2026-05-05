# Stage 2.3 核心領域邏輯

## 專案的「心臟」

Hermes WebUI 的核心邏輯分兩個層面：

1. **SSE streaming engine**（`api/streaming.py`）— 協調 AI agent 執行與前端即時串流
2. **Session state machine**（`api/models.py` + `static/sessions.js`）— 管理對話生命週期

---

## 1. SSE Streaming Engine（`api/streaming.py`）

### 兩端點協作架構

```
POST /api/chat/start
  → 建立 queue.Queue
  → STREAMS[stream_id] = queue
  → 啟動 daemon thread（_run_agent_streaming）
  → 立即回傳 {stream_id}

GET /api/chat/stream?stream_id=X（長連線）
  → 讀取 STREAMS[stream_id]
  → queue.get(timeout=5) loop：
      timeout → ": heartbeat\n\n"（防 proxy timeout）
      event   → "data: {json}\n\n"
      'done'/'error' → 關閉連線
```

**關鍵設計選擇**：SSE 而非 WebSocket（ADR-004）
- 單向（server → client）足夠
- 標準 browser EventSource API
- 不需要 upgrade handshake

### Cancel 機制

```
DELETE /api/chat/stream 或 Cancel 按鈕
    → CANCEL_FLAGS[stream_id] = True
    → _run_agent_streaming() 定期檢查 CANCEL_FLAGS
    → 停止 agent → finally 清理 STREAMS[stream_id]
```

`AGENT_INSTANCES[stream_id] = agent` 儲存 AIAgent 實例，允許外部呼叫 `agent.cancel()`（若支援）。

### Thread 安全設計

| 層次 | 機制 | 保護對象 |
|------|------|---------|
| `_ENV_LOCK` | 全域 Lock | os.environ 寫入（不同 session 間） |
| `SESSION_AGENT_LOCKS[sid]` | Per-session Lock | 同一 session 的並發 agent run |
| `_set_thread_env()` | threading.local | Thread-local env 備份（TERMINAL_CWD 等） |
| `STREAMS_LOCK` | Lock | STREAMS dict 讀寫 |
| `LOCK` | Lock | SESSIONS OrderedDict |

⚠️ **已知限制**（TD1，PARTIAL）：`os.environ` 仍有 process-global 寫入作為 fallback。不同 session 的並發 agent run 仍可能互相干擾。`_ENV_LOCK` 是 interim fix，完整解方需要 terminal tool 讀取 thread-local。

---

## 2. Session Model（`api/models.py:Session`）

### Session 資料結構

```python
Session:
  session_id   str     # uuid4().hex[:12]，12 字元 hex
  title        str     # 自動從第一條訊息設定，限 64 字元
  workspace    str     # 絕對路徑，agent 工作目錄
  model        str     # model ID，e.g. "anthropic/claude-sonnet-4.6"
  messages     list    # OpenAI 格式訊息陣列
  created_at   float   # Unix timestamp
  updated_at   float   # Unix timestamp，每次 save() 更新
  pinned       bool    # 釘選到 sidebar 頂部
  archived     bool    # 封存（隱藏但不刪除）
  project_id   str|None  # FK 到 projects.json
  profile      str     # 建立時的 active profile
  tool_calls   list    # tool call dicts（Sprint 10）
```

### 原子寫入模式

```python
def save(self):
    tmp = path.with_suffix(f'.tmp.{os.getpid()}.{threading.get_ident()}')
    tmp.write_text(json.dumps(self.__dict__))
    os.replace(tmp, self.path)  # 原子操作
    _write_session_index(...)   # 同步更新 _index.json
```

`.bak` 備份：每次 save 前先寫 `.bak`（`#1558` 修復資料遺失問題）。

### Session Index 機制

```
sessions/
  {sid}.json        一個 session
  _index.json       compact 摘要清單
  *.tmp.*           原子寫入的中間檔（重啟時清理）
  *.bak             備份檔（session recovery 用）
```

`all_sessions()` 讀取 `_index.json`（O(1)）而非掃描目錄（O(N)）（Phase C 修復 TD6）。

### LRU Cache

```python
SESSIONS = collections.OrderedDict()   # 最多 SESSIONS_MAX=100 個 in-memory
SESSIONS_MAX = 100
# get_session() 使用 move_to_end()
# 超過 100 時 popitem(last=False) 淘汰最舊的
```

---

## 3. Title Generation（`api/streaming.py`）

智慧化 session title 生成，是 UX 的重要細節：

```
run_conversation() 完成後
    ↓
_maybe_schedule_title_refresh(session, put_event, agent)
    ↓
    若 title 是 provisional（第一條訊息前 64 字元）
    → 啟動 background title generation thread
    → 嘗試 aux title（獨立 LLM 呼叫，快速小 model）
    → fallback：從對話摘取
    → SSE 送出 'title_update' 事件
    → 前端即時更新 sidebar session 標題
```

相關函式：`_generate_llm_session_title_via_aux()`、`_fallback_title_from_exchange()`、`_sanitize_generated_title()`

---

## 4. Frontend State Machine（`static/ui.js`）

### 全域狀態 `S`

```javascript
const S = {
  session:      null,     // 當前 Session compact dict
  messages:     [],       // 當前 session 的完整訊息陣列
  entries:      [],       // workspace 目錄內容
  busy:         false,    // agent 執行中（禁用 Send）
  pendingFiles: []        // 待上傳的 File objects
}
const INFLIGHT = {}       // session_id → {messages, uploaded}（in-flight 狀態）
```

### Session 切換的 INFLIGHT 機制

```
send() 開始 → INFLIGHT[sid] = {messages: [...], uploaded: [...]}
              也寫入 localStorage（頁面刷新後恢復）

loadSession(sid) → 先查 INFLIGHT[sid]
                → 若存在：顯示 in-flight 狀態（含 pending tool cards）
                → 若不存在：從 server 載入

done event → delete INFLIGHT[sid]，clearInflight(sid) from localStorage
```

---

## 5. Markdown Rendering Pipeline（`static/ui.js:renderMd`）

**Streaming 模式**（token-by-token，使用 `streaming-markdown` vendor lib）：
- SSE token 事件 → `streaming-markdown` 增量更新 DOM
- rAF throttling 避免 layout thrashing

**靜態模式**（`renderMd(raw)`，歷史訊息渲染）：
手工 regex chain，處理順序：
1. 暫存 fenced code blocks / inline code（防被後續 regex 誤處理）
2. 轉換 `<strong>/<em>/<br>` → markdown 等效
3. Mermaid blocks → `<div class="mermaid-block">`（Mermaid.js 非同步渲染）
4. Code blocks → `<pre><code>` with Prism.js 語法高亮
5. Bold/Italic/Headings/HR/Blockquote/Lists/Links/Tables
6. Safety net：未知 HTML tag 用 SAFE_TAGS allowlist 過濾（XSS 防護）
7. 段落 wrapping

**已知缺陷**：巢狀 list、bold+link 同行混排有可能渲染錯誤（B8，PARTIAL）。

---

## 6. Profile 系統（`api/profiles.py`）

Profile = 獨立的 hermes-agent 環境（獨立 `~/.hermes/<profile>/`）

```python
get_active_profile_name()     # 讀取 active profile（優先 request thread-local）
set_request_profile(name)     # 設定 per-request profile（Handler 在每個請求開頭設定）
clear_request_profile()       # 清除（finally block）

switch_profile(name)          # 切換 active profile：
                               # 1. 更新 active_profile 檔案
                               # 2. _reload_dotenv(new_home)（載入新 .env）
                               # 3. 重新載入 config、skills、memory、models、cron
```

Context manager 模式：
```python
with ProfileContext(home):    # 進入時 switch，離開時還原
    agent.run_conversation(...)
```

---

## 7. Auth 機制（`api/auth.py`）

```
HERMES_WEBUI_PASSWORD env var 或 Settings 設定
    → 啟用 auth
    → 每個請求 check_auth()：
        - PUBLIC_PATHS 直通（/health, /login, /manifest.json 等）
        - 否則：讀取 Cookie 'hermes_session'
        - HMAC 驗證 cookie token
        - sessions dict 中確認未過期（TTL: 30 天）
        - 失敗 → 302 redirect 到 /login（GET）或 401（API）

POST /api/auth/login：
    - bcrypt-like HMAC 驗證密碼
    - 成功 → 生成 token → 寫入 _sessions → Set-Cookie HttpOnly SameSite=Strict

Rate limiting：IP-based，login 失敗計數（_login_attempts dict）
```
