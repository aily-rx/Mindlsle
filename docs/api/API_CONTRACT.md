# API 契约
## 心屿 · 学生心理健康辅助 Agent（W2 Lab + 完整项目）

> 版本：v1.0  
> 日期：2026-11-07  
> 适用：W2 流式 Lab、W3-W8 完整项目  
> 这是前后端唯一接口事实来源。字段、路径、事件名变更时，先改本文档，再改代码和 Swagger。  
> 默认：前端 Vue3 + JavaScript（无 TypeScript）；后端 Spring Boot 3 + JDK17；JSON + SSE；两人各自独立仓库、各自全栈实现，可交叉 Review，不可直接复制代码。

---

## 0. 两类接口，不要混用

| 类别 | 用途 | 仓库 | 鉴权 | 数据库 |
|---|---|---|---|---|
| W2 Lab | 第一次汇报：流式对话 + 公文要素抽取 | 个人 W2 实验仓 | 无/固定演示即可 | 无 |
| 完整项目 | 心理健康 Agent：登录、会话、RAG、LangGraph、SQL、SSE | 个人综合项目仓 | JWT | MySQL/Redis/PG |

W2 Lab 与完整项目的路径可以相似，但不要在同一份代码里混用；W2 只求 10 分钟干净环境跑通。

---

## 1. 通用约定

### 1.1 基础

- Base Path：`/api`
- 编码：UTF-8；请求/响应：JSON（SSE 除外）
- 时间：ISO-8601 字符串，例如 `2026-11-13T14:00:00+08:00`
- 所有 Long 型 ID 序列化为字符串，避免 JS 精度丢失
- 统一响应（非 SSE）：

```json
{
  "code": 0,
  "message": "ok",
  "data": {},
  "traceId": "b7c1..."
}
```

- `code=0` 表示成功；失败用非 0 业务码，同时配合 HTTP 状态码。
- HTTP 语义：400 参数错误、401 未登录、403 无权限、404 不存在、409 冲突、422 解析失败、429 限流、503 依赖不可用、504 超时。
- 错误响应示例：

```json
{
  "code": "MODEL_TIMEOUT",
  "message": "模型响应超时，请稍后重试",
  "data": null,
  "traceId": "b7c1..."
}
```

### 1.2 认证

- 除 `/api/auth/**`、`/api/health`、Swagger 外，完整项目接口都需要：

```http
Authorization: Bearer <accessToken>
```

- Refresh Token 放在 HttpOnly Cookie，Path=`/api/auth`，前端 JS 不可读。
- Access Token 只放前端内存，不写 localStorage。

### 1.3 分页

- 列表：`?page=1&size=20`，`page` 从 1 开始，`size` 最大 100。
- 消息历史：`?beforeId=<messageId>&size=20`，游标方式，避免 offset 漂移。
- 响应：

```json
{
  "items": [],
  "page": 1,
  "size": 20,
  "total": 57,
  "nextCursor": "1001"
}
```

### 1.4 枚举

| 枚举 | 值 |
|---|---|
| Role | `COUNSELOR`、`ADMIN` |
| MessageRole | `USER`、`ASSISTANT` |
| MessageStatus | `COMPLETED`、`STOPPED`、`FAILED`、`STREAMING` |
| Intent | `RAG`、`WEB`、`SQL`、`DIRECT`、`PRIVACY_REFUSED` |
| SafetyLevel | `NORMAL`、`SENSITIVE`、`CRISIS` |
| CitationType | `DOC`、`WEB`、`SQL` |
| DocumentStatus | `PENDING`、`PARSING`、`EMBEDDING`、`READY`、`FAILED` |
| DocumentCategory | `PSY_EDU`、`COUNSELING_PROCESS`、`CRISIS_GUIDE`、`ASSESSMENT`、`REFERRAL`、`POLICY` |
| Tool | `KNOWLEDGE`、`WEB`、`SQL` |
| TraceStatus | `RUNNING`、`SUCCESS`、`FAILED`、`SKIPPED` |

---

## 2. W2 Lab 接口

> W2 Lab 不接数据库、不做登录；目标是通过 11/13 的硬检查：Vue → Spring Boot → 大模型流式，首字 ≤3s、不截断、Prompt 变量生效。

### 2.1 POST `/api/chat/stream`

**请求**

```json
{
  "message": "用三句话解释什么是 SSE",
  "style": "严谨"
}
```

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| message | string | 是 | 1–4000 字符 |
| style | string | 是 | `严谨` / `温和` / `简洁`，注入后端 Prompt |

**响应**

`Content-Type: text/event-stream`，事件如下：

| event | data | 说明 |
|---|---|---|
| `meta` | `{ "messageId": "m-1" }` | 连接建立 |
| `token` | `{ "delta": "你" }` | 增量文本 |
| `done` | `{ "finishReason": "stop", "usage": { "promptTokens": 10, "completionTokens": 20 }, "firstTokenMs": 820 }` | 正常结束 |
| `error` | `{ "code": "MODEL_TIMEOUT", "message": "模型超时", "traceId": "..." }` | 错误结束 |
| 注释 | `: ping` | 每 15 秒心跳 |

**后端行为**

- 上游 DeepSeek 使用 `stream=true`；后端解析上游 SSE，再转成简化事件。
- 首 token 记录 `firstTokenMs`，写入 `done` 和日志。
- 上游 `[DONE]` 映射为 `done`；上游错误映射为 `error`。
- 客户端断开：停止写入，清理 emitter；能取消上游则取消，不能则记录。
- 响应头：`X-Accel-Buffering: no`、`Cache-Control: no-cache`。
- 超时：60 秒；心跳 15 秒。

**curl 自查**

```bash
curl -N -X POST http://localhost:8080/api/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"message":"用三句话解释什么是SSE","style":"严谨"}'
```

### 2.2 POST `/api/chat`（可选非流式兜底）

请求同上；响应：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "messageId": "m-2",
    "content": "SSE 是……",
    "usage": { "promptTokens": 10, "completionTokens": 20 },
    "firstTokenMs": 800
  },
  "traceId": "..."
}
```

用途：对比流式/非流式、SSE 失败时兜底；不是 W2 硬检查必须。

### 2.3 POST `/api/extract`

**请求**

```json
{
  "text": "关于做好2026年秋季心理健康排查工作的通知\n各学院：\n……\n学生工作处\n2026年11月1日"
}
```

**响应（成功）**

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "docType": "通知",
    "title": "关于做好2026年秋季心理健康排查工作的通知",
    "addressee": "各学院",
    "signature": "学生工作处",
    "date": "2026-11-01",
    "warnings": []
  },
  "traceId": "..."
}
```

**响应（解析失败兜底）**

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "docType": null,
    "title": null,
    "addressee": null,
    "signature": null,
    "date": null,
    "warnings": ["结构化解析失败，请人工核对"]
  },
  "traceId": "..."
}
```

**字段规则**

| 字段 | 类型 | 规则 |
|---|---|---|
| docType | string/null | 枚举：通知/请示/报告/函/纪要/其他 |
| title | string/null | 非空时去首尾空格 |
| addressee | string/null | 可空 |
| signature | string/null | 可空 |
| date | string/null | `YYYY-MM-DD`；无法识别置空并 warning |
| warnings | string[] | 解析/校验问题 |

**实现要求**

- DeepSeek JSON mode：`response_format={"type":"json_object"}`；Prompt 必须包含 “json”；`temperature=0`。
- 后端用 Jackson 解析，不允许直接把模型原文当 JSON 返回。
- 解析失败不返回 500；返回空字段 + warnings，并记录脱敏后的原始响应摘要与 traceId。
- 输入超过 8000 字符返回 400；空输入返回 400。

### 2.4 GET `/api/health`

```json
{ "status": "UP", "deepseek": "UP" }
```

W2 只检查应用与模型配置；不要把 Key 返回。

---

## 3. 完整项目：Auth

### 3.1 POST `/api/auth/register`

```json
{ "username": "counselor01", "password": "Demo@12345", "displayName": "王老师" }
```

- 用户名 4–32 位，唯一；密码至少 8 位；演示环境默认角色 `COUNSELOR`。
- 响应：

```json
{
  "code": 0,
  "message": "ok",
  "data": { "id": "1", "username": "counselor01", "displayName": "王老师", "role": "COUNSELOR" },
  "traceId": "..."
}
```

### 3.2 POST `/api/auth/login`

```json
{ "username": "counselor01", "password": "Demo@12345" }
```

响应：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "accessToken": "eyJ...",
    "tokenType": "Bearer",
    "expiresIn": 1800,
    "user": { "id": "1", "username": "counselor01", "displayName": "王老师", "role": "COUNSELOR" }
  },
  "traceId": "..."
}
```

同时 `Set-Cookie: refreshToken=...; HttpOnly; Path=/api/auth; SameSite=Lax`。

### 3.3 POST `/api/auth/refresh`

- 请求带 Cookie；无 body。
- 旋转 Refresh Token；响应同登录（新 accessToken + 新 Cookie）。
- 旧 Refresh 立即失效；失败 401。

### 3.4 POST `/api/auth/logout`

- 撤销 Refresh；清 Cookie；返回 204。

### 3.5 GET `/api/auth/me`

```json
{
  "code": 0,
  "message": "ok",
  "data": { "id": "1", "username": "counselor01", "displayName": "王老师", "role": "COUNSELOR" },
  "traceId": "..."
}
```

---

## 4. 完整项目：Conversations

### 4.1 GET `/api/conversations?page=1&size=20`

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "items": [
      { "id": "10", "title": "考试焦虑怎么调节", "lastMessageAt": "2026-11-13T14:00:00+08:00", "createdAt": "2026-11-13T13:50:00+08:00" }
    ],
    "page": 1,
    "size": 20,
    "total": 1
  },
  "traceId": "..."
}
```

### 4.2 POST `/api/conversations`

```json
{ "title": "新对话" }
```

`title` 可省略，默认“新对话”；首条用户消息后由后端自动更新标题。

### 4.3 GET `/api/conversations/{id}/messages?beforeId=&size=20`

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "items": [
      {
        "id": "102",
        "role": "ASSISTANT",
        "content": "……",
        "status": "COMPLETED",
        "intent": "RAG",
        "safetyLevel": "NORMAL",
        "citations": [],
        "trace": [],
        "usage": { "promptTokens": 1200, "completionTokens": 300 },
        "createdAt": "2026-11-13T14:00:05+08:00"
      }
    ],
    "nextCursor": "102"
  },
  "traceId": "..."
}
```

### 4.4 PATCH `/api/conversations/{id}`

```json
{ "title": "新的标题" }
```

### 4.5 DELETE `/api/conversations/{id}`

- 级联删除消息；204。
- 不属于当前用户统一 404。

---

## 5. 完整项目：Chat

### 5.1 POST `/api/conversations/{id}/messages`（非流式）

```json
{
  "content": "本月各年级高风险学生人数是多少？",
  "tools": ["KNOWLEDGE", "WEB", "SQL"],
  "clientMessageId": "c-001"
}
```

- `tools` 可省略，默认全部开启；服务端仍按角色/配置裁剪。
- `clientMessageId` 可选，用于前端重试幂等。

响应：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "userMessage": { "id": "101", "role": "USER", "content": "……", "createdAt": "..." },
    "assistantMessage": {
      "id": "102",
      "role": "ASSISTANT",
      "content": "……",
      "status": "COMPLETED",
      "intent": "SQL",
      "safetyLevel": "NORMAL",
      "citations": [],
      "trace": [],
      "usage": { "promptTokens": 1200, "completionTokens": 300 },
      "createdAt": "..."
    }
  },
  "traceId": "..."
}
```

### 5.2 POST `/api/conversations/{id}/messages/stream`（SSE）

请求同上。事件：

| event | data | 说明 |
|---|---|---|
| `meta` | `{ "messageId": "102", "conversationId": "10" }` | 占位消息已创建 |
| `safety` | `{ "level": "CRISIS", "resources": [...] }` | 危机/敏感时下发 |
| `node` | `{ "node": "route", "label": "意图路由", "status": "SUCCESS", "durationMs": 320, "summary": "intent=SQL" }` | 节点状态 |
| `token` | `{ "delta": "本" }` | 增量文本 |
| `citation` | `{ "citations": [...] }` | 引用列表 |
| `done` | `{ "messageId": "102", "status": "COMPLETED", "usage": {...}, "trace": [...] }` | 正常结束 |
| `error` | `{ "code": "...", "message": "...", "traceId": "..." }` | 错误结束 |
| 注释 | `: ping` | 心跳 |

规则：

- 先鉴权，再建 SSE；异步线程不依赖 SecurityContext，userId 在进入异步前取出。
- 助手消息先落库 `STREAMING`；完成更新 `COMPLETED`；客户端停止/断开保存部分内容并标 `STOPPED`；异常标 `FAILED`。
- 非流式接口保留，作为 SSE 失败兜底。
- 业务数据只通过工具查询，不新增公开 REST 接口。

---

## 6. 完整项目：Knowledge（共享资料库）

### 6.1 POST `/api/knowledge/documents`

`multipart/form-data`：

- `file`：PDF/DOCX/MD/TXT，≤20MB
- `category`：`PSY_EDU` / `COUNSELING_PROCESS` / `CRISIS_GUIDE` / `ASSESSMENT` / `REFERRAL` / `POLICY`

响应：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "id": "12",
    "originalName": "考试焦虑自助指南.pdf",
    "category": "PSY_EDU",
    "status": "PENDING",
    "chunkCount": 0,
    "createdAt": "..."
  },
  "traceId": "..."
}
```

后端异步解析/切片/向量化；前端轮询详情或列表。

### 6.2 GET `/api/knowledge/documents?page=1&size=20&status=&category=`

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "items": [
      {
        "id": "12",
        "originalName": "考试焦虑自助指南.pdf",
        "category": "PSY_EDU",
        "fileSize": 102400,
        "status": "READY",
        "chunkCount": 36,
        "errorMessage": null,
        "uploadedBy": "counselor01",
        "createdAt": "..."
      }
    ],
    "page": 1,
    "size": 20,
    "total": 1
  },
  "traceId": "..."
}
```

### 6.3 GET `/api/knowledge/documents/{id}`

返回同上单条；不存在/无权 404。

### 6.4 POST `/api/knowledge/documents/{id}/reindex`

- 删除旧向量后重建；返回 202 + 文档状态 `PENDING`。
- 同一文档重复触发应幂等/串行，避免并发重建。

### 6.5 DELETE `/api/knowledge/documents/{id}`

- 删除记录、向量、文件；204。
- 上传人/管理员可删；一期可简化为所有 `COUNSELOR` 可删，但保留 `uploadedBy` 审计。

### 6.6 GET `/api/knowledge/stats`

```json
{
  "code": 0,
  "message": "ok",
  "data": { "documentCount": 12, "chunkCount": 480, "readyCount": 10, "failedCount": 1 },
  "traceId": "..."
}
```

**安全约束**

- 共享知识库只放公开/校内可共享的科普、流程、量表说明；禁止学生个案、咨询记录、姓名/学号/联系方式。
- 文档删除必须同时删向量，避免“删了文件还能检索到”。
- SQL 工具不查知识库表；知识库不存学生业务数据。

---

## 7. 共享数据结构

### 7.1 Citation

```json
{
  "type": "DOC",
  "documentId": "12",
  "fileName": "考试焦虑自助指南.pdf",
  "pageNumber": 3,
  "chunkIndex": 2,
  "snippet": "……"
}
```

```json
{ "type": "WEB", "title": "……", "url": "https://example.com", "snippet": "……" }
```

```json
{ "type": "SQL", "views": ["v_risk_overview"], "rowCount": 4 }
```

- `url` 只允许 http/https；前端必须清洗。
- `snippet` 截断 200–300 字；不要放学生 PII。

### 7.2 Trace

```json
{
  "node": "sql_query",
  "label": "业务数据查询",
  "status": "SUCCESS",
  "durationMs": 820,
  "summary": "使用 v_risk_overview，返回 4 行",
  "detail": { "views": ["v_risk_overview"], "rowCount": 4 }
}
```

- `detail` 不放大段学生记录、不放大模型原始 Prompt。
- 前端按 `node` upsert；最终以 `done.trace` 为准。

### 7.3 Safety

```json
{
  "level": "CRISIS",
  "resources": [
    { "name": "全国心理援助热线", "phone": "12356", "note": "7×24，以当地实际开通为准" },
    { "name": "急救", "phone": "120" },
    { "name": "报警", "phone": "110" },
    { "name": "学校心理中心", "phone": "以学校公布为准" }
  ]
}
```

- 非流式响应没有 `safety` 事件时，前端用常量资源表按 `safetyLevel` 渲染。
- 危机响应不查知识库/SQL/联网，不诊断、不提供危险方法细节。

### 7.4 Usage

```json
{ "promptTokens": 1200, "completionTokens": 300 }
```

DeepSeek 流式若未返回 usage，允许为 `null`；不要伪造。

---

## 8. 错误码

| code | HTTP | 场景 |
|---|---|---|
| `VALIDATION_ERROR` | 400 | 参数/长度/枚举错误 |
| `AUTH_INVALID` | 401 | 未登录/Token 无效 |
| `FORBIDDEN` | 403 | 角色无权 |
| `NOT_FOUND` | 404 | 资源不存在或不属于当前用户 |
| `DUPLICATE` | 409 | 用户名/文件 hash 重复 |
| `PARSE_FAILED` | 422 | 文档解析失败 |
| `RATE_LIMITED` | 429 | 应用限流 |
| `MODEL_RATE_LIMITED` | 429 | 模型限流 |
| `MODEL_TIMEOUT` | 504 | 模型超时 |
| `MODEL_ERROR` | 502 | 模型返回异常 |
| `EMBEDDING_UNAVAILABLE` | 503 | Ollama 不可用 |
| `SEARCH_UNAVAILABLE` | 503 | 搜索源不可用 |
| `PRIVACY_REFUSED` | 200 | 隐私拒绝；`intent=PRIVACY_REFUSED` |
| `CRISIS_PROTOCOL` | 200 | 已进入危机响应 |
| `INTERNAL_ERROR` | 500 | 未分类异常（不暴露堆栈） |

---

## 9. 契约变更流程

1. 先改 `docs/api/API_CONTRACT.md`。
2. 同步改 Swagger 注解与示例。
3. 在 PR 描述里写明“接口变更”。
4. 通知对方；前端 API 层一起改。
5. 不允许只改代码不改文档；不允许静默改字段名。
