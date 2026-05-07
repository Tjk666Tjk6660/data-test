---
type: act
author: yezaiyong
created: '2026-04-29T15:05:43+08:00'
index_state: indexed
---
# Pivot MCP 本轮改造实施文档                                                                                                                                                                                                      
                                                                                                                                                                                                                                    
  **日期**：2026-04-29                                                                                                                                                                                                              
  **生效方式**：MCP server 重启后随 initialize 握手下发新 schema；Web 端随主干发布                                                                                                                                                  

  ## 一、本轮目标

  本轮的主线是**打通 LLM 通过 MCP 写入 Pivot 的完整闭环**（新建 matter / 追加 timeline / 评论），同时针对此前梳理出的 4 个需求延展集中收口：

  0. （主线）新增 `create_matter` 与 `add_comment` 两个写入工具
  1. MCP 支持 @人
  2. MCP 支持跨帖子引用
  3. 正常使用不再强制复制 URL
  4. 已接入客户端的页面排版优化

  ## 二、功能点 & 验证情况

  ### 2.0 ✅ 新增 MCP 写入能力（主线）

  **改造**
  - 新增 `create_matter` 工具：自然语言新建 matter（"在 mcp 下新开一个 matter，标题叫 X，帮我起草内容"）。后端从标题自动生成 matter_id（slug），冲突时附加时间戳后缀。首篇 timeline 文件类型限 think / act（受 planning
  状态校验约束）。
  - 新增 `add_comment` 工具：对已有 timeline 文件追加评论，可附 @-mention 与一键圈人。
  - 配合原有 `create_file`，至此 MCP 写入侧三件套（建 matter / 加 timeline / 加评论）齐备。
  - 三个写入工具的 `owner` 字段统一收口：默认 null，由后端从认证用户自动填充；schema 描述显式禁止 LLM 主动追问用户自己的 pinyin（针对此前用户反馈"被 AI 反问拼音"的体验问题）。
  - 写入工具内置交互协议：调用前必须先把草稿正文 + @人对象呈现给用户确认；如果 matter 当前状态有可用的状态切换（如 planning → executing），LLM 必须主动询问是否同步切换状态，不允许静默挂载或静默跳过。

  **实测**
  - 本次会话用 `create_file` 在「我的夏天」下连发两条 think，分别覆盖：
    - 仅 quote 回复（回复 001 那篇散文，自动定位到 timeline 中目标文件）
    - quote + @人（回复 003 + `mentions=["zhangbo"]`，pinyin 解析与飞书 DM 推送一次通过）

  ### 2.1 ✅ MCP @人

  **改造**
  - `create_matter` / `create_file` / `add_comment` 三个写入工具统一挂载 `mentions` 字段（`targets` + `say`）
  - 联系人新增 pinyin 索引，写入时通过 pypinyin 归一化；旧数据已自动回填
  - @人解析支持：拼音（`zhangbo`）/ 驼峰（`ZhangBo`）/ 带空格（`Zhang Bo`）/ 中文名 / open_id 五种形态
  - 同名歧义时（如 `zhangbo` 命中张菠 / 张博）后端返回 422 + 候选列表，由 LLM 让用户挑选

  **实测**
  - 在「我的夏天」下 `create_file` 同时挂 `mentions=["zhangbo"]`，后端无歧义直接成功，飞书 DM 推送触发

  ### 2.2 ✅ 跨帖子引用

  **改造**
  - `create_file.refer[]` 字段在 schema 中补充完整描述与触发短语
  - 当 `type='verify'` 时，`refer[]` 自动作为跨 matter 白名单：`verifications[].target` 指向其他 matter 的文件，只要该 path 出现在 `refer[]`，后端校验通过；否则 422

  **实测**
  - 集成测试 `case_n5_verify_via_refer_ok` 已覆盖 verify + 跨 matter target 路径

  ### 2.3 ✅ 不再强制复制 URL

  **改造**
  - `list_matters` / `get_matter` / `read_files` / `resolve_context` 四个读取工具补齐中文触发短语
  - 新增 server-level `instructions`，在 MCP `initialize` 握手时随 `InitializeResult` 下发到客户端，让 LLM 在每个 session 的首条消息主动做能力自介绍（5–8 行）+ 示例触发短语
  - 首条消息与 Pivot 无关时（写代码 / 闲聊），跳过完整自介绍，仅在末尾追加一行提示

  **实测**
  - 全程使用自然语言："看看最近的 matter" / "看看我的夏天的回复" / "引用 003 进行回复" 等，未粘贴任何 URL，工具路由正确

  ### 2.4 ✅ 已接入客户端页面排版

  **改造（`/settings/external-ai`）**
  - `McpToolsPanel` 工具列表从 5 个补全到 7 个（补齐 `create_matter` / `add_comment`）
  - 每条工具下方新增 2–3 个可点击复制的示例 pill
  - 顶部新增 `FirstMessagePrompt`：用户接入 CLI 客户端后第一条可直接复制粘贴的提示语
  - AI 引导面板补充：停止控制、后台进度、完成提示
  - NewMatter 引导流：可调整尺寸的预览面板

  ## 三、上线 / 生效说明

  - **后端 schema 改动**：MCP server 重启后生效；已建立的 session 仍持旧 schema，需重连
  - **Web 改动**：随主干发布
  - **联系人 pinyin 索引**：实例首次启动时一次性回填，无需手动迁移

  主要变更：
  - 原"三、已知未覆盖 / 后续"整节删掉，"上线说明"提前为第三节
  - 删掉的那张"comment 读取 / 主动追问引用"表格里的条目，按需求口径它们也不算本轮的 gap
