# Hermes Agent API Server 参考手册

> 来源：`gateway/platforms/api_server.py` 源码整理
> 日期：2025-06-03

---

## 一、配置项

| 配置 | 环境变量 | config.yaml 路径 | 默认值 | 说明 |
|------|---------|-----------------|--------|------|
| 监听地址 | `API_SERVER_HOST` | `platforms.api_server.host` | `127.0.0.1` | 监听 IP |
| 监听端口 | `API_SERVER_PORT` | `platforms.api_server.port` | `8642` | 监听端口 |
| API 密钥 | `API_SERVER_KEY` | `platforms.api_server.key` | **必填** | Bearer token，不设启动报错 |
| CORS | `API_SERVER_CORS_ORIGINS` | `platforms.api_server.cors_origins` | 无 | 逗号分隔，或 `*` |
| 模型名 | `API_SERVER_MODEL_NAME` | `platforms.api_server.model_name` | `hermes-agent` | `/v1/models` 返回的模型名 |
| 启用 | `API_SERVER_ENABLED=true` | — | 关闭 | 通过环境变量启用 |

⚠️ **启动强制要求设置 `API_SERVER_KEY`**，绑定非 loopback 地址时 key 必须 ≥8 字符。

---

## 二、认证

所有端点（除 `/health*` 外）需要 Header：

```
Authorization: Bearer <API_SERVER_KEY>
```

错误响应（401）：

```json
{
  "error": {
    "message": "Invalid API key",
    "type": "invalid_request_error",
    "code": "invalid_api_key"
  }
}
```

---

## 三、自定义 Header

| Header | 方向 | 说明 |
|--------|------|------|
| `X-Hermes-Session-Id` | 请求/响应 | 继续已有会话，从 `state.db` 加载历史。需认证，最大 256 字符 |
| `X-Hermes-Session-Key` | 请求/响应 | 长期记忆隔离标识（独立于 Session-Id）。需认证，最大 256 字符 |
| `X-Hermes-Completed` | 响应 | `"true"` / `"false"` — Agent 是否正常完成 |
| `X-Hermes-Partial` | 响应 | `"true"` / `"false"` — 响应是否被截断 |
| `X-Hermes-Error` | 响应 | 错误信息（最大 200 字符） |
| `Idempotency-Key` | 请求 | 幂等去重（缓存 300s，最多 1000 条） |

---

## 四、核心端点

### 4.1 健康检查

**`GET /health`** / **`GET /v1/health`**
无需认证。

```json
{"status": "ok", "platform": "hermes-agent"}
```

**`GET /health/detailed`**
无需认证。返回 gateway 状态、活跃 agent 数、PID 等。

---

### 4.2 OpenAI Chat Completions（主力端点）

**`POST /v1/chat/completions`**

请求体：

```json
{
  "model": "hermes-agent",
  "stream": false,
  "messages": [
    {"role": "system", "content": "You are helpful"},
    {"role": "user", "content": "Hello"},
    {"role": "assistant", "content": "Hi!"},
    {"role": "user", "content": "How are you?"}
  ]
}
```

- `messages`（必填）：支持 `system` / `user` / `assistant` 角色
- `content` 支持字符串或多模态数组（text + image_url）
- `stream`：`true` 启用 SSE 流式输出

**Session 续接**：不带 `X-Hermes-Session-Id` 时，按 system_prompt + 首条 user message 哈希生成 session ID；带则从数据库加载历史。

**非流式响应（200）：**

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1700000000,
  "model": "hermes-agent",
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": "I'm doing well!"},
    "finish_reason": "stop"
  }],
  "usage": {
    "prompt_tokens": 100,
    "completion_tokens": 50,
    "total_tokens": 150
  }
}
```

`finish_reason`：`"stop"` | `"length"`（截断）| `"error"`

**流式响应（SSE）：**

Content-Type: `text/event-stream`

```
data: {"id":"...","object":"chat.completion.chunk","choices":[{"delta":{"role":"assistant"},"finish_reason":null}]}

data: {"id":"...","choices":[{"delta":{"content":"token"},"finish_reason":null}]}

event: hermes.tool.progress
data: {"tool":"...","emoji":"...","label":"...","toolCallId":"...","status":"running"}

event: hermes.tool.progress
data: {"tool":"...","toolCallId":"...","status":"completed"}

data: {"choices":[{"delta":{},"finish_reason":"stop"}],"usage":{...}}

data: [DONE]
```

保活心跳：每 30s 发 `: keepalive\n\n`

---

### 4.3 OpenAI Responses API

**`POST /v1/responses`**

请求体：

```json
{
  "model": "hermes-agent",
  "input": "Hello!",
  "instructions": "Be helpful",
  "previous_response_id": "resp_xxx",
  "conversation": "my-conv",
  "store": true,
  "stream": false,
  "conversation_history": [],
  "truncation": "auto"
}
```

响应：

```json
{
  "id": "resp_...",
  "object": "response",
  "status": "completed",
  "model": "hermes-agent",
  "output": [
    {"type": "function_call", "name": "...", "arguments": "{}", "call_id": "..."},
    {"type": "function_call_output", "call_id": "...", "output": [{"type": "input_text", "text": "..."}]},
    {"type": "message", "role": "assistant", "content": [{"type": "output_text", "text": "..."}]}
  ],
  "usage": {"input_tokens": 100, "output_tokens": 50, "total_tokens": 150}
}
```

**`GET /v1/responses/{response_id}`** — 获取已存储的响应
**`DELETE /v1/responses/{response_id}`** — 删除已存储的响应

---

### 4.4 异步 Agent Run（长时间任务）

**`POST /v1/runs`** — 提交异步任务

```json
{
  "input": "Run this task",
  "instructions": "...",
  "session_id": "optional",
  "model": "optional"
}
```

响应（202）：

```json
{
  "run_id": "run_...",
  "session_id": "...",
  "status": "queued",
  "created_at": 1700000000,
  "model": "hermes-agent"
}
```

**`GET /v1/runs/{run_id}`** — 查询状态

`status`：`queued` → `running` → `completed` / `failed` / `waiting_for_approval` / `stopping`

**`GET /v1/runs/{run_id}/events`** — SSE 事件流

```
{"event": "run.started", "run_id": "..."}
{"event": "message.delta", "delta": "token"}
{"event": "tool.started", "tool_name": "...", "preview": "..."}
{"event": "tool.completed", ...}
{"event": "approval.request", "choices": ["once","session","always","deny"], ...}
{"event": "run.completed", "result": {...}, "usage": {...}}
```

**`POST /v1/runs/{run_id}/approval`** — 审批命令

```json
{"choice": "once", "all": false}
// choice: "once" | "session" | "always" | "deny"
```

**`POST /v1/runs/{run_id}/stop`** — 中断运行

---

## 五、Session 管理 API

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/sessions?limit=50&offset=0&source=api_server` | 列出 sessions |
| `POST` | `/api/sessions` | 创建 session `{id?, model?, system_prompt?, title?}` |
| `GET` | `/api/sessions/{id}` | 获取 session 详情 |
| `PATCH` | `/api/sessions/{id}` | 更新 `{title?, end_reason?}` |
| `DELETE` | `/api/sessions/{id}` | 删除 session |
| `GET` | `/api/sessions/{id}/messages` | 获取消息历史 |
| `POST` | `/api/sessions/{id}/fork` | 分叉 session |
| `POST` | `/api/sessions/{id}/chat` | 在 session 中对话 `{message, system_message?}` |
| `POST` | `/api/sessions/{id}/chat/stream` | SSE 流式对话 |

Session Chat 请求：

```json
{
  "message": "Hello!",
  "system_message": "Be helpful"
}
```

Session Chat 响应：

```json
{
  "object": "hermes.session.chat.completion",
  "session_id": "...",
  "message": {"role": "assistant", "content": "..."},
  "usage": {"input_tokens": 100, "output_tokens": 50, "total_tokens": 150}
}
```

---

## 六、Cron Job API

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/jobs?include_disabled=true` | 列出定时任务 |
| `POST` | `/api/jobs` | 创建 `{name, schedule, prompt, deliver, skills?, repeat?}` |
| `GET` | `/api/jobs/{job_id}` | 获取详情 |
| `PATCH` | `/api/jobs/{job_id}` | 更新 `{name?, schedule?, prompt?, enabled?}` |
| `DELETE` | `/api/jobs/{job_id}` | 删除 |
| `POST` | `/api/jobs/{job_id}/pause` | 暂停 |
| `POST` | `/api/jobs/{job_id}/resume` | 恢复 |
| `POST` | `/api/jobs/{job_id}/run` | 手动触发 |

---

## 七、辅助端点

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/v1/models` | 模型列表 |
| `GET` | `/v1/capabilities` | 能力描述 |
| `GET` | `/v1/skills` | 已安装 Skills |
| `GET` | `/v1/toolsets` | 已启用 Toolsets |

---

## 八、常量与限制

| 常量 | 值 |
|------|-----|
| 默认端口 | `8642` |
| 最大请求体 | 10 MB |
| SSE 保活间隔 | 30s |
| 最大归一化文本 | 64 KB |
| 内容列表最大条目 | 1000 |
| 幂等缓存 TTL | 300s |
| 幂等缓存上限 | 1000 条 |
| Session Header 最大长度 | 256 字符 |
| 存储响应上限 | 100 条 |

---

## 九、兼容前端

以下 UI 可直接对接 `http://localhost:8642/v1`，用 `API_SERVER_KEY` 作为 Bearer token：

- **Open WebUI**
- **LobeChat**
- **LibreChat**
- **AnythingLLM**
- **NextChat**
- **ChatBox**

---

## 十、Python 调用示例

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8642/v1",
    api_key="your-api-server-key"
)

# 简单对话
response = client.chat.completions.create(
    model="hermes-agent",
    messages=[{"role": "user", "content": "你好旺财"}]
)
print(response.choices[0].message.content)

# 流式对话
stream = client.chat.completions.create(
    model="hermes-agent",
    messages=[{"role": "user", "content": "写一个快排"}],
    stream=True
)
for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")

# 带 Session 续接（使用 httpx 手动加 Header）
import httpx, json

resp = httpx.post(
    "http://localhost:8642/v1/chat/completions",
    headers={
        "Authorization": "Bearer your-api-server-key",
        "X-Hermes-Session-Id": "my-session-123",
        "Content-Type": "application/json"
    },
    json={
        "model": "hermes-agent",
        "messages": [{"role": "user", "content": "继续上次的讨论"}]
    }
)
print(resp.json())
```
