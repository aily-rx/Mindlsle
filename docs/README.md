# 文档中心

本目录是项目全部文档的唯一入口。后续所有需求、设计、接口、前后端实现、计划与汇报文档都必须按分类放入此目录。

## 分类目录

| 目录 | 范围 | 当前文档 |
|---|---|---|
| `product/` | 产品需求、总体方案、范围与验收 | `PRD.md`、`PROJECT_PLAN.md` |
| `api/` | 前后端共享的接口契约、数据结构、错误码、SSE 事件 | `API_CONTRACT.md` |
| `backend/` | 后端架构、数据库、Agent、RAG、SQL、SSE、安全实现 | `BACKEND_DEV_GUIDE.md` |
| `frontend/` | 前端架构、页面、组件、状态管理、SSE 解析、UI 安全 | `FRONTEND_DEV_GUIDE.md`、`w2-prototype.md` |

> 学习笔记、周计划与汇报材料属于个人内部文档，仅保存在本地 `docs/planning/`（已被 `.gitignore` 忽略），不推送到仓库。

## 归属规则

1. 一份文档只属于一个分类；跨端内容拆分到 `backend/`、`frontend/`，共享契约放 `api/`。
2. 前后端文档禁止混放；接口字段变更必须同步 `api/API_CONTRACT.md`。
3. 产品/范围/验收放 `product/`；学习笔记、周计划等内部文档不入库。
4. 新增分类目录前先更新本文件，避免目录无序扩张。
5. 历史设计文档保留原名；新增文档使用小写 kebab-case（如 `rag-pipeline.md`）。

## 文档优先级

1. `api/API_CONTRACT.md`：接口唯一事实来源。
2. `product/PROJECT_PLAN.md`：总体设计与排期基线。
3. `backend/`、`frontend/` 开发指南：实现参考。
4. `product/PRD.md`：需求摘要与项目结构决策。

## 写作约定

- 中文为主；标题、表格、代码块保持可渲染。
- 链接使用仓库根相对路径（如 `docs/backend/xxx.md`）。
- 不写入真实学生 PII、密钥、Token、完整 Prompt 或未脱敏日志。
- 变更记录写在文档末尾；接口/需求变更必须先改文档再改代码。
