# Stage 2.1 Entry Points

## 程式啟動方式

### 1. 標準啟動（`start.sh` / `bootstrap.py`）

```
start.sh  →  bootstrap.py
```

**`start.sh`**（`start.sh`）：薄 shell wrapper，偵測 Python 可執行檔後呼叫 `bootstrap.py`。

**`bootstrap.py`** 啟動流程：
1. 偵測 hermes-agent 目錄（多策略搜尋）
2. 找 Python executable（agent venv → local .venv → system python3）
3. 驗證依賴（pyyaml）
4. 啟動 `server.py`（子程序 or 直接 exec）
5. 輪詢 `/health` 直到 server ready
6. 除非 `--no-browser`，否則開啟瀏覽器
7. 若是首次執行，跳出 onboarding wizard

### 2. Daemon 啟動（`ctl.sh`）

```
ctl.sh start  →  bootstrap.py（背景 daemon mode）→ PID 寫入 ~/.hermes/webui.pid
ctl.sh status / logs / restart / stop
```

日誌寫入 `~/.hermes/webui.log`。

### 3. 直接啟動（`server.py`）

```bash
HERMES_WEBUI_PORT=8787 venv/bin/python /path/to/server.py
```

**`server.py:main()`** 初始化流程（`server.py:191-280`）：
1. `print_startup_config()` — 印出設定摘要
2. `_raise_fd_soft_limit(4096)` — 提升 RLIMIT_NOFILE（macOS launchd 問題）
3. `fix_credential_permissions()` — 強制 `.env` 等敏感檔案為 0600
4. `recover_all_sessions_on_startup(SESSION_DIR)` — 從 `.bak` 還原資料遺失的 sessions（#1558）
5. `start_watcher()` — 啟動 gateway session watcher（SSE 推送 Telegram/Discord/Slack sessions）
6. 建立 `QuietHTTPServer((HOST, PORT), Handler)`
7. TLS 設定（若 `HERMES_WEBUI_TLS_CERT` 和 `HERMES_WEBUI_TLS_KEY` 都設定）
8. `httpd.serve_forever()`

### 4. Docker 啟動

```
docker_init.bash  →  設定 UID/GID  →  server.py
```

三種 compose 設定：
- `docker-compose.yml`：單容器（agent in-process）
- `docker-compose.two-container.yml`：agent + WebUI 分容器
- `docker-compose.three-container.yml`：agent + dashboard + WebUI

---

## HTTP 請求處理初始化

**`server.py:Handler`** 類別（`server.py:40-155`）：

```
ThreadingHTTPServer ─▶ Handler(BaseHTTPRequestHandler)
    setup()          — 設定 TCP_NODELAY + SO_KEEPALIVE（Linux TCP_KEEPIDLE/KEEPINTVL/KEEPCNT）
    do_GET()         — 驗證 auth → handle_get(routes.py)
    do_POST()        — 驗證 auth → handle_post(routes.py)
    do_PATCH()       — 驗證 auth → handle_patch(routes.py)
    do_DELETE()      — 驗證 auth → handle_delete(routes.py)
    log_request()    — JSON structured log
```

**每個請求**的初始化步驟（`server.py:108-145`）：
1. `_req_t0 = time.time()`（計時）
2. `get_profile_cookie(self)` — 從 cookie 取得 profile name
3. `set_request_profile(cookie_profile)` — 設定 thread-local profile context
4. `check_auth(self, parsed)` — 若啟用密碼認證，驗證 cookie
5. 路由到對應 handler
6. `clear_request_profile()` — 清除 thread-local（finally block）

---

## 模組初始化（import time）

**`api/config.py`** 在 import 時執行：
- `_discover_agent_dir()` — 多策略搜尋 hermes-agent 目錄
- `sys.path.insert(0, str(AGENT_DIR))` — 讓 hermes-agent modules 可 import
- 載入 `~/.hermes/config.yaml`（若存在）
- 初始化全域狀態：`SESSIONS = OrderedDict()`、`STREAMS = {}`、`CANCEL_FLAGS = {}`
- 偵測 hermes-agent 版本與可用 toolsets

**`api/gateway_watcher.py`** 在 `start_watcher()` 時：
- 啟動背景 thread 監視 hermes-agent 的 gateway session 資料庫
- 有新 session 時透過 SSE 推送給前端

---

## 前端初始化

**`static/index.html`** 載入順序：
1. `<head>` 中：preload workspace panel state（`documentElement.dataset.workspacePanel`，防止 first-paint flash）
2. 外部 CDN：Prism.js、Mermaid.js（deferred，帶 SRI hash）
3. `<body>` 末尾依序載入：
   - `static/vendor/streaming-markdown.js`（vendored，無 CDN）
   - `static/icons.js`
   - `static/i18n.js`
   - `static/ui.js`
   - `static/workspace.js`
   - `static/sessions.js`
   - `static/messages.js`
   - `static/panels.js`
   - `static/commands.js`
   - `static/onboarding.js`
   - `static/boot.js`（含 boot IIFE）

**`static/boot.js` 的 boot IIFE**：
1. `_initResizePanels()` — 設定可拖曳面板寬度
2. 初始化行動導覽（hamburger sidebar）
3. 初始化 voice input（Web Speech API，不支援時隱藏）
4. `localStorage.getItem('hermes-webui-session')` — 取得上次 session ID
5. `loadSession(savedSid)` — 若有，載入上次 session；否則顯示空白狀態
6. **絕不**自動建立新 session（RULE-6）
