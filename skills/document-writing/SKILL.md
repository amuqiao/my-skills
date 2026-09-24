---
name: document-writing
description: "Repository documentation and Markdown structure skill. Use this skill whenever a pi coding agent is asked to create, rewrite, restructure, review, or polish long-lived repo documentation: README.md, docs/*.md, docs index/navigation pages, setup/install guides, usage guides, troubleshooting guides, contributor docs, architecture notes, method docs, migration notes, or other maintainable Markdown. Trigger for Chinese or casual prompts such as 写文档、改 README、整理 docs、补说明、优化说明、写使用指南、补安装步骤、检查 Markdown 链接、让文档更清楚, even when the user does not explicitly say 'documentation skill'. Use when the task needs reader-path design, section ordering, scope boundaries, stable file naming, relative links, or a brief restatement of the user's documentation intent before editing. Do not use for ordinary chat replies, short final answers, code comments/docstrings, transient planning notes, PR descriptions, .docx/.pdf deliverables, or specialized current/contract/roadmap docs handled by implementation-contract-plans unless the task explicitly needs Markdown writing structure."
version: 0.2.0
---

# Document Writing Skill

使用这个 skill 编写、重构、审查和维护仓库内的长期 Markdown 文档。目标是降低读者理解成本，建立稳定结构，明确文档边界，并保持文档易于链接、迁移和后续维护。

本 skill 只处理文档表达、结构、导航、链接和文件组织；不改变代码行为、配置语义、prompt runtime、agent/harness 调度策略或产品承诺。

## Use this skill for

当任务涉及仓库内长期维护文档时使用，包括但不限于：

- `README.md`、`docs/*.md`、文档索引页、导航页、安装说明、使用说明、排查说明、贡献指南、架构说明、方法论文档、迁移说明。
- 用户用中文或口语化表达：写文档、改 README、整理 docs、补说明、优化说明、写使用指南、补安装步骤、检查 Markdown 链接、让文档更清楚。
- 需要先理解并简短复述用户的文档目标，再决定读者路径、章节顺序、边界、相对链接和文件命名。
- 文档会被后续读者反复查阅、维护或引用，而不是一次性聊天回复。

## Do not use this skill for

- 普通聊天回复、短 final answer、临时总结、PR 描述、代码注释或 docstring。
- 主要交付物是 `.docx`、`.pdf`、slides、spreadsheet 或在线协作文档的任务；这类任务应使用对应的更具体 skill。
- 专门区分 current implementation、external contract、future plans 的文档治理任务；优先使用 `implementation-contract-plans`，除非用户还明确要求优化 Markdown 表达结构。
- 改变代码、配置、API 行为、agent 路由、trust policy、workflow 语义或 runtime 规则。

## Workflow

1. **确认文档职责。**

   判断目标是 README、索引页、安装说明、使用说明、解释文档、架构说明、排查说明、贡献指南、方法论文档还是维护规则。仓库已有清晰文档惯例时，优先沿用。

2. **理解并复述诉求。**

   对新建、大幅重写或需求含糊的文档任务，先用 1–3 句话复述用户要解决的文档问题、目标读者和交付边界。小范围修补可以直接执行，但仍要保持边界清楚。

3. **收集最小事实。**

   编辑前读取目标文档、相邻索引、相关 README，以及必要的代码或配置事实。不要凭空补产品能力、安装步骤、API 行为或未来计划；无法确认时标注为待确认，而不是写成既成事实。

4. **建立读者路径。**

   先明确读者需要建立什么心智模型，再按理解顺序或操作顺序组织章节。长期维护、说明型、架构型、调优型文档，开头优先设置心智模型章节或等价结构；主流程、可选流程、背景说明、排查内容和维护说明要分层放置。

5. **按需读取写作规则。**

   在新建或大幅重写长期维护文档、调整标题结构、审查文档质量、创建 Markdown 索引、选择文档文件名之前，读取 `references/writing-rules.md`。

6. **编辑最小完整范围。**

   只改目标文档和直接相关的链接或索引。不要为了“更完整”创建平行临时 notes、重复说明页或未被链接的孤岛文档；除非仓库已有这种模式或用户明确要求。

7. **验证链接和适配性。**

   检查相对 Markdown 链接、章节推进、文件命名一致性，以及文档是否仍匹配已实现行为或用户给出的范围。修改代码库文档后，运行最小必要验证；无法运行时说明替代检查和剩余风险。

## Writing principles

- 优先写清结构，不用宽泛叙述填充篇幅。
- 先帮助读者建立整体理解，再展开步骤、细节和排查。
- 章节不是信息容器，而是读者理解路径上的节点；每个一级章节应承担明确职责。
- 主流程、可选流程、补充说明、FAQ 和排查内容要分层放置，不要混排。
- 可选内容必须显式标注，避免读者误以为它是必做路径。
- 仓库内 Markdown 链接默认使用相对路径；文件名应稳定、可预测、便于链接。
- 保持文档与事实一致；不要擅自增加未实现能力、路线图承诺或隐含兼容保证。

## Project-specific boundary

处理 `pi-agent-harness` 或 pi agent 相关文档时，只优化 README、说明结构、导航、相对链接、章节顺序和边界说明；不要擅自新增或改变 workflow、agent 路由、trust policy、subagent 参数规则或 harness runtime 语义。

## Quality checklist

完成前快速检查：

- 文档职责、目标读者和不负责范围是否清楚。
- 章节顺序是否符合读者理解或操作路径。
- 是否缺少关键前置概念、安装条件、命令上下文或验证步骤。
- 主流程和可选流程是否分层清楚。
- 相对链接、标题层级和文件命名是否一致。
- 内容是否只描述已确认事实或用户明确给出的计划。

## References

- `references/writing-rules.md`：详细的结构、章节逻辑链、可视化、相对链接和文件命名规则。
