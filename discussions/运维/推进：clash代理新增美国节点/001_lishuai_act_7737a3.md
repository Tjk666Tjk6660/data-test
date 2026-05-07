---
type: act
author: lishuai
created: '2026-05-06T20:50:12+08:00'
body_source: ai
index_state: indexed
---
# 推进：clash代理新增美国节点

## 背景

当前 Clash 代理缺少美国地区节点。为满足相关业务与访问需求，计划新增一个美国节点。本项目技术栈明确为：服务端采用 Sing-box，客户端统一使用 Clash Verge，底层部署机器规格为 t3.medium。

## 推进事项

1. **环境部署**：在 t3.medium 实例上完成 Sing-box 服务端的安装、调优与基础网络策略配置。
2. **代理接入方案**：流量转发与代理配置拟采用以下两种架构，后续根据实测效果确定最终方案或做主备切换：
   - CloudFront + gRPC-Query
   - ELB + gRPC-Query
3. **客户端联调**：更新 Clash Verge 配置文件，确保能正确解析上述 gRPC-Query 代理协议并完成鉴权与路由。
4. **连通性验证**：已圈定相关人员对新增节点进行可用性、延迟及带宽表现验证。

## 验收

- **配置完备**：确认域名解析、证书配置及 gRPC-Query 接入参数已全部就位且正确。
- **指标达标**：Reviewers 验证节点延迟、连通性及带宽表现符合预期，确认无误后正式推送到团队配置。
