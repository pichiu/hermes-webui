# Stage 2.6 設定與環境

## 設定載入機制

### 優先順序

```
環境變數（最高優先）
    ↓
~/.hermes/config.yaml（hermes-agent 設定）
    ↓
~/.hermes/webui/settings.json（WebUI 使用者設定）
    ↓
api/config.py 中的 hardcoded defaults（最低優先）
```

---

## 完整環境變數清單（`api/config.py`）

| 變數 | 預設值 | 說明 |
|------|--------|------|
| `HERMES_WEBUI_HOST` | `127.0.0.1` | Bind address |
| `HERMES_WEBUI_PORT` | `8787` | HTTP 端口 |
| `HERMES_WEBUI_STATE_DIR` | `~/.hermes/webui` | Session/設定儲存目錄 |
| `HERMES_WEBUI_DEFAULT_WORKSPACE` | `~/workspace` | 新 session 的預設工作目錄 |
| `HERMES_WEBUI_DEFAULT_MODEL` | `openai/gpt-5.4-mini` | 預設模型 |
| `HERMES_WEBUI_PASSWORD` | （未設定） | 設定後啟用密碼認證 |
| `HERMES_WEBUI_AGENT_DIR` | （自動偵測） | hermes-agent checkout 路徑 |
| `HERMES_WEBUI_PYTHON` | （自動偵測） | Python executable |
| `HERMES_WEBUI_BOT_NAME` | `Hermes` | UI 中 assistant 的顯示名稱 |
| `HERMES_WEBUI_EXTENSION_DIR` | （未設定） | Extension 目錄，serve 在 `/extensions/` |
| `HERMES_WEBUI_EXTENSION_SCRIPT_URLS` | （未設定） | Comma-separated script URLs |
| `HERMES_WEBUI_EXTENSION_STYLESHEET_URLS` | （未設定） | Comma-separated stylesheet URLs |
| `HERMES_WEBUI_TLS_CERT` | （未設定） | TLS 憑證路徑 |
| `HERMES_WEBUI_TLS_KEY` | （未設定） | TLS 私鑰路徑 |
| `HERMES_HOME` | `~/.hermes` | Hermes 狀態基目錄 |
| `HERMES_CONFIG_PATH` | `~/.hermes/config.yaml` | Hermes config.yaml 路徑 |
| `HERMES_SKIP_CHMOD` | （未設定） | 設為 1 跳過 credential 權限設定 |

**Per-request（由 `_run_agent_streaming()` 設定，非啟動設定）**：

| 變數 | 說明 |
|------|------|
| `TERMINAL_CWD` | Agent 的工作目錄（= session.workspace） |
| `HERMES_EXEC_ASK` | 設為 `1` 啟用危險命令的 approval gate |
| `HERMES_SESSION_KEY` | Session ID，approval 系統用 |
| `HERMES_HOME` | Active profile 的 home 目錄（每次 run 前設定，之後還原） |

---

## `~/.hermes/webui/settings.json`（使用者設定）

由 `api/routes.py` 的 `/api/settings` 端點讀寫：

```json
{
  "default_model": "anthropic/claude-sonnet-4.6",
  "default_workspace": "/home/user/workspace",
  "send_key": "enter",
  "theme": "dark",
  "password_hash": "...",
  "show_cli_sessions": true,
  "show_token_usage": false,
  "language": "en",
  "bot_name": "Hermes"
}
```

**載入機制**：`api/config.py:load_settings()` 在每次需要時讀取（不快取，確保即時性）。

---

## `~/.hermes/config.yaml`（hermes-agent 設定）

這是 hermes-agent 的設定檔，WebUI 讀取其中與自身相關的部分：

```yaml
platform_toolsets:
  cli:
    - browser
    - clarify
    - code_execution
    - file
    - memory
    - terminal
    - web
    # ...

# Model 設定（hermes-agent 端）
model: anthropic/claude-sonnet-4.6
fallback_model: openai/gpt-5.4-mini

# Provider 設定
providers:
  - name: openai
    api_key: ${OPENAI_API_KEY}
  - name: anthropic
    api_key: ${ANTHROPIC_API_KEY}
```

**載入時機**：`api/config.py` 在 import 時讀取，`api/profiles.py` 在 profile 切換時重新載入。

---

## Profile 設定（`~/.hermes/profiles/<name>/`）

每個 profile 有獨立的設定空間：

```
~/.hermes/profiles/work/
├── config.yaml        # profile 專屬的 model/toolset 設定
├── .env               # profile 專屬的 API keys（0600 權限）
├── memories/
│   ├── MEMORY.md      # agent 的跨 session 記憶
│   └── USER.md        # 使用者資訊
└── skills/            # profile 專屬的 skills
    ├── my_skill/
    │   └── SKILL.md
    └── ...
```

`api/profiles.py:get_profile_runtime_env()` 讀取 profile 的 `.env` 檔，並 merge 進 process 環境。

---

## Feature Flags

沒有正式的 feature flag 系統。但有幾個等效機制：

| 功能 | 啟用方式 |
|------|---------|
| 密碼認證 | `HERMES_WEBUI_PASSWORD` env var 或 Settings 面板 |
| 顯示 CLI sessions | `settings.json:show_cli_sessions`（預設 true） |
| 顯示 token 用量 | `settings.json:show_token_usage`（預設 false）或 `/usage` command |
| TLS | `HERMES_WEBUI_TLS_CERT` + `HERMES_WEBUI_TLS_KEY` 都設定 |
| Extension 注入 | `HERMES_WEBUI_EXTENSION_DIR` 指向存在的目錄 |
| 自動安裝 agent deps | `HERMES_WEBUI_AUTO_INSTALL=1` |

---

## Secrets 管理

- **API Keys**：儲存在 `~/.hermes/config.yaml` 或 profile `.env` 中，不進入 WebUI session data
- **密碼 Hash**：HMAC-based，儲存在 `settings.json`
- **Cookie Token**：HMAC-signed，儲存在 `~/.hermes/webui/.sessions.json`（0600）
- **Signing Key**：隨機生成，儲存在 `~/.hermes/webui/.signing_key`（0600），啟動時自動生成
- **`fix_credential_permissions()`**：啟動時強制設定 `.env`、`.signing_key`、`.sessions.json` 為 0600

Credential redaction：`api/helpers.py:redact_session_data()` 在 API response 中過濾 API keys。

---

## 測試環境隔離

`tests/conftest.py` 設定獨立的測試環境，完全不碰 production 資料：

```python
os.environ['HERMES_WEBUI_PORT'] = '8788'          # 獨立端口
os.environ['HERMES_WEBUI_STATE_DIR'] = '...test'  # 獨立 state 目錄
os.environ['HERMES_HOME'] = '...test-home'         # 獨立 HERMES_HOME
os.environ['HERMES_WEBUI_DEFAULT_WORKSPACE'] = '...test-workspace'
```

測試前清空 state 目錄，測試後刪除。Production port 8787 從不被測試觸及。
