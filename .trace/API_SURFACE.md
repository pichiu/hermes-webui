# Hermes Web UI — API 參考文件（API_SURFACE）

> 版本：v0.50.245（May 2026）  
> 來源程式碼：`api/routes.py`（~4100 行）、`api/streaming.py`（~660 行）、`api/auth.py`（~201 行）

---

## 文件結構

本文件分為兩部分：

| 檔案 | 內容 |
|------|------|
| [API_SURFACE_part1.md](API_SURFACE_part1.md) | 認證模式、CSRF 保護、Error Handling、SSE 串流協定（含 Mermaid 流程圖）、Core（健康/Session/Chat/Approval/Settings） |
| [API_SURFACE_part2.md](API_SURFACE_part2.md) | Workspace、Skills/Cron/Memory、Auth Endpoints、Onboarding、完整 Endpoint 清單（~120 個） |

---

## 快速參考摘要

### 認證模式

- **機制**：HMAC cookie（`hermes_session`），30 天 TTL（`api/auth.py:17`）
- **預設關閉**；啟用：`HERMES_WEBUI_PASSWORD` 環境變數 或 Settings 面板
- **PUBLIC_PATHS**（免認證）：`/login`, `/health`, `/favicon.ico`, `/api/auth/login`, `/api/auth/status`, `/manifest.json`
- **CSRF**：所有 POST/PATCH/DELETE 驗證 Origin/Referer，非瀏覽器客戶端（curl）放行（`api/routes.py:647`）

---

### SSE 串流流程（chat）

```
POST /api/chat/start → { stream_id }
GET  /api/chat/stream?stream_id=<id>  （SSE 長連線）
  events: token | reasoning | tool | approval | title | done | stream_end | error | cancel | metering
POST /api/approval/respond  （若需人工核准）
GET  /api/chat/cancel?stream_id=<id>  （若需取消）
```

**SSE Event Types**：

| Event | 說明 |
|-------|------|
| `token` | AI 回覆文字片段（`text: str`） |
| `reasoning` | Chain-of-Thought 文字片段 |
| `tool` | 工具呼叫進度（`tool.started` / `tool.completed`） |
| `approval` | 等待人工核准危險命令 |
| `title` | Session 標題生成完成 |
| `done` | 對話完成（含 `session`, `usage`） |
| `stream_end` | 串流完全結束 |
| `error` | 執行錯誤（`message`, `type`） |
| `cancel` | 使用者取消 |
| `metering` | 即時 token/TPS 計量 |

---

### Error Pattern

所有錯誤回應格式統一為：`{"error": "描述訊息"}`

| Code | 情境 |
|------|------|
| 400 | 請求格式錯誤 |
| 401 | 密碼錯誤 |
| 403 | CSRF 失敗 / Onboarding 限制 |
| 404 | Session/資源不存在 |
| 409 | 衝突（env var 密碼衝突） |
| 413 | 輸入過大 |
| 429 | 登入嘗試次數過多 |
| 500 | 伺服器內部錯誤 |

---

### 核心 Endpoint 一覽

| 類別 | 關鍵 Endpoints |
|------|----------------|
| **Health** | `GET /health` |
| **Session** | `GET /api/sessions`, `GET /api/session`, `POST /api/session/new` |
| **Chat** | `POST /api/chat/start`, `GET /api/chat/stream`, `GET /api/chat/cancel` |
| **Auth** | `GET /api/auth/status`, `POST /api/auth/login`, `POST /api/auth/logout` |
| **Settings** | `GET /api/settings`, `POST /api/settings` |
| **Workspace** | `GET /api/workspaces`, `GET /api/list`, `GET /api/file`, `POST /api/file/save` |
| **Skills** | `GET /api/skills`, `POST /api/skills/save` |
| **Cron** | `GET /api/crons`, `POST /api/crons/create`, `POST /api/crons/run` |
| **Memory** | `GET /api/memory`, `POST /api/memory/write` |
| **Profiles** | `GET /api/profiles`, `POST /api/profile/switch` |
| **Onboarding** | `GET /api/onboarding/status`, `POST /api/onboarding/setup` |
| **Gateway SSE** | `GET /api/sessions/gateway/stream` |
| **Terminal** | `POST /api/terminal/start`, `GET /api/terminal/output` |
| **Updates** | `GET /api/updates/check` |
| **Kanban** | `PATCH /api/kanban/*`, `DELETE /api/kanban/*` |

---

*程式碼來源：`api/routes.py`（handle_get L2033、handle_post L2878、handle_patch L4062、handle_delete L4074）*  
*文件最後更新：2026-05-05*
