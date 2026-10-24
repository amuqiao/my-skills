---
name: vanilla-flux-frontend
description: "无框架 Web 前端的工程架构 skill：原生 ES modules + Flux 风格单向数据流（发布订阅 Store）+ 默认转义的 html 模板 + JSDoc strict 类型检查 + design token 视觉体系 + 单元测试与 Playwright e2e 双层测试。Use when building a new web UI without React/Vue/build step, or refactoring / reviewing an existing vanilla-JS front end to this standard (state management, module boundaries, XSS-safe rendering, typing, tokens, tests). Trigger for 按 vanilla-flux 架构写前端、前端架构规范化、原生 JS 前端重构、单向数据流、去掉全局 state、前端质感升级、design token、前端补测试. 不用于已在用 React/Vue/Svelte 等框架的项目、纯后端任务、只改一两行样式的小修，或只问概念不改代码的讨论。"
version: 0.2.0
---

# Vanilla Flux Frontend

用这个 skill 把无框架的 Web 前端写成（或改造成）资深工程师水准：结构清晰、默认安全、有类型、有测试、视觉统一。核心是一个判断：**在小中型工具类前端里，用约 100 行基础设施实现 React/Redux 的思想，比引入框架和构建链更划算。**

一句话架构：原生 ES modules + Flux 风格单向数据流（发布订阅 Store）+ 分层模块 + 声明式 data-* 行为 + design token + 测试金字塔。

## 先判断是否适用

适用：内部工具、管理台、开发者工具、单页控制台；前端约 1k–10k 行；希望无构建、好改、可读。

不适用，直接告诉用户并停下：

- 项目已经在用框架（React/Vue/Svelte/Angular）：遵循该框架的惯例，不要混入本架构。
- 需要 SSR、路由很多、大量复杂表单与状态联动、多人大团队长期演进：建议上框架，说明理由。
- 只是一两处样式或文案修改：直接改，不需要本 skill 的流程。

## Workflow

1. **读事实，定范围。** 先读仓库的 `AGENTS.md` / `CLAUDE.md`、现有前端入口、HTTP 接口文档或路由、现有测试与 CI。改造老项目时，列出：全局可变状态、模块间互相调用渲染的地方、`innerHTML` 拼接处、裸写的样式数值、已有测试。

2. **复述诉求并确认约束。** 明确"功能不变"还是"新增功能"、是否允许新增开发依赖（Playwright）、视觉方向是否沿用现有品牌色。这三点不清楚时先问，最多问一轮。

3. **按顺序推进，每步一个可独立通过测试的提交。**
   1. 测试兜底：先为现有主流程写 e2e（离线、mock 外部服务、不花钱）。读 `references/testing.md`。
   2. 工程底座：core 层、store 单向数据流、`html` 模板、JSDoc strict 类型检查，界面不变。读 `references/architecture.md` 和 `references/core-templates.md`。
   3. 视觉体系：design token、基础组件、布局。读 `references/styling.md`。
   4. 逐块功能换新或新增功能，每块配测试。
   5. 文档：把约定写进项目的架构文档与 `AGENTS.md` / `CLAUDE.md`，让后来的人和 agent 照做。

   新项目从第 2 步开始，测试与功能同步写。

4. **验证后交付。** 类型检查、lint、单元测试、e2e 全过；视觉改动用无头浏览器截图检查亮/暗色和窄/宽屏（截图前等动画与过渡结束，缩放窗口后用新页面重拍，避免误判）。较大的改动交给独立 reviewer 只读审查，按 `references/review-checklist.md` 的维度出 must-fix / should-fix / follow-up，修完再交付。

## 硬性规则

- 共享数据只经 store 的 setter 修改，setter 发事件；面板订阅后自己重绘。面板之间不互相调用渲染函数，不允许循环依赖。
- 界面状态（筛选、选中、页签、展开）留在所属模块，不进全局 store。
- 拼 HTML 只用默认转义的 `html` 模板；`raw()` 只给写死的标记（图标 SVG），绝不给接口或用户数据。
- 不静默删减信息与功能：界面太挤时，加"显示"选项让用户自己选（默认全部显示），而不是删字段、藏面板；要隐藏任何已有信息先问用户。会整块藏掉关键面板的"专注 / 精简模式"先评估再做。
- 时间：存储与接口用 UTC，显示统一经一个格式化模块、用项目约定的固定时区，不用跟随本机时区的 `toLocaleString()` / `getHours()`。
- `$()` 找不到元素就抛错；不写 `?.` 吞掉缺失元素，不加静默兜底。
- 纯逻辑（计算、解析、筛选、格式化）放在顶层不碰 DOM 的模块里，用 `node:test` 单元测试；交互和流程用 e2e。
- 样式只引用 token，不写裸的尺寸和颜色；每个区域最多一个主按钮。
- 不引入框架、不加构建步骤，除非用户明确同意并且第 1 步判断为不适用。

## References

- `references/architecture.md`：分层、单向数据流、事件设计、模块职责划分、声明式 data-* 行为，以及什么时候该升级到框架。
- `references/core-templates.md`：core 层五个模块（dom / api / store / ui / types）与 tsconfig 的可直接落地模板，以及常见陷阱。
- `references/styling.md`：design token 体系、按钮分级与基础组件、应用式布局、暗色模式、可访问性底线、截图验收。
- `references/testing.md`：测试金字塔、无 DOM 纯逻辑模块的单元测试、离线 Playwright e2e（不读真实密钥）、CI 接入。
- `references/review-checklist.md`：审查清单，用于自查或交给独立 reviewer。
- `references/prompt-template.md`：给用户复用的需求 prompt 模板，以及让结果更好的写法。

## Maintenance

更新本 skill 前先读 `MAINTENANCE.md`。只沉淀可迁移的判断和模板，不搬运某个项目的目录、业务名词或一次性排障记录。
