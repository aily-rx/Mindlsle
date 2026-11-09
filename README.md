# Project-Study · 心屿

学生心理健康辅助问答 Agent 的**共享文档仓库**。本仓库只承载需求、设计、接口与开发指南，不包含可运行代码；后端、前端代码分别在独立交付仓库中实现。

> 当前阶段：**W2 Lab**（SSE 流式对话 + Prompt 变量 + 结构化抽取）。

## 文档导航

| 分类 | 文档 | 说明 |
|---|---|---|
| 产品 | [PRD](docs/product/PRD.md) | 产品需求、范围、功能清单、验收标准 |
| 产品 | [总体方案](docs/product/PROJECT_PLAN.md) | 架构、数据设计、分工、里程碑 |
| 接口 | [API 契约](docs/api/API_CONTRACT.md) | 前后端接口唯一事实来源 |
| 后端 | [后端开发指南](docs/backend/BACKEND_DEV_GUIDE.md) | W2 Lab 与完整项目后端实现 |
| 前端 | [前端开发指南](docs/frontend/FRONTEND_DEV_GUIDE.md) | W2 Lab 与完整项目前端实现 |
| 前端 | [W2 前端原型](docs/frontend/w2-prototype.md) | W2 两个页面的低保真原型、状态与交互 |
| 规范 | [文档中心说明](docs/README.md) | 文档分类、命名与写作规范 |

## 目录结构

```text
Project-Study/
├─ README.md
├─ .env.example
├─ .gitignore
└─ docs/
   ├─ README.md
   ├─ product/     # PRD、总体方案
   ├─ api/         # 前后端共享契约
   ├─ backend/     # 后端文档
   └─ frontend/    # 前端文档
```

## 文档规则

1. 所有内容文档统一放在 `docs/` 下，并按功能分类；后续产出同样遵守。
2. 前端/后端文档分别放入 `docs/frontend/`、`docs/backend/`，不混放。
3. 共享接口契约放 `docs/api/`；产品与方案放 `docs/product/`；学习笔记、周计划等内部文档不入库。
4. 新增文档使用小写 kebab-case 命名；既有基线文档保留原名。
5. 需求与契约变更：先改文档，再改 Swagger/代码，不允许静默变更。

## 交付仓库

| 仓库 | 用途 | 状态 |
|---|---|---|
| `w2-stream-lab` | 个人 W2 实验仓（SSE + Prompt 变量 + 结构化抽取） | 待创建 |
| `mental-health-agent`（暂名） | 个人完整项目仓（W3–W8） | 待 W3 创建 |

## 说明

- 本仓库不存放密钥、真实学生数据或上传文件；`.env` 已被忽略。
- 学习笔记、周计划与汇报材料属于个人内部文档，保存在本地 `docs/planning/`（已被 `.gitignore` 忽略），不推送到仓库。
- 心理健康场景的安全边界见 [PRD](docs/product/PRD.md) 与 [总体方案](docs/product/PROJECT_PLAN.md)。
