# Stage 2.4 Extension Points

## Plugin / Extension 機制

---

## 1. WebUI Extensions（`api/extensions.py`）

最正式的擴充機制。透過環境變數注入自訂 script 和 stylesheet。

**設定方式**：
```bash
# 方式 A：提供本地目錄
HERMES_WEBUI_EXTENSION_DIR=/path/to/extensions ./start.sh

# 方式 B：指定 URL（同源，comma-separated）
HERMES_WEBUI_EXTENSION_SCRIPT_URLS=http://localhost:8787/extensions/my.js
HERMES_WEBUI_EXTENSION_STYLESHEET_URLS=http://localhost:8787/extensions/my.css
```

**機制**：
- `HERMES_WEBUI_EXTENSION_DIR` 指向的目錄會被 serve 在 `/extensions/`
- Server 在回傳 `index.html` 時，在 `</body>` 前注入 `<script>` 和 `<link>` tags
- 只接受同源 URL（安全防護）
- 詳細文件：`docs/EXTENSIONS.md`

**使用場景**：自訂 UI 元素、注入 analytics、加入自訂 panel。

---

## 2. 主題系統（`static/style.css` + `THEMES.md`）

CSS custom properties based。

**內建主題**（7 個）：
- Dark（預設）、Light、Slate、Solarized Dark、Monokai、Nord、OLED

**自訂主題**：定義 `:root[data-theme="name"]` CSS 區塊即可（完全 CSS-only，無 code 變更）：
```css
/* 例：自訂主題 */
:root[data-theme="my-custom-theme"] {
  --bg-primary: #1a1a2e;
  --text-primary: #eaeaea;
  --accent: #f39c12;
  /* ... */
}
```

主題 activation：
- `api/routes.py`：`/api/settings` → 儲存到 `settings.json`
- `static/index.html`：server 在 `<html>` 的 `data-theme` 屬性中預設主題（flicker-free）
- 前端：`document.documentElement.setAttribute('data-theme', name)`

---

## 3. Slash Command System（`static/commands.js`）

可在 composer 輸入 `/` 觸發 autocomplete，未知命令直接傳給 agent。

**內建命令**：`/help`, `/clear`, `/compress`, `/compact`（alias），`/model`, `/workspace`, `/new`, `/usage`, `/theme`

**Hermes agent 端**：所有 Hermes agent 的 slash commands 也可在此使用（不需前端知道）。

**Extension point**：⚠️ 未驗證，但 `commands.js` 的 registry 是 plain array，理論上可透過 extension script 注入自訂命令。

---

## 4. Toolset 機制（`api/config.py`）

Agent 可用的 tools 由 hermes-agent 的 `config.yaml` 中的 `platform_toolsets.cli` 控制。

```yaml
# ~/.hermes/config.yaml
platform_toolsets:
  cli:
    - browser
    - clarify
    - code_execution
    - cronjob
    - delegation
    - file
    - image_gen
    - memory
    - session_search
    - skills
    - terminal
    - todo
    - tts
    - vision
    - web
```

**Extension point**：新增 hermes-agent 支援的 toolset 名稱即可啟用對應工具。WebUI 目前對所有 session 使用相同的 full CLI toolset（per-session toolset restriction 在 ROADMAP Wave 4）。

---

## 5. Profile 系統（`api/profiles.py`）

每個 profile 是獨立的 hermes-agent 環境：

```
~/.hermes/
├── config.yaml       # root profile
├── memories/
├── skills/
├── hermes-agent/
└── profiles/
    ├── work/         # 'work' profile
    │   ├── config.yaml
    │   ├── .env      # 獨立 API keys
    │   ├── memories/
    │   └── skills/
    └── personal/
```

**Extension point**：建立新 profile（從 Settings 面板或 `create_profile_api()`），可設定：
- 獨立的 base_url + API key（支援 Ollama、LMStudio 等自訂 endpoint）
- 獨立的 `config.yaml`（不同 toolsets、不同 models）
- 獨立的 memories 和 skills

---

## 6. Gateway Session Watcher（`api/gateway_watcher.py`）

Hermes agent 的 messaging gateway（Telegram/Discord/Slack 等）的 sessions 可即時同步到 WebUI sidebar。

**機制**：
- `start_watcher()` 啟動背景 thread
- 監視 hermes-agent 的 gateway session 資料庫
- 有新 session 或 update 時，透過 SSE 推送到前端（`pending_user_message` event）
- 前端 `sessions.js` 接收後更新 sidebar

**Extension point**：⚠️ 未驗證，但理論上新增 messaging platform 到 hermes-agent 後，WebUI 會自動顯示其 sessions。

---

## 7. Onboarding Wizard（`api/onboarding.py` + `static/onboarding.js`）

首次執行時的引導流程。支援多種 OAuth provider。

**Extension point**：可在 `onboarding.py` 新增 provider 偵測邏輯（支援新的 AI provider）。

---

## 8. Middleware Chain（`server.py:Handler`）

每個 HTTP 請求的 middleware：

```
check_auth()                # 認證
  ↓
get_profile_cookie()        # 解析 profile context
  ↓
set_request_profile()       # 設定 thread-local profile
  ↓
handle_get/post/patch/del() # 路由
  ↓
clear_request_profile()     # 清除（finally）
```

**Extension point**：在 `do_GET()`/`do_POST()` 的 `try/finally` 前後加入邏輯（但需修改 `server.py`）。

---

## 如何新增功能而不動到核心

### 新增 API endpoint（最常見）

1. 在 `api/routes.py` 的 `handle_get()` 或 `handle_post()` 加入新的 `elif` 分支
2. 使用 `require(body, 'field')` 做 validation
3. 使用 `j(handler, {result})` 回傳 JSON
4. 在 `tests/` 新增對應測試

### 新增 sidebar panel

1. 在 `static/index.html` 加入 `.panel-view` div
2. 在 `static/panels.js` 實作 `load*()` 函式
3. 在 `static/boot.js` 的 `switchPanel()` 新增 case
4. 在 `static/style.css` 加入對應 CSS

### 新增主題

只需在 `static/style.css` 加入 `:root[data-theme="name"]` 區塊，並在 Settings 面板的主題選項中新增選項。

### 新增 slash command

在 `static/commands.js` 的 commands array 新增 entry。
