---
type: think
author: liuyu
created: '2026-05-06T21:07:23+08:00'
index_state: indexed
---
# 方案 B 真实流程验证报告

承接 think `007_liuyu_think_c2487c.md` 提交的"AI Runner + SSE heartbeat + 前端五态状态机"设计。
方案 A/B/C 三个 commit (`afb2e90` / `c88e4b3` / `f8f9d16`) 已通过 merge commit `0b80ccc` 落到
`feat/user-management`。本次报告的是落地后的真实流程验证 + 验证过程中发现的 3 个隐藏问题。

## 1. 验证方法

新增了 `scripts/mock_slow_ai.py` —— OpenAI 兼容协议的 mock 上游，五种模式（fast / slow /
stalled / auth / server_5xx），把"OpenRouter 真实卡顿 / 鉴权失败 / 5xx"等不可控场景变成
本地可复现。dev 环境 admin 设置里把 base_url 切到 `http://localhost:9100/v1` 即可。

## 2. 场景结果（5/5 通过）

| 场景 | mock 模式 | 验证项 | 结果 |
|---|---|---|---|
| 1 | fast | 流式 token 控制组（不应触发 banner） | ✅ |
| 2 | auth (401) | 红色错误条 + **不显示重试按钮**（retryable=false） | ✅ |
| 3 | slow (12s 静默) | 黄色 heartbeat banner 出现 → token 接续后自动消失 | ✅ |
| 4 | stalled (60s 不回包) | NewMatter summary 桶 hard_timeout 触发 + 友好降级 | ✅ |
| 5 | server_5xx (503) | 红色错误条 + **重试按钮** + 累积失败历史 | ✅ |

场景 4 走的是 NewMatterGuidedFlow 而非 AIPane（产品上引导式 NewMatter 默认是步骤化引导）；
GuidedFlow 自带降级 UI（"AI 没有响应，请重试" + "可以再试一次或直接跳到「自己写」模式"），
我们方案 B 的后端归一化（`upstream_timeout` code + 中文用户文案）在它的 toast 里也正确展示
了。AIPane 路径的 F4 红色错误条 + 重试按钮则在场景 5 里完整验证。

## 3. 验证拽出的 3 个真 bug（已修，commit `f6d59f4`）

### 3.1 httpx 默认走系统代理拦截 localhost

**现象**：场景 1 第一次跑就报 `parse_error AI endpoint 404: 404 page not found`。直接 curl
打 mock 200 OK，但 Python httpx 同 URL 拿到 404。

**根因**：Windows 上常见的 Clash / v2ray / VPN 客户端会拦截 `localhost:*` 流量交给本机代理；
httpx 默认 `trust_env=True` 读环境/系统代理设置，结果走了代理，被代理 404 回来。

**修复**：`server/ai/client.py` 在打 loopback URL（`localhost` / `127.0.0.1` / `::1`）时
显式 `trust_env=False`。生产打 OpenRouter / DashScope 仍正常走代理。

### 3.2 `asyncio.wait_for(__anext__())` 取消底层 generator

**现象**：场景 3 第一次跑发现"第一个 token 出来 → 一次 heartbeat → 流就结束了"，本应在 12s
后到达的 "我醒了！" 永远收不到。

**根因**：`asyncio.wait_for` 超时时取消内部 awaitable；async generator 的 `__anext__()` 一旦
被取消，generator 的 frame 就毁了，下次再 iterate 立即 `StopAsyncIteration`。这是 Python
异步迭代经典坑。

**修复**：`server/ai/client.py` 改用 `asyncio.ensure_future` 把 `__anext__()` 包成 task，
`asyncio.wait({task}, timeout=...)` 跨 heartbeat tick 复用同一个 task；timeout 不取消它，
只是回头再等。`finally` 里负责关闭时的 task cleanup。

### 3.3 重试历史：抹除 vs 保留

**初始设计**（think 007 §5）：重试时把上次失败的 user / assistant 占位对从 messages 里抹掉，
"假装从未发生过"。

**真实使用反馈**：跑场景 5 时观察到，用户期望**看到**自己重试了几次都失败的痕迹（决定要不
要换问法或放弃）。

**修复**：`web/src/pages/Dashboard.tsx` `retryLastSend` 改为只清交互态（errorDetail / slow），
保留 messages，让 sendMessage 在尾部 append 新的 user / assistant 对。心智上对齐 IM 工具的
"再发一次"，不是"撤回重写"。

## 4. 当前状态

* `feat/user-management` HEAD = `f6d59f4`，已推到 origin
* 38/38 targeted 测试通过；TypeScript build clean
* 全 5 个场景在浏览器走通

## 5. 沉淀

* `scripts/mock_slow_ai.py` 留作长期工具，后续触类旁通的 AI 异步路径都可以用它做注入式验证
* `AICallSpec` + 错误码归一化 + SSE heartbeat 协议这套契约已稳定，后续再上新的 AI 入口
  （比如 daily-report 重跑、批量摘要）直接复用 `spec_for(...)` 即可
