# Hermes Web UI — 專案總覽與速查

## 一句話總結

Hermes Web UI 是一個**無 build 步驟的輕量瀏覽器介面**，讓你能在瀏覽器或手機上使用 Hermes Agent（NousResearch 出品的自主式 AI agent），提供與 CLI 完全相同的功能：對話、session 管理、cron 排程、skills、memory，以及 workspace 檔案瀏覽。

---

## 技術棧總覽

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Python | 3.11 / 3.12 / 3.13 | 伺服器執行環境 |
| HTTP Server | `http.server.ThreadingHTTPServer` | stdlib | 核心 HTTP 伺服器（無框架） |
| 前端 | Vanilla JavaScript | ES2017+ | 無 build step，直接 `<script>` 載入 |
| CSS | 純 CSS + custom properties | - | 主題系統、mobile responsive |
| Markdown 渲染 | streaming-markdown | 0.2.15 (vendored) | 串流 token 的增量 markdown 渲染 |
| 語法高亮 | Prism.js | CDN + SRI | 程式碼區塊語法高亮 |
| 圖表 | Mermaid.js | CDN + SRI | Mermaid 圖表 inline 渲染 |
| 設定格式 | YAML | pyyaml ≥ 6.0 | 讀取 hermes-agent config.yaml |
| 容器 | Docker (python:3.12-slim) | - | 生產部署 |
| CI/CD | GitHub Actions | - | Multi-arch build + GHCR release |
| 測試 | pytest | - | 3309+ tests，366 test files |
| Agent 核心 | hermes-agent (NousResearch) | 同程序 import | AIAgent、tools、cron、skills |

**唯一直接依賴**：`pyyaml>=6.0`（`requirements.txt`）。所有 AI/ML 依賴住在 hermes-agent venv。

---

## 關鍵指令速查

### 啟動

```bash
# 快速啟動（推薦）
python3 bootstrap.py

# Shell wrapper
./start.sh

# Daemon 模式
./ctl.sh start             # 後台啟動，PID 寫入 ~/.hermes/webui.pid
./ctl.sh status            # PID、uptime、端口、log 路徑
./ctl.sh logs --lines 100  # 即時 log
./ctl.sh restart
./ctl.sh stop

# Docker（單容器）
docker compose up -d       # http://localhost:8787

# 直接啟動
HERMES_WEBUI_PORT=8787 venv/bin/python server.py
```

### 測試

```bash
pytest tests/ -v --timeout=60

# 或用 agent venv
/path/to/hermes-agent/venv/bin/python -m pytest tests/ -v
```

### 常用偵錯

```bash
curl -s http://127.0.0.1:8787/health | python3 -m json.tool
tail -f /tmp/webui-mvp.log
curl -s http://127.0.0.1:8787/api/sessions | python3 -m json.tool
```

### 遠端存取（SSH tunnel）

```bash
ssh -N -L 8787:127.0.0.1:8787 user@your.server.com
# 然後開啟 http://localhost:8787
```

---

## 文件地圖

| 文件 | 位置 | 說明 |
|------|------|------|
| **INDEX.md** | `.trace/INDEX.md` | 本文件：總覽與速查 |
| **ARCHITECTURE.md** | `.trace/ARCHITECTURE.md` | 系統架構、元件、通訊模式、ADR |
| **DATA_MODEL.md** | `.trace/DATA_MODEL.md` | Session 資料模型、儲存結構、生命週期 |
| **API_SURFACE_part1.md** | `.trace/API_SURFACE_part1.md` | 認證、CSRF、Error Handling、SSE 串流協定、Core（健康/Session/Chat） |
| **API_SURFACE_part2.md** | `.trace/API_SURFACE_part2.md` | Core 續、Workspace 檔案操作 |
| **API_SURFACE_part3.md** | `.trace/API_SURFACE_part3.md` | Skills/Cron/Memory、Auth、Onboarding、完整 endpoint 清單 |
| **DEV_GUIDE.md** | `.trace/DEV_GUIDE.md` | 開發者上手指南、測試、偵錯 |
| **CODEBASE_MAP.md** | `.trace/CODEBASE_MAP.md` | 程式碼地圖、模組依賴、「我想改 X」速查 |
| **DISCOVERY_LOG.md** | `.trace/DISCOVERY_LOG.md` | 探索紀錄、落差分析、技術債、待解問題 |

**官方文件**（repo 根目錄）：

| 文件 | 說明 |
|------|------|
| `ARCHITECTURE.md` | 最詳盡的技術文件，含 ADR、sprint log、endpoint reference |
| `README.md` | 快速入門、功能清單、部署指南 |
| `HERMES.md` | Hermes 背景、與競品比較 |
| `TESTING.md` | 測試計畫與自動化說明 |
| `THEMES.md` | 主題系統與自訂指南 |
| `docs/docker.md` | Docker 完整指南 |
| `docs/EXTENSIONS.md` | WebUI 擴充套件指南 |

---

## 專案專屬術語表

| 術語 | 說明 |
|------|------|
| **Session** | 一次對話記錄，含訊息、model、workspace 設定，持久化為 `sessions/{sid}.json` |
| **Workspace** | Agent 的工作目錄（file system 根），右側面板瀏覽的目標 |
| **Profile** | 獨立的 hermes-agent 環境（`~/.hermes/profiles/<name>/`），含獨立 config、memory、skills、API keys |
| **AIAgent** | hermes-agent 的核心 Python class，WebUI 透過 in-process import 呼叫它 |
| **SSE** | Server-Sent Events，WebUI 用來串流 AI tokens 的協定（非 WebSocket） |
| **stream_id** | `POST /api/chat/start` 回傳的 UUID，用來訂閱對應的 SSE 串流 |
| **INFLIGHT** | 前端的 in-flight state 追蹤機制，在 session 切換或頁面刷新時保留進行中的對話 |
| **Hermes Control Center** | sidebar 底部按鈕開啟的 tabbed modal（Conversation / Preferences / System tab） |
| **Composer footer** | 輸入框下方的工具列，含 model selector、profile chip、workspace chip、send button |
| **Tool call card** | chat 中顯示 agent tool 執行過程的可折疊卡片 |
| **Thinking card** | 顯示 Claude extended thinking 或 o3 reasoning 的金色主題卡片 |
| **Approval card** | 危險 shell 命令的授權 UI（allow once / session / always / deny） |
| **CLI bridge** | 讀取 hermes-agent SQLite sessions 並顯示在 WebUI sidebar 的機制（gold 'cli' badge） |
| **Gateway session** | 來自 Telegram/Discord/Slack 等 messaging platform 的 sessions |
| **Toolset** | 一組 hermes-agent tools（如 `terminal`、`web`、`memory`），由 `config.yaml` 控制 |
| **Skills** | hermes-agent 自動生成並儲存的可重用程序（`skills/` 目錄下的 SKILL.md 檔案） |
| **State dir** | `~/.hermes/webui/`，儲存 sessions、settings、projects、workspaces（不在 repo 中） |
| **SRI hash** | Subresource Integrity，固定 CDN 資源版本的安全機制 |
| **QuietHTTPServer** | 繼承自 `ThreadingHTTPServer` 的自訂 class，靜默處理常見的網路中斷錯誤 |
