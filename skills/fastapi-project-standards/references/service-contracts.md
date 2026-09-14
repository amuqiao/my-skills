# FastAPI Service Contracts

本 reference 用于 FastAPI 服务合同的事实源、投影、变更和验证。它不是响应格式模板、接口字段清单、registry 设计文档、Job 平台规范，也不是某个项目的实现快照；它的作用是在不同项目中复用同一套判断方法：哪些承诺应稳定、由谁维护、如何防止漂移。

核心问题不是"这个接口应该返回哪些字段"，而是：

```text
调用方真正依赖的承诺是什么？
它属于协议外壳、业务扩展位、错误语义、操作语义、追踪身份还是跨通道投影？
哪个代码事实源拥有这条承诺？
FastAPI route、OpenAPI、README、SDK 是否只是从事实源投影？
新增或修改它会不会扩大后续必须兼容的合同面？
```

## 合同心智模型

FastAPI 服务合同是调用方可依赖的稳定承诺，不是 route decorator、OpenAPI patch、README 示例或某个 response helper。框架代码只是把合同投影到 HTTP 入口。

合同设计的目标是减少调用方必须猜测的东西。对外只稳定协议边界、业务输入输出、错误判断、身份追踪、幂等和副作用语义；内部实现字段、临时调试信息、供应商原文、框架默认结构和历史兼容残留不应为了"方便"进入合同。

当模型准备新增或修改接口行为时，先把变化放入下列类别之一：

| 类别 | 含义 | 进入合同的条件 |
| --- | --- | --- |
| `protocol shell` | HTTP status、响应外壳、header、request id、server time 等协议层承诺 | 已是项目公开风格，或调用方需要稳定依赖 |
| `business extension` | request schema、response data、Job params/result、callback data 等业务扩展位 | 由具体 operation 或能力拥有，并有明确 schema |
| `error semantics` | 错误码、错误分类、可重试性、public/internal 可见性 | 调用方需要据此分支、重试、告警或排障 |
| `operation semantics` | operation id、调用方边界、幂等键、副作用、成功状态 | 同一操作会被文档、测试、SDK、CLI 或多入口引用 |
| `cross-channel projection` | HTTP、Job 轮询、callback、CLI、外部写回之间的同一语义投影 | 多个通道暴露同一业务结果、错误或状态 |
| `implementation detail` | ORM 字段、内部 DTO、供应商响应、调试字段、临时兼容逻辑 | 不进入合同；留在实现边界内 |

无法分类时，不要新增对外字段、错误字符串、OpenAPI patch 或文档示例。先收束需求：这到底是调用方承诺，还是实现细节。

## 项目既有合同优先

不要把某一种响应格式当成所有 FastAPI 项目的默认答案。项目可能使用统一 envelope、裸 REST response、Problem Details、JSON:API、自定义 header 合同，或只为内部接口保留 FastAPI 默认行为。

审查时先确认：

- 项目已有的公开响应风格是什么。
- 哪些 route 是业务 / 对外 / 跨模块合同，哪些只是 health、metrics、docs、static、download、stream 或内部运维接口。
- 旧调用方是否已经依赖某个字段、status、header、错误结构或 OpenAPI schema。
- 当前任务是在延续既有合同，还是在引入新的合同风格。

只有在项目确实选择统一 envelope 时，才讨论 envelope 字段如何统一定义、投影和测试。不要因为本 reference 提到 envelope，就把裸 REST 或 Problem Details 项目改造成 `code/msg/data` 风格。

## 稳定外壳与业务扩展位

服务合同要先区分稳定外壳和业务扩展位。

稳定外壳负责让调用方识别协议状态、错误分类、追踪身份和通用元信息。它应少而稳定，并由一个事实源维护。新增业务能力时，优先扩展业务位置，不扩展通用外壳。

业务扩展位负责承载具体能力的输入输出：

- 同步接口扩展 request schema 和 response data schema。
- 异步能力扩展 params、public result 或 callback data。
- 外部写回、CLI 或内部服务调用扩展各自通道的业务 payload。

不要把业务字段提升到通用响应顶层、通用 Job 壳、callback 顶层、通用错误对象或 request id / trace 字段中。顶层字段越多，后续 route、OpenAPI、SDK、测试和调用方兼容成本越高。

## 合同事实源

合同必须有代码侧事实源，不能只停留在 Markdown。事实源可以是 Pydantic schema、类型定义、枚举、轻量映射、registry、生成脚本输入、contract tests 或项目等价机制。

选择事实源时看复杂度，不看形式感：

- 少量内部接口可以由 route、schema 和 targeted contract tests 共同承担事实源。
- 公开接口、长期兼容接口、多入口投影、业务包能力、SDK 生成、callback 或 CLI 投影需要更显式的结构化事实源。
- registry 只是复杂项目的一种实现方式，不是默认要求。
- README、OpenAPI、SDK 类型、CLI help 和示例是投影，不应成为第二套事实源。

事实源必须能回答：

- 这条合同由哪个模块拥有。
- 它对调用方是否公开、是否稳定、是否允许破坏性变更。
- route、OpenAPI、文档、测试和 SDK 如何与它对齐。
- 删除、改名、兼容窗口和版本迁移由谁处理。

不要让同一合同在 route decorator、schema、README 示例、OpenAPI patch 和测试 fixture 中各维护一遍。

## Pydantic Schema 边界

Pydantic schema 应表达调用方合同，不只是让 FastAPI 能解析 JSON。

- request schema、response data schema、ORM model、内部 DTO 不要混成一个类。
- 默认拒绝未声明字段；只有明确设计为透传扩展位的对象才允许自由键。
- `null`、省略、空数组、空字符串、默认值和字段排序如果被调用方依赖，就属于合同语义。
- 响应 data schema 只描述业务数据；通用响应外壳由项目统一机制处理。
- 输入校验失败应进入项目选择的统一错误合同；不要让框架默认错误结构意外成为公开合同。
- 内部字段、供应商原文、堆栈、token、签名、调试字段不得因为 `model_dump()`、继承或 OpenAPI 自动推导暴露。

严格 schema 的价值不是复制某个基类，而是让未声明字段、内部字段外泄和 schema 投影漂移快速暴露。

## 操作合同决策门

新增或修改公开 operation 前，先回答这些问题：

- 这是外部调用方、跨模块调用方还是内部实现调用？
- operation id、method / path / channel 是否需要稳定？
- 调用方身份、鉴权、租户或权限边界在哪里进入？
- 成功语义是什么，HTTP status 或通道状态是否表达真实语义？
- request schema、response data schema、错误集合和可重试语义由谁维护？
- 是否存在幂等键、外部副作用、状态迁移、资源创建或费用影响？
- 是否会投影到 OpenAPI、SDK、README、CLI help、callback 或外部写回？
- 新增字段、删除字段、改变 `null` 语义或改变错误分类会不会破坏旧调用方？

只有当 operation 需要被多个投影面长期引用，才需要显式 operation registry 或等价结构。否则用清晰 route、schema 和 contract tests 保持轻量。

## 错误合同

错误合同要让调用方稳定判断失败，而不是让每个 route 临时拼一份错误响应。

错误事实源应回答：

- 哪些错误是 public，哪些只允许内部排障使用。
- 调用方按什么字段判断分类、重试、权限、资源不存在、容量保护或外部依赖失败。
- HTTP status、错误码、reason、message、details 和 retryable 之间的职责边界是什么。
- details 中哪些字段允许对外暴露，哪些只能进入日志或内部排障事实源。
- 未知异常、框架异常、校验异常和外部依赖异常如何投影到 public 错误。

不要把底层库异常类名、数据库错误、供应商错误码、完整第三方响应、堆栈、密钥、token、签名或大 payload 暴露给调用方。需要定位的问题应通过 request id、错误分类和服务端日志关联。

不要为了"兼容"添加 silent catch、空结果假成功、默认错误码吞错或动态补造未声明错误。稳定意味着错误结构稳定、错误分类明确、可测试、可排查，不是错误消失。

## FastAPI 投影边界

FastAPI route、dependency、middleware、exception handler 和 OpenAPI 定制都应服务合同投影，不应各自定义合同。

审查 FastAPI 投影时确认：

- route 返回的对象与 response schema / response data schema 一致。
- middleware 只处理协议外壳、request id、通用 header 等横切边界，不注入业务字段。
- exception handler 覆盖校验错误、业务错误、HTTP exception 和未知异常，并投影到项目选择的错误合同。
- OpenAPI 展示的是实际合同，而不是框架默认结构或手写理想结构。
- health、metrics、docs、static、download、stream、internal debug 等非业务接口有明确例外边界。

如果 OpenAPI 与实际 response 不一致，应修正事实源或投影链路，不要只改文档让它看起来正确。

## Request ID 与追踪身份

request id 是合同与日志之间的共享索引，不是鉴权身份、租户身份、业务 id、幂等键或 job id。

- 入口接受、校验或生成 request id 的规则应有单一 owner。
- 成功响应、错误响应、响应 header 和日志上下文中的 request id 应一致。
- 非法 request id 的处理方式属于错误合同，不应由下游随意清洗或丢弃。
- 后台任务可以保存触发请求 id 或 trace context，但不得把后台对象 id 冒充为 HTTP request id。
- 日志字段细节按 `observability-logging.md` 执行；本文只约束调用方可见的追踪身份语义。

## 跨通道合同

HTTP、Job 轮询、callback、CLI、外部写回和内部服务调用可以有不同协议外壳，但不能对同一业务语义给出互相冲突的承诺。

只有项目已经存在异步 Job、callback、第三方写回或多入口能力时，才引入跨通道合同。不要为了"规范完整"给普通 FastAPI 服务强行设计 Job 或 callback。

跨通道审查重点：

- 哪个状态或结果是权威事实源。
- 轮询、callback、CLI 或外部写回只是同一事实的不同投影，还是各自独立合同。
- 如果两个通道展示的 data 不同，是否声明了从同一 canonical result 或等价事实源派生的映射。
- callback 是终态通知还是业务事实源；轮询是否仍能确认最终状态。
- 投递状态、重试状态、内部恢复状态不应混进调用方业务结果。

这部分只沉淀跨通道一致性思想，不维护 Job 状态机、执行器、lease、重试、恢复或业务结果字段。

## 合同变更审查

以下变化默认按合同变更处理：

- 改名、删除字段，或改变字段类型、枚举、格式、单位。
- 改变 `null`、省略、空数组、空字符串或默认值语义。
- 改变 HTTP status、错误分类、可重试语义、鉴权边界或幂等键。
- 改变 response 外壳、错误外壳、OpenAPI 暴露形态、SDK 类型或调用方示例中的稳定结构。
- 改变跨通道结果、callback data、轮询结果或二者映射。
- 将内部字段、供应商响应、堆栈、token、签名、调试字段或大 payload 暴露给调用方。

兼容变更通常是新增可选字段、补充更具体但同类的错误子码、新增 operation 或新增非破坏性投影。即使看似兼容，也应确认旧调用方不会因为 SDK 生成、默认值、排序、空值、严格解析或缓存策略被破坏。

结论不确定时，默认不要扩大合同面；先保持实现内部化，并把需要调用方依赖的承诺说清楚。

## 验证要求

合同规则必须进入测试或机器检查，不能只依赖文档提醒。根据项目规模选择最小可执行检查：

- 公开 route 的实际响应、status、header 与项目合同风格一致。
- OpenAPI success / error schema 与实际 response 一致，没有框架默认错误结构意外泄漏。
- request schema 拒绝未声明字段，response schema 不暴露内部字段。
- 请求校验错误、业务错误、HTTP exception、未知异常都投影到项目选择的错误合同。
- request id 的合法、缺失、非法路径有覆盖，且响应、header、日志上下文语义一致。
- operation 事实源存在时，route metadata、schema、错误集合和投影文档与事实源对齐。
- 多通道项目覆盖轮询、callback、CLI 或外部写回之间的结果、错误和状态映射。

检查失败时应在项目能控制的最早阶段 fail-fast，例如测试、启动、合并或发布前。不要用运行时默认值、动态注册、OpenAPI patch 或文档补丁掩盖合同漂移。
