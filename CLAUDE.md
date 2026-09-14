# my-skills 项目约定

## 目录结构

```text
my-skills/
├── skills/            # 个人维护的 skill 真源
│   └── <skill-name>/
│       └── SKILL.md   # 必需：YAML front matter，含 name / description / version
├── AGENTS.md          # 软链接到 CLAUDE.md
└── README.md          # 仓库说明、安装与维护流程
```

## 维护规则

- `skills/` 是唯一真源；不要把 `~/.codex/skills`、`~/.claude/skills` 或 `~/.agents/skills` 中的同名入口当源码维护。
- 新增 skill 使用小写字母、数字和连字符命名，目录名应与 `SKILL.md` 的 `name` 一致。
- 每个 `SKILL.md` 必须包含 `name`、`description`、`version`。
- 安装到本机时使用 `skills-link --source ~/Code/agent-skills/sources/my-skills/skills`，不要手动复制目录。
- 修改后运行 `check_skill_spec.py skills` 和 `skills-doctor --no-project` 做最小验证。

## 当前 skills

| Skill | 职责 |
| --- | --- |
| choose-architecture-pattern | 架构选型和生产落地审查。 |
| document-writing | 长期维护文档写作和结构优化。 |
| implementation-contract-plans | current / contract / plans 文档边界整理。 |
