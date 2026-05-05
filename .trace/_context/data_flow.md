# Stage 2.2 Request / Data Flow

## 代表性 Use Case：使用者傳送訊息給 AI Agent

完整追蹤：從使用者按 Enter 到 AI 回應渲染完成。

---

### 流程圖

```
使用者按 Enter
      │
      ▼
[static/messages.js] send()
      │ 1. 守衛：S.busy 或無文字？返回
      │ 2. 若 S.session 為 null → POST /api/session/new
      │ 3. activeSid = S.session.session_id（在所有 await 之前捕獲）
      │ 4. 上傳檔案 POST /api/upload（若有 pendingFiles）
      │ 5. push userMsg 到 S.messages，renderMessages()，appendThinking()
      │ 6. setBusy(true)
      │ 7. INFLIGHT[activeSid] = {...}（localStorage 持久化）
      │ 8. startApprovalPolling(activeSid)
      ▼
POST /api/chat/start
      │ [server.py → routes.py handle_post]
      │
      ▼
[api/routes.py] handle_post → chat/start handler
      │ 1. check_auth
      │ 2. read_body() → {session_id, message, model, workspace}
      │ 3. get_session(sid)（SESSIONS cache 或 disk）
      │ 4. 更新 session.model/workspace
      │ 5. queue.Queue() → STREAMS[stream_id]
      │ 6. threading.Thread(target=_run_agent_streaming, daemon=True).start()
      │ 7. 回傳 {stream_id, session_id}（立即返回）
      ▼
GET /api/chat/stream?stream_id=X（長連線 SSE）
      │ [routes.py SSE handler]
      │ 設定 Content-Type: text/event-stream
      │ 輪詢 queue.get(timeout=5s)
      │   - 5s 無事件：送 ": heartbeat\n\n"（保持 proxy/NAT alive）
      │   - 有事件：寫入 "data: {json}\n\n"
      ▼
[api/streaming.py] _run_agent_streaming（背景 daemon thread）
      │ 1. _build_agent_thread_env() → 設定 TERMINAL_CWD, HERMES_EXEC_ASK 等
      │ 2. 鎖定 _ENV_LOCK（全域 os.environ 寫入串列化）
      │ 3. 鎖定 _agent_lock（per-session，防同 session 並發）
      │ 4. _set_thread_env(**env)（thread-local 備份）
      │ 5. os.environ.update(env)（也寫 process-global，fallback）
      │ 6. get_ai_agent() → AIAgent(model, platform='cli', quiet_mode=True, ...)
      │ 7. agent.run_conversation(
      │        user_message=msg_text,
      │        conversation_history=filtered_messages,
      │        task_id=session_id       ← 注意：是 task_id 非 session_id
      │    )
      │ 8. 每個 token → on_token callback → queue.put('token', {text})
      │ 9. 每個 tool → on_tool callback → queue.put('tool', {name, preview})
      │ 10. 若有 pending approval → queue.put('approval', {...})
      │ 11. 完成後 → 更新 session.messages、_maybe_schedule_title_refresh()
      │ 12. queue.put('done', {session: ...})
      │ finally: _clear_thread_env(), os.environ 還原, STREAMS.pop(stream_id)
      ▼
[static/messages.js] EventSource SSE handlers
      │ 'token'    → assistantText += d.text，ensureAssistantRow()，renderMd（streaming-markdown）
      │ 'tool'     → setStatus('Running tool_name...')
      │ 'approval' → showApprovalCard(d)（停止等待使用者授權）
      │ 'title_update' → S.session.title = d.title，更新 sidebar
      │ 'done'     → sync S.messages，renderMessages()，loadDir，renderSessionList
      │              setBusy(false)，delete INFLIGHT[activeSid]
      │              stopApprovalPolling()
      │ 'error'    → showError，setBusy(false)
      └──────────────────────────────────────────────────────────
                    使用者看到完整 AI 回應
```

---

### 詳細層次說明

#### Layer 1：Frontend → Backend（HTTP）

| 步驟 | 端點 | 說明 |
|------|------|------|
| 1 | `POST /api/upload` | 若有附檔，先上傳到 `session.workspace/` |
| 2 | `POST /api/chat/start` | 傳送訊息，取得 `stream_id` |
| 3 | `GET /api/chat/stream?stream_id=X` | 開啟 SSE 長連線 |

#### Layer 2：Routing（`api/routes.py`）

路由是 `if/elif` 鏈（無 routing framework）：

```python
# routes.py handle_post（簡化）
if parsed.path == '/api/upload':        # 必須在 read_body() 前（RULE-2）
    return handle_upload(handler)
body = read_body(handler)
if parsed.path == '/api/chat/start':
    # ... chat 處理
```

#### Layer 3：Validation（`api/helpers.py`）

```python
require(body, 'session_id', 'message')  # 缺少欄位 → 400
s = get_session(sid)                    # KeyError → 404
```

#### Layer 4：Business Logic（`api/streaming.py`）

核心：`_run_agent_streaming()` 在 daemon thread 中執行，透過 `queue.Queue` 與 SSE handler 通訊。

#### Layer 5：Agent 呼叫（`hermes-agent`）

`AIAgent.run_conversation()` 是 hermes-agent 的核心，webui 只傳入：
- `conversation_history`（過濾掉 webui-only metadata 後的 OpenAI 格式訊息）
- `stream_delta_callback`（每個 token 的回呼）
- `tool_progress_callback`（每個 tool invocation 的回呼）

#### Layer 6：Persistence（`api/models.py`）

```python
session.save()  # 原子寫入：先寫 .tmp 再 os.replace()
                # 同時更新 sessions/_index.json
```

---

### Session 切換中途的並發保護

```
send() 開始時：activeSid = S.session.session_id
                       ↓
               ...等待 agent 完成...
                       ↓
agent done → if (S.session.session_id === activeSid)
                 → 更新 UI（正常）
             else
                 → 只刷新 sidebar，不 setBusy(false)（RULE-5）
```

---

### Approval Flow（工具執行授權）

```
agent 呼叫危險工具
      │
      ▼
tools/approval.py → _pending[session_key] = {command, pattern_keys}
      │
      ▼
on_tool callback → 偵測到 pending approval
      │
      ▼
queue.put('approval', {...})
      │
      ▼
前端 showApprovalCard()  ←── 每 1500ms 輪詢 GET /api/approval/pending（fallback）
      │
   使用者點擊按鈕
      │
      ▼
POST /api/approval/respond {choice: 'once'|'session'|'always'|'deny'}
      │
      ▼
approve_session() or approve_permanent() or deny
      │
      ▼
agent is_approved() → True → 繼續執行工具
```

---

### 資料轉換路徑

```
前端 S.messages（含 webui metadata：attachments, timestamp, _ts）
          │
          │ _sanitize_messages_for_api()
          ▼
agent conversation_history（只含 role, content, tool_calls, tool_call_id, name, refusal）
          │
          │ AIAgent 呼叫 LLM API
          ▼
LLM response（streaming tokens）
          │
          │ on_token callback → queue.put
          ▼
SSE stream → 前端 renderMd()（streaming-markdown）→ DOM
          │
          │ agent.run_conversation() 完成後
          ▼
updated messages → session.save() → sessions/{sid}.json
          │
          │ done event
          ▼
前端 S.messages 更新 → renderMessages() → DOM 完整渲染
```
