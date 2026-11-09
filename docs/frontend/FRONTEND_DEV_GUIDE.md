# 前端开发指南
## 心屿 · 学生心理健康辅助 Agent（W2 Lab + 完整项目）

> 版本：v1.0  
> 日期：2026-11-07  
> 默认：Vue3 + JavaScript（**不使用 TypeScript**）+ Vite + Vue Router + Pinia + Element Plus + Axios；Markdown 用 markdown-it + DOMPurify + highlight.js；流式用原生 `fetch` + ReadableStream。  
> 重要：培养计划要求每个人都能独立跑通前端、后端和 Agent。本文档供你**自己完整实现**，搭档可以 Review，但不能一人只做前端、一人只做后端。

---

## 0. 先分清两个前端

| | W2 Lab 前端 | 完整项目前端 |
|---|---|---|
| 目标 | 11/13 硬检查：流式 + Prompt 变量 + 抽取 | W7-W8 综合项目 |
| 仓库 | 个人 `w2-stream-lab` | 个人综合项目仓 |
| 依赖 | Vue3 + Vite + markdown-it | Vue3 + Router + Pinia + Element Plus |
| 登录 | 无 | JWT 登录/刷新/路由守卫 |
| 数据 | 无历史 | 会话、消息、知识库 |
| 重点 | SSE 解析、Markdown、首字计时 | 全栈体验、引用/轨迹、知识库、安全 UI |
| 不要做 | 登录/MySQL/RAG | 不要拖到 W2 |

W2 只做最小页面；完整项目的复杂状态从 W3/W5 再引入。

---

# 第一部分：W2 Lab 前端

## 1. 目标与验收

- 页面输入问题 → 调 `/api/chat/stream` → 逐字显示 → Markdown 正常 → 错误可读。
- 首字 ≤3s；长输出不截断；切换“严谨/温和/简洁”输出有变化。
- 公文抽取页：粘贴文本 → 展示字段与 warnings；失败不白屏。
- 干净环境按 README 启动；无 TypeScript；无 Key。

## 2. 技术栈与创建

```bash
npm create vite@latest frontend -- --template vue
cd frontend
npm install
npm install vue-router markdown-it dompurify highlight.js
```

- 不要选 `vue-ts`；不要生成 `.ts` 文件；不要 `vue-tsc`。
- 样式可用原生 CSS；Element Plus 可留到完整项目，W2 不必引入。
- 关键 API 对象用 JSDoc 注释（可选，但推荐）。

## 3. 目录结构

```text
frontend/
├─ src/
│  ├─ api/
│  │  ├─ http.js
│  │  ├─ chat.js
│  │  └─ extract.js
│  ├─ components/
│  │  ├─ ChatMessage.vue
│  │  ├─ ChatInput.vue
│  │  ├─ MarkdownRenderer.vue
│  │  ├─ StyleSelect.vue
│  │  └─ ExtractionPanel.vue
│  ├─ composables/
│  │  ├─ useChatStream.js
│  │  └─ useMarkdown.js
│  ├─ views/
│  │  ├─ ChatView.vue
│  │  └─ ExtractView.vue
│  ├─ router/index.js
│  ├─ utils/sse.js
│  ├─ constants/index.js
│  ├─ App.vue
│  └─ main.js
├─ vite.config.js
├─ .env.development
└─ README.md
```

## 4. Vite 代理与环境变量

`vite.config.js`：

```js
server: {
  port: 5173,
  proxy: {
    '/api': {
      target: 'http://localhost:8080',
      changeOrigin: true
    }
  }
}
```

- 前端只写 `/api/...`，不要写死后端地址。
- 不需要在前端配置 DeepSeek Key。
- `.env.development` 可放 `VITE_APP_TITLE` 等非敏感变量；不要放 Key。
- SSE 走 Vite 代理时确认不缓冲；若出现一次性输出，直接用后端 8080 对比。

## 5. SSE 解析工具

### 5.1 为什么不用 EventSource

- EventSource 只能 GET，不能带 `Authorization` 和 JSON body。
- 本项目要 POST + Header，所以用 `fetch` + `ReadableStream`。

### 5.2 `utils/sse.js` 设计

输入：`ReadableStream<Uint8Array>`；输出：事件回调。

处理规则：

- 用 `TextDecoder('utf-8')` 流式解码，保留跨包半字符。
- 按空行 `\n\n` 分帧；兼容 `\r\n`。
- 每帧解析 `event:`、`data:`（可能多行）、`id:`、注释 `:`。
- 忽略 `: ping`。
- `data` 通常是 JSON，解析失败要容错并记录，不要中断整个流。
- 收到 `done`/`error` 后停止读取并取消 reader。
- 处理 `AbortController`：用户点击停止/切换页面时 abort。

伪代码：

```text
buffer += decoder.decode(chunk, {stream:true})
frames = buffer.split(/\r?\n\r?\n/)
buffer = frames.pop()
for frame of frames:
  event = 'message'
  dataLines = []
  for line of frame.split(/\r?\n/):
    if line startsWith ':' -> skip
    if line startsWith 'event:' -> event = ...
    if line startsWith 'data:' -> dataLines.push(...)
  onEvent(event, parse(dataLines.join('\n')))
```

### 5.3 `useChatStream.js`

返回：

```js
{
  send,          // ({message, style}) => Promise
  stop,          // abort
  streaming,     // ref
  error,         // ref
  firstTokenMs,  // ref
  reset
}
```

内部状态：

- `controller`：AbortController。
- `startedAt`：performance.now()。
- `firstTokenAt`：首个 token 时间。
- `onToken(delta)`：追加到当前助手消息。
- `onDone(payload)`：标记完成，记录 usage/firstTokenMs。
- `onError(err)`：保留已生成内容，显示错误。

## 6. 页面与组件

### 6.1 `ChatView.vue`

- 消息列表：用户/助手气泡；助手流式时显示光标/加载态。
- 输入框：Enter 发送、Shift+Enter 换行；空内容禁用；流式中禁用发送/允许停止。
- 风格选择：严谨/温和/简洁；发送时带给后端。
- 首字计时显示（开发模式）：`首字 0.9s / 总 4.2s`。
- 错误提示：保留部分回答 + 重试按钮。
- 自动滚动：仅当接近底部时跟随；用户上滑后不要强制拉回。
- 空状态：示例问题（“用三句话解释 SSE”等）。

### 6.2 `ChatMessage.vue`

Props：

| prop | 类型 | 说明 |
|---|---|---|
| message | Object | `{id, role, content, status}` |
| streaming | Boolean | 是否正在流式 |

- 用户消息纯文本（或简单换行）；助手消息用 MarkdownRenderer。
- 不要用 `v-html` 直接渲染未清洗内容。

### 6.3 `MarkdownRenderer.vue`

Props：`content: String`。

处理链：

1. `markdown-it({ html: false, linkify: true, breaks: true })`
2. DOMPurify 清洗
3. highlight.js 代码高亮（可在结束后做，避免流式卡顿）
4. 代码块复制按钮（事件委托）
5. 外链 `target="_blank" rel="noopener noreferrer"`

性能：

- 流式时不建议每个 token 全量 Markdown 渲染；按 30–50ms 节流。
- 代码高亮可在 `done` 后做；流式中先纯文本/简单换行，避免重复高亮。
- 用 `shallowRef` 或节流减少 Vue 更新。

### 6.4 `ChatInput.vue`

Emits：`send({message, style})`、`stop()`。

- `disabled` 时不可发送；流式中显示“停止”。
- 输入长度限制与后端一致（1–4000）。
- 不要在组件内部直接 fetch；交给父组件/composable。

### 6.5 `ExtractionPanel.vue`

- 文本域 + 示例按钮 + 提交。
- 展示字段：文种、标题、主送、落款、日期。
- `warnings` 用黄色提示；失败时显示“需人工核对”，不白屏。
- loading/错误/清空。
- 不接学生业务数据；只用公文样例。

## 7. 实现顺序

1. Vite + Vue 骨架、Router、代理、全局样式。
2. `sse.js` + 假数据测试（用一个 mock ReadableStream 或后端 curl）。
3. `useChatStream` + ChatView 接入非流式 `/api/chat`，确认渲染。
4. 接 `/api/chat/stream`；首字计时；修截断。
5. 风格选择 + Prompt 变量；对比输出。
6. Markdown/XSS/代码复制/自动滚动。
7. 抽取页 + warnings + 错误态。
8. README、演示脚本、构建。

## 8. W2 前端验收清单

- [ ] 干净环境 `npm install && npm run dev` 可启动。
- [ ] 流式逐字显示；长输出不截断；`done/error` 正确处理。
- [ ] 首字计时可见；≤3s。
- [ ] 切换风格输出变化；能解释。
- [ ] Markdown 正常；XSS 测试不执行；代码块可复制。
- [ ] 抽取字段正确；异常文本显示 warnings；不白屏。
- [ ] 无 `.ts`；无 Key；README 完整。
- [ ] 移动端基本可用。

## 9. W2 故障排查

| 现象 | 检查 |
|---|---|
| 一次性显示 | 代理/压缩缓冲；`X-Accel-Buffering: no`；后端是否逐条 send |
| 中文乱码 | TextDecoder 是否流式；Content-Type charset |
| 事件丢失 | 半包 buffer；`\r\n`；空行分帧；多行 data |
| 点停止报错 | AbortController 后 reader.cancel；不要把 abort 当业务错误 |
| Markdown 卡顿 | 每 token 全量渲染；改为节流/结束高亮 |
| XSS | markdown-it `html:false` + DOMPurify；外链协议白名单 |
| 首字慢 | 后端冷启动/模型延迟；前端计时从 fetch 开始还是首 token 要注明 |

---

# 第二部分：完整项目前端（W3-W8）

## 10. 目录结构

```text
src/
├─ api/
│  ├─ http.js          # axios 实例、拦截器、刷新队列
│  ├─ auth.js
│  ├─ conversation.js
│  ├─ chat.js
│  └─ knowledge.js
├─ components/
│  ├─ layout/          # AppSidebar、AppHeader、UserMenu
│  ├─ chat/            # MessageList、ChatMessage、ChatInput、ToolToggles
│  ├─ markdown/        # MarkdownRenderer、CodeBlock
│  ├─ citation/        # CitationList、CitationCard、CitationDrawer
│  ├─ trace/           # TraceTimeline、TraceItem
│  ├─ safety/          # SafetyBanner、CrisisCard
│  ├─ knowledge/       # UploadPanel、DocumentTable、StatusTag、StatsCards
│  └─ common/          # EmptyState、ErrorState、Loading、ConfirmDialog
├─ composables/
│  ├─ useChatStream.js
│  ├─ useMarkdown.js
│  └─ useAutoScroll.js
├─ layouts/AppLayout.vue
├─ router/index.js
├─ stores/
│  ├─ auth.js
│  ├─ conversation.js
│  ├─ chat.js
│  └─ knowledge.js
├─ constants/
│  ├─ enums.js
│  └─ errorCodes.js
├─ utils/
│  ├─ sse.js
│  ├─ url.js
│  ├─ format.js
│  └─ storage.js
├─ views/
│  ├─ LoginView.vue
│  ├─ RegisterView.vue
│  ├─ ChatView.vue
│  ├─ KnowledgeView.vue
│  └─ NotFoundView.vue
├─ App.vue
└─ main.js
```

## 11. 路由与布局

| 路径 | 页面 | 守卫 |
|---|---|---|
| `/login` | 登录 | 已登录跳 `/chat` |
| `/register` | 注册 | 已登录跳 `/chat` |
| `/chat/:id?` | 聊天 | 需要登录 |
| `/knowledge` | 知识库 | 需要登录 |
| `/` | 重定向 `/chat` | 需要登录 |
| `/:pathMatch(.*)*` | 404 | 无 |

- AppLayout：左侧会话栏 + 顶部用户菜单 + 主内容区；移动端侧栏抽屉。
- 路由守卫：无 Access Token 时先尝试 `refresh`；失败跳登录。
- 页面标题随路由变化。

## 12. Axios 与登录态

### 12.1 `http.js`

- `baseURL: '/api'`；超时 30s；`withCredentials: true`（Refresh Cookie）。
- 请求拦截：内存中的 Access Token 加 `Authorization`。
- 响应拦截：解包 `{code,message,data,traceId}`；非 0 抛统一错误。
- 401：触发刷新队列；刷新成功重放原请求一次；失败清状态跳登录。
- 排除 `/auth/login`、`/auth/register`、`/auth/refresh`，避免递归。

### 12.2 刷新队列（必须防并发）

```text
refreshPromise = null
on401(config):
  if config._retried: logout
  if !refreshPromise:
    refreshPromise = authApi.refresh().finally(() => refreshPromise = null)
  await refreshPromise
  update token
  config._retried = true
  retry request
```

- 多个并发 401 只刷新一次。
- 刷新失败：清 authStore、跳登录、不要无限重试。
- Access Token 只在内存；页面刷新时 `authStore.bootstrap()` 调 `/auth/refresh` 恢复。
- 不要把 Refresh Token 写入 localStorage；不要读 Cookie。

### 12.3 Auth Store

状态：`user`、`accessToken`、`initialized`、`loading`。

方法：`login`、`register`、`logout`、`refresh`、`bootstrap`、`hasRole`。

- 登录页成功后拉取/使用响应中的 user，跳转上次会话或 `/chat`。
- 登出调用后端撤销 Refresh，清内存，跳登录。

## 13. Pinia Stores

### 13.1 `conversationStore`

状态：`items`、`page`、`size`、`total`、`currentId`、`loading`、`messages`、`nextCursor`。

方法：`fetchList`、`create`、`rename`、`remove`、`select`、`loadMessages`、`loadOlder`。

- 删除当前会话后跳 `/chat`。
- 切换会话时先中止当前 SSE 流，避免消息串到新会话。
- 列表分页去重按 id。

### 13.2 `chatStore`

状态：`sending`、`streamingMessageId`、`error`、`citations`、`trace`、`safety`。

方法：`send`、`stop`、`handleEvent`、`reset`。

- `send` 先插入用户消息 + 助手占位；流式更新助手内容。
- 事件处理：
  - `meta`：记录 messageId。
  - `safety`：存资源，显示 CrisisCard。
  - `node`：按 node upsert trace。
  - `token`：追加内容；节流渲染。
  - `citation`：更新引用。
  - `done`：状态 COMPLETED；用 done.trace 覆盖。
  - `error`：保留部分内容，状态 FAILED/STOPPED。
- 停止：AbortController；本地标记 STOPPED。
- 切换会话/离开页面：abort + 清理。
- 防重复：流式中禁用发送。

### 13.3 `knowledgeStore`

状态：`documents`、`page`、`statusFilter`、`categoryFilter`、`stats`、`pollingTimer`。

方法：`fetch`、`upload`、`remove`、`reindex`、`startPolling`、`stopPolling`。

- 有 PENDING/PARSING/EMBEDDING 时每 2s 轮询；页面隐藏暂停；无进行中停止。
- 上传进度用 axios `onUploadProgress`（或其他 XHR 方案）。
- 删除/重建后刷新 stats。

## 14. 聊天页组件

### 14.1 页面结构

```text
ChatView
├─ ConversationSidebar
├─ ChatHeader（标题、工具开关、用户菜单）
├─ MessageList
│  └─ ChatMessage
│     ├─ MarkdownRenderer
│     ├─ CitationList
│     └─ TraceTimeline（折叠）
├─ SafetyBanner / CrisisCard
└─ ChatInput
```

### 14.2 `ChatMessage` Props/Emits

Props：`message`、`isStreaming`、`showTrace`。
Emits：`open-citation`、`toggle-trace`、`retry`。

- 助手消息：内容 + 引用 + 意图标签 + 安全等级 + 折叠轨迹。
- 用户消息：纯文本换行。
- 流式中显示光标/停止按钮；错误显示重试。
- 长内容折叠“展开全文”（可选）。

### 14.3 `ChatInput`

Props：`disabled`、`streaming`、`tools`。
Emits：`send({content, tools})`、`stop`。

- Enter 发送、Shift+Enter 换行。
- 工具开关：知识库/联网/业务数据；默认全开。
- 空内容禁用；发送中禁用；流式中显示停止。

### 14.4 工具开关

- 传给后端 `tools: ["KNOWLEDGE","WEB","SQL"]`。
- 后端仍按角色/配置裁剪；前端不做安全判断。
- 工具关闭时，回答应说明降级，不要假装查过。

## 15. 引用与执行轨迹

### 15.1 CitationList

- 按类型分图标：DOC/WEB/SQL。
- DOC：文件名 + 页码 + 片段，点击打开抽屉/弹层。
- WEB：标题 + URL，仅 http/https，`noopener noreferrer`。
- SQL：视图名 + 行数；不展示完整结果集。
- 引用编号与正文 `[n]` 对应；不做复杂内联跳转也可接受。

### 15.2 TraceTimeline

- 折叠在助手消息下方，默认收起。
- 每项：节点标签、状态、耗时、摘要；工具节点可展开 detail。
- 流式中按 node upsert；结束后以 done.trace 为准。
- 不展示 Prompt 原文/学生记录。

### 15.3 SafetyBanner / CrisisCard

- 页面固定免责声明：“本助手不提供诊断或处方；紧急情况请联系 120/110/12356 或学校心理中心。”
- `safetyLevel=CRISIS`：醒目卡片 + 资源列表；不要渲染成普通回答。
- 资源来自 `safety` 事件或前端常量；电话可点击 `tel:`。
- 隐私拒绝：`intent=PRIVACY_REFUSED`，用解释性卡片，不显示错误样式。

## 16. 知识库页

- 上传：拖拽/点击；PDF/DOCX/MD/TXT；≤20MB；分类选择；进度条。
- 警告文案：“禁止上传学生个案、咨询记录、含姓名/学号/联系方式的资料。”
- 表格：文件名、分类、大小、状态、分片数、上传人、时间、错误信息。
- 状态 Tag：PENDING/PARSING/EMBEDDING/READY/FAILED；失败可展开原因。
- 操作：删除（确认）、重建索引；轮询状态。
- 统计卡片：文档数、分片数、READY/FAILED。
- 空态/加载/错误；分页。
- 移动端表格可改为卡片列表。

## 17. SSE（完整项目）

- 复用 W2 的 `utils/sse.js` 和 `useChatStream.js`；增加事件处理。
- `fetch('/api/conversations/{id}/messages/stream', {method:'POST', headers, body, signal})`。
- 事件：
  - `meta`：创建占位。
  - `safety`：资源卡片。
  - `node`：更新轨迹。
  - `token`：追加；30–50ms 节流。
  - `citation`：引用。
  - `done`：最终 trace/usage；刷新会话列表。
  - `error`：保留部分 + 重试。
- 停止：AbortController；后端保存 STOPPED；刷新后可见部分内容。
- 首 token 失败：可回退非流式一次；已输出后失败保留部分内容。
- 切换会话/离开页面：abort + 清理定时器。
- 不要用 EventSource；不要在前端解析 DeepSeek 原始 SSE。

## 18. 安全与体验

- Markdown：`html:false` + DOMPurify；外链协议白名单。
- 不把 Access Token 放 localStorage；不打印 Token/Key。
- 不在前端拼接 SQL；业务数据只走后端工具。
- 错误提示不暴露堆栈/内部路径。
- 空态、加载、错误、重试、404。
- 自动滚动仅在接近底部；提供“回到底部”。
- 移动端：侧栏抽屉、引用抽屉、表格卡片化。
- 键盘：Enter/Shift+Enter、Esc 关闭弹层、焦点管理。
- 性能：流式节流、长列表虚拟化可选、避免每 token 高亮。

## 19. 前端 W3-W8 路线

| 周次 | 前端重点 | 交付 |
|---|---|---|
| W3 | 上传、文档列表、引用展示、检索结果高亮 | 知识库页 + 引用卡片 |
| W4 | 工作流状态可视化、人工确认弹窗、导出 | Workflow UI（Dify 或项目内） |
| W5 | 登录、路由守卫、Pinia、历史会话、模板管理 | 登录 + 历史 + 模板页 |
| W6 | 私有化环境联调、错误提示、traceId 展示 | 私有链路可用 |
| W7 | 引用/轨迹/安全 UI、工具开关、响应式 | Agent 体验完整 |
| W8 | SSE 打磨、空态/错误/演示、构建 | 交付演示 |

## 20. 前端 Definition of Done

- [ ] 干净环境按 README 启动；build 通过。
- [ ] 登录/刷新/登出/401 队列正确；Token 不落 localStorage。
- [ ] 会话 CRUD/分页/切换不串消息。
- [ ] 流式逐字、停止、错误、引用、轨迹正常。
- [ ] Markdown XSS 清洗；外链安全。
- [ ] 知识库上传/轮询/删除/重建可用。
- [ ] 危机/隐私提示清楚；固定免责声明可见。
- [ ] 移动端基本可用。
- [ ] 无 TypeScript；无 Key；两人都能独立跑通。

---

## 21. 需要确认

- Element Plus 默认是否接受；若换 Naive UI/Ant Design Vue，组件结构不变。
- 引用展示是否需要内联角标可点击（[1] 跳来源）；一期可只做下方来源卡片。
- 是否需要在 W4 做“人工确认弹窗”；取决于 Dify Workflow 还是项目内 LangGraph。
- 知识库是共享资料库（当前默认）；若改为个人私有，列表/检索需加 user_id 过滤。
