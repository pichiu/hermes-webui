# Stage 1 Web Findings

## 線上資源摘要

### 官方資源

| 資源 | URL | 重要摘要 |
|------|-----|---------|
| GitHub Repo | https://github.com/nesquena/hermes-webui | 主要開發平台，66 contributors |
| 官方網站 | https://nesquena.github.io/hermes-webui/ | 「Hermes WebUI: The best way to use Hermes Agent」 |
| Hermes Agent 官網 | https://hermes-agent.nousresearch.com/ | 上游 agent 產品頁面 |
| Hermes Agent Docs | https://hermes-agent.nousresearch.com/docs/ | Agent 文件（非 WebUI 文件） |
| GHCR 映像檔 | `ghcr.io/nesquena/hermes-webui:latest` | amd64 + arm64，每次 release 推送 |

### 相關/競品資源

| 資源 | URL | 關係 |
|------|-----|------|
| EKKOLearnAI/hermes-web-ui | https://github.com/EKKOLearnAI/hermes-web-ui | 類似專案（分析/dashboard 取向） |
| outsourc-e/hermes-workspace | https://github.com/outsourc-e/hermes-workspace | 類似專案（native workspace 取向） |
| hermes-agent-docs | https://github.com/mudrii/hermes-agent-docs | 第三方 Hermes agent 文件 |
| Hermes Atlas | https://hermesatlas.com/projects/nesquena/hermes-webui | 社群 tracker |

### 版本資訊

- v0.50.0 是 UI 大改版的里程碑，由 @aronprins 主導（PR #242，26 commits）
- v0.50.245 是撰寫此文件時的 base commit 版本
- v0.50.291 是截至 2026-05-04 的最新 release
- newreleases.io 追蹤頁面：https://newreleases.io/project/github/nesquena/hermes-webui

### Hermes Agent 近況（2026 年 5 月）

根據 releasebot.io 資料，hermes-agent 近期新增：
- 4 個新 inference providers
- 第 18/19 個 messaging 平台（含 Teams plugin）
- Native Spotify + Google Meet 整合
- ComfyUI + TouchDesigner-MCP 從可選轉為內建

這些 agent 端變更可能需要 WebUI 端的對應 API 更新。

### WebUI v0.50.x 主要 feature（v0.50.245）

從 GitHub 摘要：
- **三面板佈局**：sidebar（sessions/nav）、main（chat）、rightpanel（workspace/preview）
- **Composer footer**：model selector、profile chip、workspace chip 整合
- **Hermes Control Center**：側邊欄底部按鈕開啟 tabbed modal（Conversation/Preferences/System）
- **SSE-driven session sidebar**：`pending_user_message` + `active_stream_id` lifecycle tracking
- **Gateway session sync**：Telegram/Discord/Slack sessions 透過 SSE 即時同步到 sidebar
- **Profile 系統**：多 profile 支援，可切換不同 agent 環境（獨立 ~/.hermes/）
- **i18n**：多語言支援（en、de、es、zh、zh-Hant 等）
- **PWA**：service worker、manifest.json（可加入主畫面）
- **Extensions**：`HERMES_WEBUI_EXTENSION_DIR` 可注入自訂 script/stylesheet

### 社群討論發現

- 多個使用者反映 WSL2 + Podman 3.4 有已知限制（`keep-id` 問題），建議升級到 Podman 4+ 或用單容器模式
- Docker 兩容器模式已知限制（#681）：tool 在 WebUI 容器執行，非 agent 容器
- `.env: permission denied` 問題（#1389）：`fix_credential_permissions()` 強制 0600，可設 `HERMES_SKIP_CHMOD=1` 繞過
- CSRF 在 Nginx Proxy Manager 非標準 port 後的問題已在 v0.50.x 修復

### 搜尋關鍵詞

- `hermes-webui nesquena github`
- `hermes agent nousresearch autonomous agent web interface 2026`
- `hermes-webui github releases changelog v0.50 features 2026`
