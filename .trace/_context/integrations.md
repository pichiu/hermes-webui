# Stage 2.5 外部整合

## 核心整合：hermes-agent（NousResearch）

**最關鍵的外部依賴**，整個 WebUI 的存在目的就是為 hermes-agent 提供 UI。

### 整合方式

WebUI 與 hermes-agent 的整合是**同程序 Python import**（非 HTTP call、非 subprocess）：

```python
# api/config.py（import time）
sys.path.insert(0, str(AGENT_DIR))   # AGENT_DIR = ~/.hermes/hermes-agent

# api/streaming.py（lazy import）
from run_agent import AIAgent         # hermes-agent 的主要 class
```

**依賴的 hermes-agent 模組**：

| 模組 | 用途 |
|------|------|
| `run_agent.AIAgent` | 核心 agent class，執行對話 |
| `tools.approval` | 危險命令的授權機制（module-level shared state） |
| `cron.jobs` | Cron job 管理（list, create, run, pause, resume, delete） |
| `tools.skills_tool` | Skills 管理（list, view, save, delete） |
| `memory` module | MEMORY.md + USER.md 讀寫 |

### AIAgent 介面

```python
agent = AIAgent(
    model="anthropic/claude-sonnet-4.6",
    platform='cli',
    quiet_mode=True,
    enabled_toolsets=['file', 'terminal', 'web', ...],
    session_id=session_id,
    stream_delta_callback=on_token,       # (text: str|None) → None
    tool_progress_callback=on_tool,       # (name, preview, args) → None
)
result = agent.run_conversation(
    user_message="...",
    conversation_history=[...],           # OpenAI format
    task_id=session_id                    # 注意：是 task_id 非 session_id（RULE-3）
)
# result: {messages, final_response, completed, ...}
```

### 失敗處理

- `AIAgent` import 失敗（agent 未安裝）：`AIAgent = None`，lazy retry 在 `_get_ai_agent()`
- `auto_install_agent_deps()` 在 `server.py:main()` 嘗試安裝缺失的 agent 依賴
- agent run 拋出例外：streaming 的 `except Exception` → `queue.put('error', {message, trace})` → 前端顯示錯誤訊息

---

## 外部 AI Providers（透過 hermes-agent）

WebUI 本身不直接呼叫 AI API，而是透過 AIAgent：

| Provider | Model ID 格式 | 範例 |
|----------|-------------|------|
| OpenAI | `openai/...` | `openai/gpt-5.4-mini` |
| Anthropic | `anthropic/...` | `anthropic/claude-sonnet-4.6` |
| Google | `google/...` | `google/gemini-2.5-pro` |
| DeepSeek | `deepseek/...` | `deepseek/deepseek-chat-v3-0324` |
| Meta Llama | `meta-llama/...` | `meta-llama/llama-4-scout` |
| Nous Portal | `nousresearch/...` | - |
| OpenRouter | `openrouter/...` | - |
| MiniMax | `minimax/...` | - |
| Z.AI | `zai/...` | - |
| 自訂 | 任意 | 透過 profile base_url |

Provider routing 邏輯在 `api/config.py:resolve_model_provider()`。

---

## Docker Registry（GHCR）

```
ghcr.io/nesquena/hermes-webui:latest
ghcr.io/nesquena/hermes-webui:v0.50.245
```

- GitHub Actions CI（`.github/workflows/`）：每次 tag push 自動 build + push
- 支援 amd64 + arm64（multi-arch）

---

## CDN 資源（帶 SRI Hash）

| 資源 | 用途 | 載入方式 |
|------|------|---------|
| Prism.js | 語法高亮 | CDN, deferred, SRI hash |
| Mermaid.js | Mermaid 圖表渲染 | CDN, deferred, SRI hash |

⚠️ SRI hash 固定了版本。升級需同步更新 `static/index.html` 中的 integrity 屬性。

---

## hermes-agent CLI（`api/profiles.py`）

部分操作透過 subprocess 呼叫 `hermes` CLI：

```python
# api/profiles.py
subprocess.run(['hermes', 'profile', 'switch', name], ...)
subprocess.run(['hermes', 'model', 'list'], ...)
```

主要用於：Profile 建立/切換、model 探索。

---

## Gateway Platforms（透過 hermes-agent）

WebUI 的 sidebar 可顯示來自以下 messaging platform 的 sessions：

| Platform | 整合方式 |
|----------|---------|
| Telegram | hermes-agent gateway → SQLite → gateway_watcher → SSE |
| Discord | 同上 |
| Slack | 同上 |
| WhatsApp | 同上 |
| Signal | 同上（⚠️ 未驗證 WebUI 端支援程度） |
| Email | 同上（⚠️ 未驗證） |
| 其他 10+ | 同上（⚠️ 未驗證） |

**WebUI 端機制**（`api/gateway_watcher.py`）：
```
hermes-agent SQLite session DB
    ↓
gateway_watcher（背景 thread，輪詢）
    ↓（有變更時）
STREAMS 中所有 active SSE connections
    ↓
前端 sidebar 即時更新（gold 'cli' badge）
```

---

## Claude Code Session Import

`api/models.py` 含有 Claude Code session 讀取邏輯：

```python
_iter_claude_code_jsonl_files(projects_dir)
_parse_claude_code_jsonl(path)
```

可從 `~/.claude/projects/` 讀取 Claude Code 的 JSONL session 格式，轉換為 WebUI 可顯示的 session。

---

## Self-Update（`api/updates.py`）

```python
# 向 GitHub API 查詢最新版本
# 比對 WEBUI_VERSION（從 server.py 或 git tag 取得）
# 在前端顯示 update banner（若有新版本）
```

GET `/api/updates/check` → GitHub releases API → `{available, current, latest, url}`

不自動更新，只顯示通知。

---

## TLS/HTTPS（Optional）

```bash
HERMES_WEBUI_TLS_CERT=/path/to/cert.pem
HERMES_WEBUI_TLS_KEY=/path/to/key.pem
```

`server.py:main()` 中：`ssl.SSLContext` wrap `httpd.socket`，`TLSVersion.TLSv1_2` 最低版本。

失敗時 fallback 到 HTTP（不 crash）。

---

## 失敗處理模式總結

| 整合 | 失敗情境 | 處理方式 |
|------|---------|---------|
| hermes-agent import | 未安裝 | `AIAgent = None`，lazy retry |
| hermes-agent import | 依賴缺失 | `auto_install_agent_deps()` → retry |
| agent run 例外 | 任何錯誤 | SSE `error` event → 前端錯誤訊息 |
| CDN 無法載入 | 網路問題 | Prism/Mermaid 不可用，但 UI 仍可用 |
| TLS 設定失敗 | 憑證錯誤 | fallback HTTP，印出警告 |
| gateway_watcher 啟動失敗 | exception | 印出警告，繼續啟動 |
| session recovery 失敗 | exception | 印出警告，繼續啟動 |
| GitHub API 無法連接 | 網路問題 | 靜默忽略，不顯示 update banner |
