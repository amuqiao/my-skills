---
name: implementation-contract-plans
description: >-
  Use when the agent needs to organize, create, repair, or review documentation that must separate current implementation facts, external API or capability contracts, and future plans. Trigger for API docs, route/capability contract docs, current architecture docs, roadmap/plan cleanup, code-vs-doc drift checks, or converting scattered notes into maintainable current/contract/plan documentation. Do not use for pure implementation, ordinary bug fixes, generic documentation cleanup, or maintenance unless the task explicitly requires separating implemented behavior, external contracts, and future plans.
version: 0.1.0
---

# Implementation Contract Plans

使用这个 skill 维护三类项目记忆的边界：

```text
current = 当前已经实现并可验证的事实
contract = 外部调用方、使用者或其他模块可以依赖的承诺
plans = 尚未实现但仍值得跟踪的未来工作
```

本 skill 的目标不是强制某个目录结构，而是防止文档把“已实现事实”“对外承诺”和“未来计划”混在一起。优先沿用当前仓库已有的文档命名和组织规则。如果仓库已经使用 `docs/current`、`docs/api`、`docs/plans`，就遵循这些名字；如果仓库使用其他结构，就把三类职责映射到本地结构中。

不要把本 skill 用作普通实现、纯 bug fix、普通 code review、README 美化、prompt 文案清理或 routes 表小修。维护 `pi-agent-harness` / `subagent` harness 时，只有任务明确涉及以下情况才使用本 skill：

- 需要区分“当前 harness 已实现行为”“用户可依赖的 prompt/agent contract”“未来计划”。
- 需要做 code-vs-doc 或 prompt-vs-runtime drift check。
- 需要把散乱 agent notes 转成可维护的 current / contract / plan 文档。

普通 harness 结构说明、中文化、入口说明、路由表高频项维护，优先交给 `document-writing` 或直接处理。

## Workflow

1. 读取本地入口规则。

   先读 `AGENTS.md`、`CLAUDE.md` 或仓库等价入口；如果存在模块级 agent 文件，也读取最接近当前任务的文件。把这些入口视为文档归属、事实源和验证命令的路由规则，而不是历史说明。

2. 分类本次请求。

   使用这组三分法：

   | Kind | Owns | Must not contain |
   | --- | --- | --- |
   | `current` | 已实现架构、模块边界、运行流、数据模型、能力路径、验证基线。 | TODO、未来承诺、历史迁移流水账。 |
   | `contract` | 外部 API 路径、请求/响应 shape、状态/错误语义、callback payload、版本和兼容性规则、prompt/agent 对外可依赖行为。 | 调用方不能依赖的内部运行细节。 |
   | `plans` | 活跃缺口、计划工作、验收标准。 | 已完成事实、陈旧历史、没有 owner 的 speculative design。 |

3. 编辑前寻找事实源。

   API 任务先查 route、schema/type、registry、OpenAPI projection、contract tests。能力或 worker 任务先查 route、worker、service、shared schema、runtime registry、相关测试。harness 任务先查 prompt、routes、manifest、settings、subagent extension docs 或源码。不要让 Markdown 声称代码、prompt 或 runtime 不支持的行为。

4. 更新最小完整集合。

   - Contract change：更新 route/API/prompt/agent contract，并在仓库已有 contract-test 习惯时补测试或检查。
   - Current fact change：更新当前实现文档。只有当模块边界、依赖、命令或文档归属 materially 改变时，才更新最近的 agent 入口文件。
   - Plan change：只在工作未实现时更新计划。
   - Completed plan：把已验收事实移入 current docs，或将 plan 标记/移除为 superseded。

5. 做 drift check。

   按仓库实际存在的表面检查：route docs、route files、operation registries、capability enums、worker/task registries、current capability summaries、tests、OpenAPI snapshots、agent entrypoints、prompt templates、routes.md、manifest.json。没有的表面不要硬造。

6. 验证。

   运行能证明 changed contract 或 current fact 的最窄验证。无法验证时说明具体未验证表面和剩余风险。

## Writing Rules

- 即使同一功能同时涉及三类内容，也要保持 `current`、`contract`、`plans` 分离。
- current docs 用现在时，只写已交付、可验证的行为。
- contract docs 写调用方或用户可依赖的承诺，不暴露不可依赖的内部实现细节。
- plans 写缺口和验收标准，不写成伪 current 文档。
- 避免重复字段表。能力/current 页面应链接到 route/API contract，而不是复制外部字段。
- 保留 shared schema、status、error code、callback、state machine 的单一 canonical location。
- 如果公共 API 文档和 route-specific 文档冲突，只有当仓库明确采用 route-specific 优先策略时才优先后者；否则遵循本地冲突规则。
- 不创建 shadow docs，例如 `temporary API notes`、`supplemental contract`、第二套 API matrix，除非仓库文档地图明确要求。
- 优先短表格、运行路径和归属边界，避免宽泛架构散文。
- 对 `pi-agent-harness` 文档，不要把 prompt 操作规则误写成 runtime guarantee；区分 prompt policy、subagent extension behavior 和用户约定。

## Plan Template

活跃未来计划使用这个结构：

```markdown
# <Plan Name>

## Current Baseline

- What is already implemented and verified.

## Remaining Gaps

- What is still missing or risky.

## Planned Work

- The smallest coherent work items.

## Acceptance

- Observable criteria that allow the plan to be moved into current facts or closed.
```

## Reference Templates

创建或修复文档时，读取 `references/doc-templates.md`，其中包含紧凑模板：

- API route contract
- current capability page
- current runtime flow
- active plan
- drift checklist
