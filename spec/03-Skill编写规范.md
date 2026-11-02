# 03 · Skill 编写规范

Skill 是**按需加载的能力包**：一个文件夹封装一类可复用任务的完整做法。
本规范**不重新定义 Skill 文件格式**，一切格式问题以
[agentskills.io Specification](https://agentskills.io/specification) 为准；
本篇只规定工作区层面的约束力标注。与上游冲突时，**以上游为准**，
并在本篇修订说明中记录差异。

## 1. 与上游标准的硬对齐（Agent Skills）

### 【必须】最小交付单位

一个技能 = 一个目录，目录内至少包含 `SKILL.md`：

```text
<skill-name>/
├── SKILL.md              # 必需
├── scripts/              # 可选：可执行脚本
├── references/           # 可选：长文档，按需再读
└── assets/               # 可选：模板、样例、静态资源
```

- 目录名**必须**等于 frontmatter 的 `name`。
- `name` **必须**为 1–64 字符，仅小写字母、数字、连字符 `-`；
  不得以 `-` 开头或结尾，不得出现连续 `--`。

### 【必须】`SKILL.md` frontmatter

`SKILL.md` **必须**以 YAML frontmatter 开头，且至少包含：

| 字段 | 约束 |
|------|------|
| `name` | 同上；与目录名一致 |
| `description` | 1–1024 字符；写清**做什么**与**何时使用**（供 Agent 匹配） |

【应当】补充：

```yaml
---
name: code-review
description: >
  审查本地 diff 与 PR 的正确性、安全与可维护性。
  在用户要求 code review、检查提交、合并前检查时使用。
  不要用于从零实现新功能。
license: MIT
metadata:
  version: "1.0.0"
  author: your-org
---
```

【可以】使用上游实验字段（如 `compatibility`、`allowed-tools`），
使用前确认当前 CLI 是否识别。

### 【必须】渐进披露（Progressive Disclosure）

1. 会话启动时，Agent **应当**只依赖各技能的 `name` + `description` 做发现
   （量级约百 token 级/技能），**不得**把所有 `SKILL.md` 正文一次性塞进上下文。
2. 仅当任务与某技能的 `description` 匹配，或用户显式点名该技能时，
   再读取该技能的正文。
3. `references/`、`assets/`、`scripts/` **仅在正文步骤要求时**再打开或执行。
4. 【应当】单个 `SKILL.md` 正文控制在上游建议范围内（实践上宜 < 5000 tokens，
   过长拆到 `references/`）。

### 【必须】与 AGENTS.md 的分工

| 内容 | 放哪里 |
|------|--------|
| 人设、全局禁止项、目录约定、完成定义 | `AGENTS.md`（L2） |
| 某一类可复用任务的步骤、检查清单、脚本入口 | `skills/<name>/SKILL.md`（L3） |
| 必须确定性执行的转换/校验 | `scripts/`（L4），由 SKILL 调用 |

【必须】禁止把整本工作流只写在巨型 `AGENTS.md` 而不拆技能。
【必须】禁止在技能正文里写「本智能体的人设」（人设只属于 L2）。

### 【应当】正文推荐结构（非上游强制，本规范推荐）

```markdown
# <技能标题>

## Use When
- …

## Don't Use When
- … → 改用技能 `other-skill`

## Workflow
1. …
2. 如需脚本：运行 `scripts/….py`（说明参数与退出码）

## Rules
- Always …
- Never …（Why: …）
```

### 【必须】工作区内的放置与发现

工作区模板默认路径：

```text
<workspace>/skills/<skill-name>/SKILL.md
```

不同 CLI 的发现目录不一致时，【应当】用下列之一保证可被加载
（在 `AGENTS.md` 中写明本工作区采用哪一种）：

| 策略 | 做法 |
|------|------|
| A. 根目录 skills | 保持 `skills/`，依赖 CLI 支持项目级 skills |
| B. 符号链接 | `.agents/skills/<name> -> ../../skills/<name>` |
| C. Claude Code | 另链 `.claude/skills/<name> -> ../../skills/<name>` |

【必须】同一技能在磁盘上只有一份正文；多路径仅为发现入口，
禁止复制多份后分头修改。

### 【应当】校验

发布或合并技能前，【应当】使用上游或等价校验（若可用）：

```bash
# 示例：官方 skills-ref（以当前工具文档为准）
skills-ref validate ./skills/code-review
```

本仓库后续可增加简易检查：目录名=`name`、存在 description、无连续 `--`。

## 2. 与 AGENTS.md 的加载分工（速查）

| | AGENTS.md | SKILL.md |
|---|---|---|
| 加载时机 | 每次会话开机必读 | 命中任务时按需加载 |
| 内容性质 | 人设、边界、总流程 | 某类任务的操作规程 |
| 判断标准 | "不写这个，agent 每次开机都会跑偏" | "遇到这类任务才需要" |

## 3. 生命周期

1. **沉淀触发**：同一流程被手工指导模型 ≥2 次后，必须沉淀为 skill；
2. **评审**：新技能入库前跑一次真实任务验证 description 命中率；【应当】为关键技能保留正常、拦截、边界三类可复现场景，记录实际输出，不用“已测试”代替证据；
3. **退役**：流程变更时更新 SKILL.md 并注明修订，而不是新建平行技能；
   分发场景的废弃与回滚见 08 篇。

## 4. 反模式

- ❌ description 写成名字复读（"weekly-report：用于周报"——没说何时触发）
- ❌ 把一次性任务包装成技能
- ❌ 同一流程在 AGENTS.md 和 SKILL.md 各写一份
- ❌ 启动时把全部技能正文打进上下文（违反渐进披露）
- ❌ 同一技能复制多份分头修改（多路径只能是入口，正文唯一）
