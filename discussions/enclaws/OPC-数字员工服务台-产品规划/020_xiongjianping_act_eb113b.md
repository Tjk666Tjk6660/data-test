---
type: act
author: xiongjianping
created: '2026-04-29T16:31:34+08:00'
index_state: indexed
---
# OPC Web MVP 进入执行阶段

基于 019 的评审结论，我已经把五一前 MVP 从方案讨论推进到执行准备阶段，并在本地 `D:\repository\opc-web\docs` 下补齐了 5 份执行文档：

- `prd-mvp.md`：产品需求文档，明确 MVP 不是静态演示，而是跑通真实业务闭环。
- `技术方案-mvp.md`：技术方案，明确技术栈、模块拆分、数据模型、状态机、API 边界、权限与降级策略。
- `测试用例-mvp.md`：测试用例，覆盖 Auth、租户隔离、入驻审批、项目主流程、文件、时间线、enClaws 降级和端到端路径。
- `用户手册-mvp.md`：用户手册，面向合作伙伴试用、平台运营人员和内部测试人员，说明真实功能、演示态功能和不做范围。
- `部署手册-mvp.md`：部署手册，明确 `opc-web.enclaws.com` 的部署形态、环境变量、数据库、文件目录、Docker Compose/容器平台部署、上线验收和回滚方式。

关键内容已经从“产品概念”收敛为“五一前可交付闭环”：

1. MVP 主线是：注册登录 -> 创建公司 -> 入驻审批 -> 发布需求 -> 申请接单 -> 合同确认 -> 履约交付 -> 验收完成。
2. 真实功能包括账号、租户、公司、入驻审批、项目大厅、接单申请、合同流程、文件上传、交付验收、项目时间线和平台运营视角。
3. AI 与 enClaws 作为可选增强：可展示需求澄清、匹配建议、验收建议，但 enClaws 不可用时必须自动降级为本地建议或人工推进，不能阻断主流程。
4. 技术方案建议采用 React + Vite + TypeScript、Fastify、PostgreSQL、Drizzle，并拆分 Auth、Tenant、Company、Platform、Project、Contract、Fulfillment、Timeline、File、enClaws Adapter 等模块。
5. 测试策略以自动化为主，目标覆盖 90% 以上；E2E 必须跑完整主链路，且上线前手动跑一次公网冒烟清单。
6. 部署方案按单应用交付，API 服务托管前端静态资源，PostgreSQL 和上传目录必须持久化，默认先关闭 enClaws，保证人工闭环可用。

完整文档目前先不直接贴到 Pivot，避免长文刷屏；等附件功能上线后再把 5 份 md 文档作为附件补齐。当前阶段可以先通过仓库查看完整内容：

https://github.com/hashSTACS-Global/opc-web.git
