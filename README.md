# my-skills

> 个人常用 Agent Skill 源码仓库，用来统一维护 Codex / Claude 等 Agent 可复用的工作规则。

本仓库采用和 [`wangruofeng/meta-skill`](https://github.com/wangruofeng/meta-skill) 类似的结构：`skills/` 是唯一源码层，`~/.agents/skills` 是本机全局 Store，其他 Agent 目录只保留软链接入口。

## 核心模型

```text
my-skills/skills/<skill-name>      # 源码真源，提交到 git
        |
        | skills-link
        v
~/.agents/skills/<skill-name>      # 全局 Store，软链接汇聚层
        |
        | Agent 加载或逐个软链接
        v
~/.codex/skills / ~/.claude/skills # Agent 入口
```

日常只维护本仓库里的 `skills/`。不要把 `~/.codex/skills` 或 `~/.claude/skills` 里的同名目录当真源修改。

## Skills

| Skill | 用途 |
| --- | --- |
| [choose-architecture-pattern](skills/choose-architecture-pattern/SKILL.md) | 架构选型、设计模式、状态管理、可靠性与生产落地审查。 |
| [document-writing](skills/document-writing/SKILL.md) | README、安装说明、使用说明、架构说明等长期维护文档写作和结构优化。 |
| [implementation-contract-plans](skills/implementation-contract-plans/SKILL.md) | 区分 current / contract / plans，整理实现事实、外部契约和未来计划。 |

## 安装到本机 Store

首次 clone 后，把本仓库的 skills 链接到 `~/.agents/skills`：

```bash
git clone git@github.com:amuqiao/my-skills.git ~/Code/agent-skills/sources/my-skills
skills-link --source ~/Code/agent-skills/sources/my-skills/skills
skills-doctor --no-project
```

如果之前已经复制安装过同名 skill，先预览，再确认覆盖为软链接：

```bash
skills-link --source ~/Code/agent-skills/sources/my-skills/skills --dry-run
skills-link --source ~/Code/agent-skills/sources/my-skills/skills --force
skills-doctor --clean-backups --no-project
```

## Agent 入口

Codex 入口建议做成逐个软链接：

```text
~/.codex/skills/choose-architecture-pattern -> ../../.agents/skills/choose-architecture-pattern
~/.codex/skills/document-writing -> ../../.agents/skills/document-writing
~/.codex/skills/implementation-contract-plans -> ../../.agents/skills/implementation-contract-plans
```

Claude Code 如需使用同一份 skill，也应链接到 `~/.agents/skills`，避免维护第二份副本。

## 日常维护

更新本仓库后：

```bash
git status
python3 ~/Code/agent-skills/manager/skillctl/skills/rf-skill-doctor/scripts/check_skill_spec.py skills
skills-doctor --no-project
```

新增 skill 时放到：

```text
skills/<skill-name>/SKILL.md
```

`SKILL.md` front matter 至少包含：

```yaml
---
name: skill-name
description: 简短说明触发场景和用途
version: 0.1.0
---
```

## 更新来源

这 3 个 skill 最初来自本机 `~/.codex/skills`，现在已迁移为本仓库真源，并通过 `~/.agents/skills` 汇聚给各 Agent 使用。
