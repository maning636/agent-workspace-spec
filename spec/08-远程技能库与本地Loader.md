# 08 · 远程技能库与本地 Loader

## 1. 目标

在**不绑定具体 Agent 框架**的前提下，做到：

- 技能正文的**唯一真相源**在 Git 远程库（可多库分类）；
- 本地工作区**不长期堆积过期 skill 正文**；
- 更新远程组合或 tag ≈ 更新本地 Agent 可用能力；
- 与 L2（AGENTS.md）、L3（Skill 格式）、L1（任意 CLI）兼容。

本篇是工作区规范的**分发层**；技能文件格式仍遵守 03 篇与 agentskills.io。

## 2. 三种仓库角色

| 角色 | 建议名 | 内容 |
|------|--------|------|
| 工作区 | 业务目录 / agent-workspace | `AGENTS.md`、台账、素材、产出、scripts（业务相关） |
| 技能库 | `skills-dev` / `skills-design` / … | 仅 `skills/<name>/SKILL.md`（+ scripts/references/assets）与 `catalog.yaml` |
| 元库 | `skills-meta` | 全局索引、**composition（组合）**、默认 tag、兼容说明 |

【必须】技能正文不进业务 Git 的长期历史（可 submodule、可启动拉取、可短缓存）。
【应当】组合变更只改 `skills-meta`，避免每个技能库都散落「推荐套装」。

## 3. 技能库内约定

每个技能库根目录【应当】包含：

```text
skills-dev/
├── README.md
├── AGENT.md                 # 给 Loader/人：如何使用本库
├── catalog.yaml             # 机器可读清单
├── skills/
│   └── code-review/
│       └── SKILL.md
└── CHANGELOG.md
```

### catalog.yaml【应当】字段

```yaml
library: skills-dev
schema_version: 1
skills:
  - id: code-review
    path: skills/code-review/SKILL.md
    version: "1.2.0"
    tags: [dev, quality]
    deprecated: false
```

【必须】`path` 指向的目录满足 03 篇与 agentskills.io。
【应当】发版打 git tag：`v1.2.0`；生产组合**钉死 tag**，不默认追 `main`。

## 4. 元库 composition【必须】有可解析定义

```yaml
# skills-meta/compositions/daily-coding.yaml
id: daily-coding
description: 日常写码：审查 + 安全重构 + 测试补全
pin:
  skills-dev: v1.2.0
skills:
  - library: skills-dev
    id: code-review
  - library: skills-dev
    id: refactor-safe
```

本地两台电脑、两个 Agent【应当】各自只引用不同 composition，
或共用同一 composition + 同一组 pin，避免「一台 main、一台旧缓存」。

## 5. 本地 Loader 行为（框架无关）

Loader 可以是 shell、Python 或 Agent 自己的开机步骤；【必须】遵守：

1. **读元信息**：解析 `composition` id → 得到 library pin + skill 列表。
2. **拉取**：按 pin 获取对应 `SKILL.md`（及该次需要的 scripts/references）。
3. **注入**：将「当前启用技能的 name/description 列表」交给会话；
   仅在匹配时再读正文（渐进披露）。
4. **不落盘或短缓存**：
   - 【应当】缓存目录如 `~/.agent-skills-cache/<lib>@<tag>/`；
   - 【必须】提供一键清空；
   - 【必须】禁止把远程 skill 正文提交进业务仓库。
5. **可观测**：启动时打印 `composition`、各 `library@tag`、技能 id 列表。

### 【可以】实现示意（伪代码）

```text
load(composition_id):
  meta = fetch(skills-meta, composition_id)
  for each skill ref in meta:
    text = fetch(repo@tag, skill.path)   # 或 cache hit
    register(name, description, body_lazy=text)
  write_session_manifest(.agent/session-skills.json)  # 可选，勿提交
```

### 与工作区目录的关系

| 模式 | 做法 | 适用 |
|------|------|------|
| A. 纯远程 | 工作区无 `skills/` 正文，仅 Loader 会话内可见 | 「本地不存垃圾」目标 |
| B. submodule | `skills` → 固定 tag 的 submodule | 要可离线、可审阅 diff |
| C. 厂商目录 | 拉取后同步到 `.agents/skills` 或 `.claude/skills` | 必须走 CLI 原生发现时 |

【必须】在工作区 `AGENTS.md` 写明采用 A/B/C 哪一种。

## 6. 更新与回滚

| 动作 | 操作 |
|------|------|
| 升级能力 | 改技能库 → tag → 改 composition 的 pin → 本地清缓存后 load |
| 回滚 | composition 改回旧 pin / 旧 tag |
| 废弃技能 | `catalog.deprecated: true`，composition 移除 id；保留旧 tag 供回滚 |

【必须】禁止在未改 pin 的情况下假设「拉 main 即升级」。
【应当】业务台账中记录：本次任务使用的 `composition@meta-tag`。

## 7. 多 CLI / 双机

- 两台机器【应当】共用同一 `skills-meta` 远程与同一 composition pin。
- 交接【必须】仍走工作区 `产出/` 交接单（见 07 篇）或台账，
  不依赖对方内存中的 skill 缓存。
- 【必须】密钥与客户数据不进公开技能库；私有库与公开库目录分离。

## 8. 安全【应当】

- 远程 skill 中的 `scripts/` 视为可执行代码：首次使用前人工过目或只读加载。
- Loader【应当】拒绝自动 `pip install` / 任意 curl 进技能脚本，
  除非 AGENTS.md 显式授权。
- 公开库 skill 的 `description` 避免诱导越权（与 AGENTS 全局禁止项一致）。

## 9. 与 01–07 的关系

| 篇目 | 关系 |
|------|------|
| 01 总纲 | 本篇属于 L3 的分发方式，不改变五层模型 |
| 03 Skill | 远程文件同样必须满足 03 |
| 06 台账 | 记录 composition 版本，不记录 skill 全文 |
| 07 多 CLI | 调度仍不在本篇；只保证两边读同一 pin |

## 10. 反模式

- ❌ 业务仓库复制多份 skill 后分头改
- ❌ 无 tag、全员追 main
- ❌ 启动时把全部技能正文打进 system prompt
- ❌ 用聊天记录代替 composition 版本记录
- ❌ 一个 composition 塞进无关领域的几十个技能（匹配噪声）
