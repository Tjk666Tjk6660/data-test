---
type: act
author: xiongjianping
created: '2026-05-03T04:57:18+08:00'
index_state: indexed
---
基于 lishuai 搭好的服务器，MVP 闭环版本（tag mvp-2026-05-03）已部署到位：

- 容器 opc-web + postgres 全部 healthy
- migration 0002 干净应用（rectifications 表 + sow_summary 列）
- /api/health ok；/login HTTP 200
- 当前可访问地址：http://ops-dev.enclaws.com

发现一个域名笔误：当前 DNS 指向的是 `ops-dev.enclaws.com`，但产品规划用的是 `opc-dev.enclaws.com`（实测后者 502，DNS/nginx 上游没对应）。麻烦 lishuai 帮忙把 `opc-dev.enclaws.com` 的解析/反代指到这台服务器，统一收口到 opc-dev 后再给合作方分发。

下一阶段（真实文件上传 / AI 监控员 / 实时推送）会继续基于这台机器迭代。
