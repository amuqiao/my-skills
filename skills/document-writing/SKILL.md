---
name: document-writing
description: "文档写作与 Markdown 文档结构优化 skill。Use when Codex needs to create, rewrite, restructure, review, or polish README、索引页、docs index、安装说明、setup guide、使用说明、usage guide、说明文档、explanation docs、方法论文档、architecture notes 或其他长期维护文档，尤其是需要明确读者心智模型、章节顺序、边界、相对链接和文件命名时。不要用于普通聊天回复、简短 final answer、纯代码注释，或已有更具体 skill 负责的 API contract/current/plans 文档，除非仍需要本 skill 处理表达结构。"
version: 0.1.0
---

# Document Writing Skill

使用这个 skill 编写和优化长期维护的项目文档。目标是降低读者理解成本，建立稳定结构，明确文档边界，并保持 Markdown 易于链接、迁移和维护。

## Workflow

1. 识别文档职责。

   判断目标文档是 README、索引页、安装说明、使用说明、解释文档、方法论文档、架构说明还是维护规则。仓库已有清晰文档惯例时，优先沿用。

2. 建立读者路径。

   先明确读者需要建立什么心智模型，再按理解顺序或操作顺序组织章节。长期维护、说明型、架构型、调优型文档，开头应优先设置心智模型章节或等价结构；主流程、可选流程、背景说明、排查内容和维护说明要分层放置。

3. 读取写作规则。

   在新建或大幅重写长期维护文档、调整标题结构、审查文档质量、创建 Markdown 索引、选择文档文件名之前，读取 `references/writing-rules.md`。

4. 编辑最小完整范围。

   只改目标文档和直接相关的链接或索引。除非仓库已有这种模式，不要创建平行的临时 notes 文档。

5. 验证链接和适配性。

   检查相对 Markdown 链接、章节推进、文件命名一致性，以及文档是否仍匹配已实现行为或用户给出的范围。

## Boundaries

- 优先写清结构，不用宽泛叙述填充篇幅。
- 不要把每个普通回复都写成正式文档。
- 当 API contract、current implementation、future plans 已由更具体的 skill 或本地文档地图负责时，不要重复创造同类内容。
- 可选内容必须显式标注，避免读者误以为它是必做路径。

## References

- `references/writing-rules.md`：详细的结构、章节逻辑链、可视化、相对链接和文件命名规则。
