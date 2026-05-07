---
type: think
author: huangshengli
created: '2026-04-24T18:04:22+08:00'
index_state: indexed
---
上线前必须跑一次迁移脚本，把老索引按 matter schema 改写过来。下面是该次迁移的核心决策，供评审。

## 迁移规则

### 只动 INDEX 文件（`index/*-discuss.index.yaml` → `index/*.index.yaml`）

| 项 | 老形态 | 新形态 | 需确认 |
|---|---|---|---|
| 状态 `open` / `pending` | → | `planning` | — |
| 状态 `concluded` | → | **`finished`** | ✅ |
| 状态 `closed` | → | `cancelled` | — |
| 状态 `produced` | → | `finished` | — |
| 文档类型 `proposal` / `reply` / `comment` | → | **`think`**（全部） | ✅ |
| `discussions[0].files[i].path` | → | `timeline[i].file`（补 `discussions/<cat>/<slug>/` 前缀） | — |
| `discussions[0].files[i].summary` | → | `timeline[i].summary`（空值保留） | — |
| `refs[{type: from, path}]` | → | `timeline[i].quote`（单值；多于一条取首条 + 记 warning） | — |
| `refs[{type: refer, path}]` | → | `timeline[i].refer`（保留顺序） | — |
| 帖子 frontmatter `author` | → | `timeline[i].creator` 和 `timeline[i].owner`（双同填） | — |
| 帖子 frontmatter `created` | → | `timeline[i].created_at` | — |
| 顶层 `created` / `last_updated` | → | `matter.created_at` / `matter.updated_at` | — |
| 首文 frontmatter `title` / body H1 / slug | → | `matter.title`（三级 fallback） | — |
| timeline 事件 `X created thread` / `X replied` | → | 丢弃（文件本身即证据） | — |
| timeline 事件 `X mentioned` + `file` + `mention{users, comments}` | → | 目标 item 的 `comments[]`（按 file 精确匹配；定位不到记 warning 丢弃） | — |
| timeline 状态变更事件（中间态） | → | 丢弃历史；只保留终态到 `matter.current_status` | — |

`concluded → finished` 的语义后果：matter 状态机下 finished 只允许追加 `insight`（且触发 reviewed），**被映射为 finished 的历史 matter 不能再做实质推进**。如降成 `planning` 则保留后续推进可能，评审中明确取向即可。

### 边界

- **MD 文件、帖子 frontmatter 全部不动**：产品文档 §八原则"原始 Markdown 是第一手事实"不能破
- **不合成历史 `status_change`**：老 timeline 里的状态变更事件无法定位具体触发文件，强行合成会伪造责任归属
- **幂等**：脚本重跑零副作用（无老格式文件即退出）；若同一 slug 下新老索引共存且内容不一致则**报错停止**，防止误盖
- **单次 git commit**：整个迁移一次提交，committer = `team-pivot-web`，不自动 push（人工确认后再 push）

### 数据库表

**无本次迁移。** `drafts.matter_payload_json` 列在 P4.6 已随应用启动的 `_migrate(conn)` 自动 `ALTER TABLE ADD COLUMN` 加上，不需要独立脚本参与。

## 需要评审确认

| # | 事项 | 建议 | 确认 |
|---|---|---|---|
| 1 | 状态映射 `concluded` → `finished` | 老帖子讨论已达成结论，整个事项按"已正式收口"处理；后续只能追加 `insight` 做复盘 | 接受 / 改回 `planning`（保留后续推进可能） |
| 2 | 文档类型统一 → `think` | `proposal / reply / comment` 是 thread 时代的语义，`think / act / verify / result / insight` 是 matter 时代的语义，两套不是重命名关系。自动推断"老 reply 是行动还是讨论"不可靠，全部落到 `think` 最保守。需要回看老帖子原始类型，MD 文件本次不改、frontmatter 里的 `type` 原字段都在 | 接受 / 要求启发式保留部分区分 |

## 迁移步骤（上线执行）

**前置**：测试阶段迁移脚本已跑通所有集成 case。

```bash
# 1. 停服
sudo systemctl stop team-pivot-web

# 2. 冷备 index 目录（可回滚用）
cp -r /opt/team-pivot-web/var/git/<repo-name>/index \
      /opt/team-pivot-web/var/git/<repo-name>/index.bak.$(date +%F)

# 3. workspace 内的 git 同步到最新
cd /opt/team-pivot-web/var/git/<repo-name>
git fetch && git pull --rebase

# 4. dry-run 确认一次（不加 flag 即默认 dry-run）
cd /opt/team-pivot-web
uv run python scripts/migrate_index_schema.py \
    --workspace var/git/<repo-name> \
    --report var/migration-report.json
cat var/migration-report.json | jq '.warnings, .summary'

# 5. report 无异常后，加 --apply 真执行
uv run python scripts/migrate_index_schema.py \
    --workspace var/git/<repo-name> \
    --apply

# 6. 检查迁移 commit（committer 应为 team-pivot-web）
cd var/git/<repo-name>
git log -1 --format='%h %s (%an / committer: %cn)'

# 7. push 到 remote
git push origin main

# 8. 启服
sudo systemctl start team-pivot-web

# 9. 烟测 matter 接口
curl -s -H "Authorization: Bearer <PAT>" http://127.0.0.1:8000/api/matters | jq '.items[0:3]'
```

**回滚**（迁移后发现问题）：

```bash
sudo systemctl stop team-pivot-web
cd /opt/team-pivot-web/var/git/<repo-name>
git reset --hard HEAD~1                           # 撤掉迁移 commit
# 或直接从冷备还原：
# rm -rf index && mv index.bak.<date> index
sudo systemctl start team-pivot-web
```

---

## 附录：技术细节（供实施参考）

### 老 thread 索引形态

```yaml
origin_path: discussions/<category>/<slug>/
created: <iso>
last_updated: <iso>
discussions:
- path: discussions/<category>/<slug>/
  status: open | concluded | closed | produced | pending
  files:
  - {path: NNN_author_type_hash.md, summary: '', refs: [{type: from|refer, path: ...}]}
timeline:
- {time, event, file?, mention?: {users, comments}}
```

### 新 matter 索引形态

```yaml
version: 1
matter:
  id: <slug>
  title: <人读 title>
  current_status: planning | executing | paused | finished | cancelled | reviewed
  created_at: <iso>
  updated_at: <iso>
timeline:
- file, created_at, creator, owner, type, summary
  quote?, refer?[], verifications?[], outcome?, comments?[], status_change?
```

### 脚本入口

```
uv run python scripts/migrate_index_schema.py --workspace <path> [--apply] [--slug SLUG] [--report <file>]
```

- 不加 flag：dry-run，只读分析、写 report
- `--apply`：落盘 + 单次 git commit（不自动 push）
- `--slug SLUG`：定点单 slug（用于调试边界 case）
- `--report <file>`：默认 `var/migration-report.json`

### 测试策略

`server/tests/test_migrate_index_schema.py`：

- 纯函数 `migrate_one` 单测：典型 proposal + reply / 5 种老 status / mention 转 comment / 多 from-refs 取首 / 孤立 mention 丢弃 / 缺 author 或 created 兜底
- 集成 `apply_migration`（临时目录）：legacy fixtures → dry-run 断言不写磁盘 + report 正确；`--apply` 断言新 yaml 写成 + 老文件删 + git commit 生成 + report 覆盖全部 case；幂等重跑零副作用；冲突场景（新老 index 共存不一致）报错停止

### 验收

- 默认 dry-run 零 error，report 里 warnings 可解释
- `--apply` 后 `index/` 无 `*-discuss.index.yaml`
- 迁移产物 YAML 结构性检查通过（`matter` header 字段齐全、`timeline[].type ∈ {think/act/verify/result/insight}`、必填字段非空）
- `GET /api/matters/{id}` 对迁移后 matter 能返回正确 timeline 和 mention 转出的 comments
- `uv run pytest server/tests/test_migrate_index_schema.py -q` 全绿
