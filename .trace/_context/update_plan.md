# 更新計畫

## 變更摘要
- **Base Commit 範圍**: e23ba59..fcc8328
- **版本**: v0.50.245 → v0.51.2
- **有影響的變更檔案數**: 8
- **變更幅度**: 1.7%（**輕量更新**）
- **變更類型**: 2 個新 feature（Logs Tab、LLM Wiki Status）+ 1 個 CLI session filtering 改進 + 1 個 scroll bug fix

---

## 受影響文件與更新策略

### INDEX.md — 需要更新
- **原因**：
  1. 版本從 v0.50.245 升至 v0.51.2
  2. 新增「Logs」sidebar panel 功能描述
- **影響段落**：一句話總結（版本號）、關鍵指令速查（無需改）
- **更新策略**：Main Agent 局部修改（2 處小改）

### CODEBASE_MAP.md — 需要更新
- **原因**：
  1. 新 panel `panelLogs`（`static/panels.js` + `static/index.html`）
  2. 新端點 `/api/logs`、`/api/wiki/status`
  3. `_context/*.md` 中的路由表已過時
- **影響段落**：「我想改 X 要看哪裡」速查表（新增 Logs panel 條目）
- **更新策略**：Main Agent 局部修改

### API_SURFACE_part2.md / API_SURFACE_part3.md — 需要更新
- **原因**：2 個新端點
  1. `GET /api/logs?file=agent|errors|gateway&tail=200`
  2. `GET /api/wiki/status`
- **影響段落**：需在完整清單中加入，並在適當分組中新增詳細說明
- **更新策略**：Main Agent 局部追加（兩個端點的詳細說明）

### DATA_MODEL.md — 需要小幅更新
- **原因**：Session.compact() 新增 `user_message_count` 欄位
- **影響段落**：Session entity 欄位表
- **更新策略**：Main Agent 局部修改（加一行到 compact() 欄位表）

### ARCHITECTURE.md — 需要更新
- **原因**：
  1. 新增 Logs sidebar panel（extension point）
  2. CLI session filtering 新邏輯（核心邏輯）
  3. LLM Wiki status（新整合點）
- **影響段落**：元件清單（panels.js 新增一個 panel）、Extension points 段落
- **更新策略**：Main Agent 局部追加

### DISCOVERY_LOG.md — 需要更新
- **原因**：追加本次更新的發現
- **更新策略**：Main Agent 追加新段落（標注日期）

### DEV_GUIDE.md — 不需更新
- 無設定系統、測試架構、開發 workflow 的重大變更

### API_SURFACE_part1.md — 不需更新
- 認證、CSRF、SSE、Core chat 無變更

---

## 執行順序

1. `_context/` 相關檔案刷新（api_surface 部分）
2. `API_SURFACE_part3.md` — 加入新端點完整清單
3. `API_SURFACE_part2.md` — 加入新端點說明
4. `DATA_MODEL.md` — 加入 user_message_count
5. `CODEBASE_MAP.md` — 加入 Logs panel 條目
6. `ARCHITECTURE.md` — 加入新元件
7. `INDEX.md` — 版本號 + 新功能
8. `DISCOVERY_LOG.md` — 追加更新紀錄
