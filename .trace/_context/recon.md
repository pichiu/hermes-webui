# Stage 1 Reconnaissance

## 專案概覽

**名稱**: Hermes Web UI  
**Repo**: https://github.com/nesquena/hermes-webui  
**目的**: 為 Hermes Agent（NousResearch 出品的自主式 AI agent）提供瀏覽器端介面，功能與 CLI 完全對等。  
**版本**: v0.50.245（April 30, 2026）；GitHub 最新 v0.50.291（May 4, 2026）  
**License**: MIT  

---

## 技術棧

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Python | 3.11-3.13 | 伺服器執行環境 |
| HTTP Server | `http.server.ThreadingHTTPServer` | stdlib | 核心 HTTP 伺服器（無框架） |
| 前端 | Vanilla JavaScript | ES2017+ | 無 build step，無 bundler，無框架 |
| 設定格式 | YAML | pyyaml ≥ 6.0 | 讀取 Hermes config.yaml |
| 語法高亮 | Prism.js | CDN, SRI hash | 程式碼區塊 |
| 圖表渲染 | Mermaid.js | CDN, SRI hash | Mermaid 圖表 in chat |
| 流式渲染 | streaming-markdown@0.2.15 | vendored (static/vendor/) | 增量 SSE token 的 markdown 渲染 |
| 容器 | Docker (python:3.12-slim) | - | CI + 生產部署 |
| CI/CD | GitHub Actions | - | Multi-arch Docker build + GitHub Release on tag |
| 測試 | pytest | - | 3309 tests，366 test files |
| 核心依賴 | hermes-agent（NousResearch） | - | AIAgent class，tools，cron，skills，memory |

**完整 requirements.txt**: 只有 `pyyaml>=6.0`。所有 AI/agent 依賴住在 hermes-agent 的 venv。

---

## 目錄結構（3 層）

```
hermes-webui/
├── server.py              # 入口點：薄 routing shell，~154 行純 Python
├── bootstrap.py           # 一鍵啟動：偵測 agent、安裝依賴、開瀏覽器
├── start.sh               # bootstrap.py 的 shell wrapper
├── ctl.sh                 # 後台 daemon 管理（start/stop/status/logs/restart）
├── requirements.txt       # pyyaml>=6.0（唯一直接依賴）
├── Dockerfile             # python:3.12-slim，~23 行
├── docker-compose.yml     # 單容器（最常用）
├── docker-compose.two-container.yml    # Agent + WebUI 分容器
├── docker-compose.three-container.yml  # Agent + Dashboard + WebUI
├── .env.example           # 環境變數範本
├── api/                   # 後端業務邏輯模組
│   ├── config.py          # 設定載入、全域狀態、model 偵測（~1110 行）
│   ├── routes.py          # 所有 GET/POST/PATCH/DELETE handlers（~2250 行）
│   ├── streaming.py       # SSE engine、run_agent、cancel（~660 行）
│   ├── models.py          # Session model + CRUD + CLI bridge（~377 行）
│   ├── auth.py            # 可選密碼認證、HMAC cookie（~201 行）
│   ├── helpers.py         # HTTP 工具函式、安全標頭（~175 行）
│   ├── profiles.py        # Profile 狀態管理、hermes_cli wrapper（~411 行）
│   ├── onboarding.py      # 首次執行精靈、OAuth provider（~507 行）
│   ├── workspace.py       # 檔案操作、workspace helpers、git 偵測（~288 行）
│   ├── upload.py          # Multipart parser、檔案上傳（~82 行）
│   ├── updates.py         # 自我更新檢查、release notes（~257 行）
│   ├── state_sync.py      # /insights sync - message_count 到 state.db（~113 行）
│   ├── providers.py       # Provider 管理 UI 後端
│   ├── agent_sessions.py  # CLI bridge：讀取 hermes-agent SQLite sessions
│   ├── background.py      # 背景任務
│   ├── gateway_watcher.py # Gateway session 即時監控（SSE push）
│   ├── metering.py        # Token/cost metering
│   ├── extensions.py      # WebUI 擴充套件注入
│   ├── session_recovery.py # 啟動時 .bak 還原（#1558）
│   ├── session_ops.py     # Session 操作輔助
│   └── ... (其他 api/ 模組)
├── static/                # 前端靜態檔案（直接從磁碟服務）
│   ├── index.html         # HTML 模板（~600 行）
│   ├── style.css          # 全部 CSS，含 mobile responsive 與主題（~1050 行）
│   ├── ui.js              # DOM helpers、renderMd、tool cards、context indicator（~1740 行）
│   ├── sessions.js        # Session CRUD、search、sidebar 渲染（~800 行）
│   ├── messages.js        # send()、SSE handlers、串流、session recovery（~655 行）
│   ├── panels.js          # Cron/Skills/Memory/Profiles/Settings（~1438 行）
│   ├── workspace.js       # 檔案預覽、file ops、git badge（~286 行）
│   ├── commands.js        # Slash command autocomplete（~267 行）
│   ├── boot.js            # 行動導覽、voice input、boot IIFE（~524 行）
│   ├── i18n.js            # 國際化（多語言支援）
│   ├── icons.js           # SVG icons
│   ├── onboarding.js      # 首次執行精靈前端
│   ├── login.js           # 登入頁面
│   ├── terminal.js        # 終端機介面
│   ├── sw.js              # Service Worker（PWA）
│   └── vendor/            # streaming-markdown@0.2.15（vendored）
├── tests/                 # 366 個測試檔案，3309 tests
│   ├── conftest.py        # 隔離測試伺服器（port 8788）
│   └── test_*.py          # 每個 sprint / feature 一個檔案
├── docs/
│   ├── docker.md          # Docker 完整指南
│   ├── EXTENSIONS.md      # WebUI 擴充套件指南
│   ├── ISSUES.md          # 已知問題
│   ├── wsl-autostart.md   # WSL 自動啟動
│   ├── supervisor.md      # Supervisor 整合
│   └── images/            # UI 截圖
├── scripts/
│   ├── windows/           # Windows 腳本
│   └── wsl/               # WSL 腳本
└── .github/workflows/     # CI：multi-arch Docker build + GitHub Release
```

**狀態目錄**（執行期，不在 repo 中）：
```
~/.hermes/webui/
├── sessions/              # 每個 session 一個 JSON 檔
│   └── _index.json        # Session 索引（O(1) 讀取）
├── workspaces.json        # 已登錄的 workspace 清單
├── settings.json          # 使用者設定（model、send key、password hash 等）
├── projects.json          # Session 群組
├── last_workspace.txt     # 最近使用的 workspace
└── .sessions.json         # Auth session tokens（HMAC）
```

---

## 架構模式

**Monolith**：單一 Python 程序，ThreadingHTTPServer，每個請求一個執行緒。

特點：
- 無 web framework（Django/Flask/FastAPI 都沒有）
- 後端 Python 直接呼叫 `hermes-agent` 的 AIAgent class（同程序 import）
- 前端無 build step（React/Vue/Angular 都沒有），直接 `<script>` tag 載入 JS modules
- SSE 串流（非 WebSocket）

---

## 既有文件

| 文件 | 路徑 | 內容 |
|------|------|------|
| ARCHITECTURE.md | 根目錄 | **最詳細**的技術文件，含所有 API endpoints、ADR、sprint log |
| README.md | 根目錄 | 快速入門、功能清單、部署指南 |
| HERMES.md | 根目錄 | Hermes agent 的背景、與競品比較 |
| DESIGN.md | 根目錄 | UI/UX 設計規範 |
| ROADMAP.md | 根目錄 | 功能 roadmap |
| SPRINTS.md | 根目錄 | Sprint 計畫（CLI + Claude parity 目標） |
| TESTING.md | 根目錄 | 測試計畫與自動化測試說明 |
| THEMES.md | 根目錄 | 主題系統說明、自訂主題指南 |
| CHANGELOG.md | 根目錄 | 每個 sprint 的 release notes |
| BUGS.md | 根目錄 | Bug backlog 與修復紀錄 |
| CONTRIBUTING.md | 根目錄 | 貢獻指南 |
| docs/docker.md | docs/ | Docker 完整指南（含常見錯誤） |
| docs/EXTENSIONS.md | docs/ | WebUI 擴充套件指南 |

**文件品質評估**：ARCHITECTURE.md 極度詳盡，內嵌了完整的 ADR、sprint log、endpoint reference、critical rules。是主要的技術參考來源。README.md 對使用者友好但已部分過時（仍提及舊版檔案行數）。

---

## 文件與程式碼落差

| 文件所述 | 程式碼實際 | 位置 |
|----------|-----------|------|
| ARCHITECTURE.md 說 `streaming.py` ~236 行 | 實際行數更多（含 STREAM_PARTIAL_TEXT 等新功能） | `api/streaming.py` |
| ARCHITECTURE.md 說 server.py ~81 行 | 實際更長（含 TLS、FD limit、recovery） | `server.py` |
| ARCHITECTURE.md 說 tests/ 有 21 test files | 實際有 366 個 test files | `tests/` |
| README 說 3309 tests | 最新版本已更多（v0.50.245 後仍在增加） | `tests/` |
| ARCHITECTURE.md 說 `api/` 模組較少 | 實際 `api/` 有更多模組（kanban_bridge, clarify, metering 等） | `api/` |
| ARCHITECTURE.md 說 auth session TTL 30天 | `auth.py` 中 `SESSION_TTL = 86400 * 30`（✓ 一致） | `api/auth.py:17` |

---

## 規模統計

- 總檔案數：484
- Test 檔案：366
- 程式碼行數估算：前端 ~8000+ 行 JS；後端 ~6000+ 行 Python
- Contributors：66 人（截至 v0.50.245）
