# 后端开发指南
## 心屿 · 学生心理健康辅助 Agent（W2 Lab + 完整项目）

> 版本：v1.0  
> 日期：2026-11-07  
> 适用：W2 流式 Lab、W3-W8 完整项目  
> 默认：JDK17、Spring Boot 3.x、Maven Wrapper、MyBatis-Plus、LangChain4j/LangGraph4j、MySQL/Redis/PG+pgvector、Ollama bge-m3、DeepSeek（W6 后切 vLLM）。  
> 重要：培养计划要求每个人都能独立跑通前端、后端和 Agent。本文档供你**自己完整实现**，搭档可以 Review，但不能一人只做后端、一人只做前端。

---

## 0. 先分清两个后端

| | W2 Lab | 完整项目 |
|---|---|---|
| 目标 | 通过 11/13 第一次汇报硬检查 | W7-W8 综合项目交付 |
| 仓库 | 个人 `w2-stream-lab` | 个人综合项目仓 |
| 数据库 | 无 | MySQL + Redis + PG |
| 核心 | SSE 流式 + Prompt 变量 + 公文抽取 | JWT、会话、RAG、LangGraph、SQL、SSE |
| 依赖 | DeepSeek API | DeepSeek→vLLM、Ollama bge-m3、Dify 独立线 |
| 不要做 | MySQL/RAG/LangGraph/JWT | 不要拖到 W2 |

W2 先把流式硬检查跑通；完整项目从 W3 的 RAG 开始，不要提前把 MySQL/RAG 接进 W2 Lab。

---

# 第一部分：W2 Lab 后端

## 1. 目标与验收

- Vue 页面 → Spring Boot `/api/chat/stream` → DeepSeek 流式 → 前端逐字显示。
- 首字 ≤3 秒；长输出不截断；客户端断开/模型异常有明确事件。
- 现场改 Prompt 变量（`style`）输出变化可解释。
- 公文要素抽取接口：文种/标题/主送/落款/日期；JSON 校验 + 失败兜底。
- README 从零可复现；无密钥入库。

## 2. 仓库结构

```text
w2-stream-lab/
├─ backend/
│  ├─ pom.xml
│  ├─ mvnw / mvnw.cmd
│  └─ src/main/
│     ├─ java/com/example/w2/
│     │  ├─ W2Application.java
│     │  ├─ common/
│     │  │  ├─ ApiResponse.java
│     │  │  ├─ ErrorCode.java
│     │  │  ├─ BusinessException.java
│     │  │  ├─ GlobalExceptionHandler.java
│     │  │  └─ TraceIdFilter.java
│     │  ├─ config/
│     │  │  ├─ LlmProperties.java
│     │  │  ├─ LlmConfig.java
│     │  │  ├─ AsyncConfig.java
│     │  │  └─ WebConfig.java
│     │  ├─ chat/
│     │  │  ├─ ChatController.java
│     │  │  ├─ ChatService.java
│     │  │  ├─ ChatRequest.java
│     │  │  ├─ ChatStyle.java
│     │  │  └─ SseTokenSink.java
│     │  ├─ extract/
│     │  │  ├─ ExtractController.java
│     │  │  ├─ ExtractService.java
│     │  │  ├─ ExtractRequest.java
│     │  │  ├─ ExtractResult.java
│     │  │  └─ ExtractValidator.java
│     │  └─ llm/
│     │     ├─ PromptLoader.java
│     │     └─ LlmClient.java
│     └─ resources/
│        ├─ application.yml
│        ├─ prompts/chat-system.md
│        └─ prompts/extract.md
├─ frontend/
├─ docs/demo-script.md
├─ docs/failure-case.md
├─ .env.example
└─ .gitignore
```

## 3. 依赖与配置

### 3.1 Maven 依赖（最小集）

- `spring-boot-starter-web`
- `spring-boot-starter-validation`
- `spring-boot-starter-actuator`
- `langchain4j` + `langchain4j-open-ai`（推荐，和完整项目一致）
- Jackson（Spring Boot 自带）
- Lombok 可选；不用也行

> 如果你在 LangChain4j 版本/依赖上卡超过 1 小时，W2 可以改用 Spring `WebClient` 直接调 DeepSeek `/chat/completions`，但完整项目仍建议回到 LangChain4j。

### 3.2 环境变量

```dotenv
SERVER_PORT=8080
DEEPSEEK_BASE_URL=https://api.deepseek.com
DEEPSEEK_API_KEY=sk-...
DEEPSEEK_CHAT_MODEL=deepseek-flash
DEEPSEEK_TIMEOUT_SECONDS=60
```

- `.env` 不提交；`.env.example` 提交。
- Spring Boot 可通过 `spring.config.import=optional:file:./.env[.properties]` 读取，或由 IDE/Run Configuration 注入。
- 不要在前端放 Key；不要硬编码。

### 3.3 `application.yml` 要点

- 端口、日志级别、Actuator 只暴露 health。
- 自定义 `app.llm.*` 绑定 `LlmProperties`。
- SSE 超时、心跳间隔、线程池大小可配置。
- `server.tomcat.max-http-form-post-size` 可不改；W2 无文件上传。

## 4. 核心类设计

### 4.1 `LlmClient`

职责：封装 DeepSeek 调用，隔离 LangChain4j。

```text
LlmClient
├─ stream(ChatCommand, TokenSink)
├─ complete(ChatCommand) -> String
└─ completeJson(ChatCommand, Class<T>) -> T
```

- `ChatCommand`：systemPrompt、userMessage、temperature、maxTokens、jsonMode。
- 流式：`StreamingChatModel.chat(request, handler)`；handler 的 `onPartialResponse` 调 `sink.onToken`，`onCompleteResponse` 调 `sink.onComplete`，`onError` 调 `sink.onError`。
- 非流式/JSON：`ChatModel.chat(...)`；JSON mode 按所用 LangChain4j 版本设置（response format / JSON schema）。
- 超时、429、5xx：映射为 `MODEL_TIMEOUT` / `MODEL_RATE_LIMITED` / `MODEL_ERROR`。
- 记录 usage；流式没有 usage 时允许 null。
- **不要**在 `LlmClient` 里写 SseEmitter；传输层由 `SseTokenSink` 负责。

### 4.2 `TokenSink`

```text
interface TokenSink {
  void onStart();
  void onToken(String delta);
  void onComplete(Usage usage);
  void onError(Throwable error);
}
```

W2 实现 `SseTokenSink`：

- `onStart`：`emitter.send(event("meta", ...))`。
- `onToken`：首次 token 记录 `firstTokenMs`；发送 `token`。
- `onComplete`：发送 `done`（含 usage、firstTokenMs），`emitter.complete()`。
- `onError`：发送 `error`，`emitter.completeWithError` 或 `complete`，避免二次发送。
- `AtomicBoolean closed` 防重复完成。
- 客户端断开时写入会抛 IOException；捕获后标记 closed，停止后续写入。

### 4.3 `ChatService`

职责：

1. 校验 `message` 长度 1–4000、`style` 枚举。
2. `PromptLoader.load("chat-system")`，替换 `${style}`。
3. 创建 `SseEmitter`（超时 60s）。
4. 注册 `onCompletion/onTimeout/onError` 清理。
5. 启动心跳（每 15s 发注释 `: ping`）。
6. 调用 `LlmClient.stream(...)`。
7. 返回 emitter。

注意：

- Controller 返回 `SseEmitter` 后请求线程结束；模型回调在别的线程，不能依赖 SecurityContext/RequestContextHolder。
- 心跳任务用 `ScheduledExecutorService`；完成时取消。
- 发送事件统一用 Jackson 序列化，避免手拼 JSON。
- 响应头 `X-Accel-Buffering: no`。

### 4.4 `SseEmitter` 生命周期

| 回调 | 处理 |
|---|---|
| `onCompletion` | 标记 closed；取消心跳；记录 totalMs |
| `onTimeout` | 尝试发 `error`（若还能写）；complete；记录 |
| `onError` | 同上；不要重复 complete |
| 客户端断开 | 写入 IOException；停止写入；记录“client disconnected” |

- 不要对同一个 emitter 多次 `complete()`。
- 错误事件发送失败时只记日志，不再抛给全局异常处理器。
- W2 不做重连/断点续传；前端重试即可。

### 4.5 上游 SSE 解析（若用 WebClient 裸调）

如果不用 LangChain4j，直接调 DeepSeek：

- `POST /chat/completions`，body 加 `"stream": true`。
- 响应是 `text/event-stream`，每行 `data: {...}`，最后 `data: [DONE]`。
- 取 `choices[0].delta.content`；`finish_reason` 出现时结束。
- 上游错误可能以 JSON 或 event 返回；不要让原始错误透传。
- 用缓冲区处理半行/多行；不要假设一次网络包就是一条完整事件。
- 后端再按第 2 节的事件协议发给前端，前端不接触 DeepSeek 原始格式。

## 5. Prompt 变量

`chat-system.md` 示例结构：

```text
你是公文写作辅助助手。
回答风格：${style}
约束：
1. 先给结论，再给要点；
2. 不确定的内容明确说明；
3. 输出 Markdown。
```

`PromptLoader`：

- 从 classpath 读取；
- 白名单替换 `${style}`；
- style 只允许 `严谨/温和/简洁`；
- 每次请求重新组装，不要缓存替换后的最终字符串（可缓存模板）。

硬检查时切换风格，必须真的改变 system prompt；只改前端文案不算。

## 6. 结构化抽取

### 6.1 流程

```text
ExtractController
→ 校验 text（非空，≤8000）
→ ExtractService
→ LlmClient.completeJson(prompt, ExtractResult)
→ ExtractValidator
→ 成功返回字段；失败返回空字段 + warnings
```

### 6.2 Prompt

- 明确输出 JSON；
- 给出字段定义、枚举、日期格式、示例；
- 明确“无法识别的字段置 null，不要编造”；
- `temperature=0`；
- 设置合理 `max_tokens`。

### 6.3 校验

- `docType` 在枚举内；
- `title` 非空时 trim；
- `date` 匹配 `YYYY-MM-DD`，无法识别置空 + warning；
- 其余字段允许 null；
- JSON 解析失败：不要 500；返回空字段 + `warnings`；日志记录截断后的原始响应和 traceId。

### 6.4 手工测试

- 正常通知；
- 缺日期；
- 多主送；
- 噪声文本；
- 空输入（400）；
- 模型返回 Markdown 代码围栏（应被清理/兜底）。

## 7. 错误、日志与 TraceId

- `TraceIdFilter`：为每个请求生成/透传 `X-Trace-Id`，放入 MDC。
- 日志格式：`traceId | method | path | status | durationMs`。
- SSE 日志：`firstTokenMs`、`totalMs`、`tokenCount`、`finishReason`、`errorCode`。
- 不打印：API Key、完整 Prompt、用户敏感文本、Cookie。
- 全局异常：JSON 接口返回统一错误；SSE 异常在 sink 内处理，不要走全局异常导致响应格式错乱。

## 8. 实现顺序（建议）

1. Spring Boot 骨架 + 统一响应 + 全局异常 + TraceId。
2. `.env`/配置 + DeepSeek 非流式 `/api/chat` 打通。
3. 流式 `LlmClient.stream` + `SseEmitter` + `/api/chat/stream`；curl 验证。
4. Prompt 变量 + 风格枚举；curl 对比输出。
5. 前端联调；修截断/首字延迟。
6. `/api/extract` + JSON 校验 + 兜底。
7. README、curl、演示脚本、失败案例。
8. 干净环境彩排 + 打 tag。

## 9. W2 后端验收清单

- [ ] 干净环境按 README 启动，无需数据库。
- [ ] `curl -N` 能看到 `meta/token/done`；无截断。
- [ ] 首字 ≤3s（记录证据）。
- [ ] 切换 `style` 输出变化可解释。
- [ ] 抽取正常文本返回完整字段；异常文本返回 warnings 而不是 500。
- [ ] 模型超时/429 返回 `error` 事件；日志有 traceId。
- [ ] `.env` 未入库；`git grep -i sk-` 无结果。
- [ ] README 包含启动、环境变量、curl、演示步骤、已知限制。

## 10. W2 故障排查

| 现象 | 优先检查 |
|---|---|
| 首字 >3s | 冷启动、Prompt 长度、模型选择、网络、是否先做了非流式预热 |
| 流式截断 | `emitter.complete()` 时机、代理缓冲、前端解析半包、`done` 是否发送 |
| 浏览器一次性显示 | 代理/压缩缓冲；`X-Accel-Buffering: no`；Vite proxy 是否流式 |
| 401/403 | Key/Base URL/模型名；DeepSeek 账户额度 |
| JSON 解析失败 | 模型是否输出代码围栏；Prompt 是否包含 json；max_tokens 是否过小 |
| 中文乱码 | SSE 编码/Jackson 配置；前端 TextDecoder |
| 重复 done | `AtomicBoolean` 未防重；complete 被多次调用 |

---

# 第二部分：完整项目后端（W3-W8）

## 11. 包结构

```text
com.example.qa
├─ common/        统一响应、错误码、异常、TraceId、Jackson
├─ config/        Security、Llm、Embedding、PgVector、Async、CORS
├─ security/      JwtService、JwtFilter、UserPrincipal、角色校验
├─ controller/    auth、conversation、chat、knowledge
├─ service/       业务编排
├─ domain/        entity + enum
├─ mapper/        MyBatis-Plus
├─ dto/           请求/响应
├─ llm/           ChatModel 工厂、Prompt、TokenSink
├─ rag/           解析、切片、Embedding、检索、引用
├─ graph/         AgentState、节点、边、Trace、危机响应
├─ tool/          SearchProvider、SqlQueryTool、SqlGuard、隐私守卫
└─ stream/        SSE Sink、事件模型
```

## 12. 配置与 Provider 抽象

### 12.1 配置

```yaml
app:
  llm:
    provider: cloud        # cloud | private
    chat:
      base-url: ${DEEPSEEK_BASE_URL}
      api-key: ${DEEPSEEK_API_KEY}
      model: ${DEEPSEEK_CHAT_MODEL:deepseek-flash}
      timeout-seconds: 60
    embedding:
      base-url: ${OLLAMA_BASE_URL:http://localhost:11434}
      model: ${OLLAMA_EMBEDDING_MODEL:bge-m3}
      dimension: 1024
```

W6 切换：

```yaml
app:
  llm:
    provider: private
    chat:
      base-url: http://<vllm-host>:8000/v1
      api-key: not-needed
      model: <vllm模型名>
```

### 12.2 工厂

- `ChatModelFactory`：按 provider 构建 `ChatModel` / `StreamingChatModel`。
- `EmbeddingModelFactory`：`OllamaEmbeddingModel`，baseUrl/model。
- DeepSeek 与 vLLM 都是 OpenAI 兼容，**同一套 `OpenAiChatModel`/`OpenAiStreamingChatModel` 代码**，只换配置。
- 不要写 `if (deepseek)` 的业务分支；业务层只依赖接口。
- 启动健康检查：chat、embedding 分别上报；Ollama 挂了不要让整个应用启动失败，但知识库接口返回 503。
- 维度固定 1024；换 Embedding 模型必须重建向量表并重灌。

## 13. 数据库

### 13.1 应用表（MySQL `qa_agent`）

| 表 | 关键字段 | 索引 |
|---|---|---|
| `users` | id, username, password_hash, display_name, role, status, timestamps | unique(username) |
| `conversations` | id, user_id, title, summary, last_message_at, timestamps | (user_id, last_message_at) |
| `messages` | id, conversation_id, user_id, role, content, status, intent, safety_level, citations_json, trace_json, prompt_tokens, completion_tokens, created_at | (conversation_id, id) |
| `documents` | id, original_name, category, uploaded_by, stored_path, file_hash, file_size, mime_type, status, error_message, chunk_count, timestamps | unique(file_hash), (uploaded_by, created_at) |
| `sql_query_log` | id, user_id, conversation_id, message_id, question, generated_sql, views_used, row_count, duration_ms, success, error_message, created_at | (user_id, created_at) |

- Flyway 迁移顺序：V1 users、V2 conversations/messages、V3 documents、V4 sql_query_log。
- `messages.citations_json/trace_json` 用 JSON 列；Service 层负责序列化/反序列化。
- 所有 Long ID 序列化为字符串。

### 13.2 业务库（MySQL `student_mental_health`，合成数据）

- 原始表：students、counselors、appointments、counseling_sessions、assessments、risk_alerts。
- 安全视图：`v_risk_overview`、`v_assessment_trend`、`v_appointment_stats`、`v_counselor_workload`、`v_follow_up_queue`、`v_topic_distribution`。
- `qa_ro` 只授视图 `SELECT`；SQL 工具只允许视图。
- 计数 <5 抑制；不暴露姓名/学号/咨询摘要。
- 详细定义见 `docs/product/PROJECT_PLAN.md` 第 4 章。

### 13.3 PG 向量表

- LangChain4j `PgVectorEmbeddingStore` 管理；`vector(1024)`；cosine；HNSW。
- metadata：doc_id、file_name、category、page_number、chunk_index、uploaded_by。
- 启动校验维度；维度不符直接失败并提示重建。

## 14. 认证与用户

- `users.role`：`COUNSELOR` / `ADMIN`；一期不做 STUDENT。
- 注册默认 COUNSELOR（演示）；生产邀请制。
- BCrypt；用户名唯一；密码 ≥8。
- Access JWT 30 分钟；Refresh 不透明串 7 天，Redis 白名单 + HttpOnly Cookie；轮换/撤销。
- Spring Security 无状态；JWT 过滤器；401/403 JSON；方法级角色校验。
- 所有资源查询带 `user_id`；不属于当前用户返回 404。
- 知识库是共享资料库；SQL 聚合数据对所有 COUNSELOR/ADMIN 开放；会话私有。

## 15. 会话与消息

- 会话 CRUD；首条消息自动标题（截断 30 字）；分页/游标。
- 删除会话级联消息；删除文档同时删向量+文件。
- 消息状态：`STREAMING/COMPLETED/STOPPED/FAILED`。
- 消息保存时机：先保存用户消息；助手消息流式前建 STREAMING，结束更新；不要包在一个长事务里。
- 历史加载：最近 20 条 + `conversations.summary`（超过阈值异步摘要）。

## 16. 对话编排

- `ChatOrchestrator`：加载历史/摘要/工具开关 → 调 LangGraph → 写消息 → 返回/流式。
- `TokenSink`：非流式收集器 / SSE sink；图节点内只依赖接口。
- `answer` 节点用 `StreamingChatModel`；非流式端点也走同一图，收集后返回。
- 事务边界：LLM/工具调用不在数据库事务内。
- 错误：工具失败降级；模型超时映射；不把异常栈返回前端。

## 17. RAG

### 17.1 摄取

```text
上传 → 校验(类型/大小) → SHA-256 去重 → 保存文件(PENDING)
→ 异步：PARSING → 解析 → 切片 → EMBEDDING → pgvector → READY
失败：FAILED + error_message
```

- 解析：PDF（页码 metadata）、DOCX、MD、TXT；扫描/空文本 FAILED。
- 切片：500–800 token、10%–15% overlap 起步。
- metadata 必须含 doc_id、category、page_number、chunk_index、uploaded_by。
- 删除：先按 metadata 过滤删向量，再删文件/记录；确认所用版本支持 `removeAll(Filter)`，否则先查 ID 再删。
- 重建：删旧向量后重灌；同一文档串行/幂等。
- 启动恢复：扫描 PENDING/PARSING，重新入队；避免重启卡死。
- 线程池有界；记录 chunk_count、错误。

### 17.2 检索

- query 向量化 → topK=6、minScore=0.5 起步 → 去重 → 编号 `[1]...[n]`。
- 共享知识库；可按 category 过滤。
- 引用：DOC（文件/页码/片段）、WEB（标题/URL）、SQL（视图/行数）。
- 无结果/低分：明确拒答，不硬编。
- Prompt：只依据资料、不足则说明、引用编号、资料不可信、不诊断。
- Rerank：W3 预留接口；W6 接 bge-reranker（Dify/TEI/Xinference），不要假设 Ollama 直接支持 rerank。

### 17.3 评测

- W4：15 题（5 题无答案），100% 引用，命中率 ≥70%，无答案拒答率 ≥80%。
- 按解析/切分/召回/生成归因 badcase；保存评测表。
- W8：扩到 30 题，输出准确率/召回率/耗时报告。

## 18. LangGraph4j

### 18.1 状态

`userId`、`role`、`conversationId`、`question`、`tools`、`history`、`summary`、`safetyLevel`、`intent`、`rewrittenQuery`、`retrievedChunks`、`webResults`、`sqlQuery`、`sqlRows`、`answer`、`citations`、`reflectionScore`、`reflectionNotes`、`retryCount`、`trace`、`status`。

### 18.2 节点/边

```text
START → load_context → safety_check
safety_check → crisis_response → finalize
safety_check → route
route → retrieve → answer
route → web_search → answer
route → sql_query → answer
route → answer            # DIRECT
answer → reflect
reflect → retrieve/web_search/sql_query   # 最多 2 次
reflect → finalize
finalize → END
```

- `safety_check`：危机关键词 + 分类器；命中固定危机响应。
- `route`：`RAG/WEB/SQL/DIRECT`；工具关闭/角色无权限降级。
- `sql_query`：隐私请求设 `intent=PRIVACY_REFUSED`，不执行 SQL。
- `reflect`：依据、共情、无诊断、无药物建议、无隐私泄露；不合格重试/兜底。
- `crisis_response`：固定模板 + 配置化资源；模型只能润色，不能删关键求助信息。
- Trace 包装器逐节点记录；图单例，每请求独立 State；一期不做 checkpoint/HITL。
- W6 另外完成培养计划的 3 节点 LangGraph 示例（合同审查），可用同一个 graph 基础。

## 19. SQL 工具

- 独立只读数据源 `qa_ro`；只授安全视图 SELECT。
- Schema Prompt 只列安全视图；不暴露原始表/系统库。
- JSqlParser：单条 SELECT、视图白名单、禁止系统库、禁止 DML/DDL、LIMIT ≤100。
- 隐私守卫：姓名/学号/咨询记录/个案请求直接拒绝，不执行 SQL。
- 查询超时 3 秒；失败修复一次；结果最多 20 行进 Prompt；写 `sql_query_log`。
- 小样本 `<5` 不展示；前端显示“样本过少”。
- SQL 摘要进 trace；不把完整结果集返回前端。

## 20. SSE（完整项目）

- 端点：`POST /api/conversations/{id}/messages/stream`。
- 事件：meta/safety/node/token/citation/done/error + 心跳。
- 助手消息 STREAMING→COMPLETED/STOPPED/FAILED；断连保存部分内容。
- 有界线程池；队列满 429；异步前捕获 userId/role。
- `X-Accel-Buffering: no`；Nginx 关闭 SSE 缓冲；心跳 15s；超时可配。
- 客户端断连：停止写入；模型能取消则取消，不能则记录成本。
- 限流：问答 20 次/分钟/用户；登录 10 次/分钟/IP。
- 非流式端点保留，作为兜底。

## 21. 观测与安全

- TraceId 贯穿 HTTP/SSE/工具；日志包含 durationMs、firstTokenMs、usage、errorCode。
- 不打印 Key、Token、Cookie、完整 Prompt、学生 PII。
- Actuator 只暴露 health/info；Swagger 生产关闭或加权限。
- 所有权校验、角色校验、SQL 只读、上传校验、XSS 清洗、CORS 白名单、Cookie Secure/SameSite。
- 危机响应有固定资源；不诊断、不药物建议、不承诺保密。
- 高风险场景人在回路；AI 不出具最终结论。

## 22. W3-W8 后端路线

| 周次 | 后端重点 | 交付 |
|---|---|---|
| W3 | 文件上传、解析、切片、Ollama Embedding、pgvector、引用 | 可溯源 RAG 后端 |
| W4 | 15 题评测、Dify RAG/Workflow、任务状态/幂等/异常 | 评测表 + 工作流版本 |
| W5 | JWT、用户/会话/消息、MyBatis-Plus、Swagger、traceId | 登录/历史接口 |
| W6 | Provider 切 vLLM、Dify Workflow + vLLM、私有 Embedding/Rerank、3 节点 LangGraph | 私有化接通 |
| W7 | LangGraph 安全/路由/反思、SQL 工具、工具开关 | Agent 后端 |
| W8 | SSE 完整链路、30 题评测、私有化全栈、文档 | 交付包 |

---

# 第三部分：Dify / vLLM 培养线（独立于项目代码）

| 周次 | 任务 | 验收 |
|---|---|---|
| W1 | 自建 Dify（Docker Compose）、首个应用、3 个 Prompt | 本机/内网可登录、可对话 |
| W3 | Dify 知识库 RAG、引用、更新 | 20+ 文档，15 题评测 |
| W4 | Dify Workflow：工具/分支/人工确认/异常 | 端到端可运行 |
| W6 | vLLM 服务 + Dify 接入 + 私有 Embedding/Rerank | 删除外部 Key 仍可跑；流式正常 |
| W8 | Compose 一键全栈 | 干净环境 ≤30 分钟 |

- vLLM 是推理引擎，不是模型；Ollama 不能直接替代 W6 硬检查，除非导师同意。
- 你的 RTX 4050 6GB 跑 7B/14B vLLM 风险高；W5 前落实 Linux/云 GPU。
- Dify 云版不算私有化；确认访问地址是自建。
- Ollama bge-m3 是 Embedding；Rerank 需要 bge-reranker 单独服务（TEI/Xinference/Dify Provider）。

---

## 23. 后端 Definition of Done

- [ ] 代码可启动；README/.env 完整；无密钥。
- [ ] Swagger/契约同步。
- [ ] 手工验收通过；错误可追踪（traceId）。
- [ ] LLM/工具调用不在长事务内。
- [ ] SSE 断连/超时清理；消息状态正确。
- [ ] SQL 只读 + 白名单 + 隐私守卫。
- [ ] 知识库删除后向量不可检索。
- [ ] 两人各自仓库均能独立跑通，不只一方会。

---

## 24. 需要确认

- Dify 网页是 `cloud.dify.ai` 还是自建？云版不算私有化。
- W6 是否必须 vLLM，还是导师接受 Ollama/等价 OpenAI 兼容服务？
- 是否有 Linux GPU/云 GPU；6GB Laptop 跑 vLLM 风险高。
- MyBatis-Plus + Element Plus 默认是否接受；如换 JPA/Naive UI，结构不变，任务照旧。
