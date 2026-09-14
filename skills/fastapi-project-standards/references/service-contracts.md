# FastAPI Service Contracts

本 reference 用于审查 FastAPI 服务合同：调用方能依赖什么、稳定字段由谁维护、FastAPI route / OpenAPI / 文档只是怎样投影这些事实。它不是接口模板、字段清单、错误码大全或某个项目的 `OperationSpec` 复制品。

服务合同的目标是防止模型或开发者在 route、schema、异常处理、OpenAPI、Job、callback、日志之间各写一套“看起来能用”的协议。合同必须有稳定外壳、业务扩展位、可检查事实源和合同测试。

## 合同心智模型

先把每个对外承诺归到正确层级：

| 合同面 | 事实源 | FastAPI 中的投影 |
| --- | --- | --- |
| HTTP envelope | 公共 envelope schema / response helper / middleware | route 返回值、OpenAPI success schema、contract tests |
| 业务输入输出 | Pydantic request / response data schema | request body、response `data`、SDK / docs 类型 |
| Operation | operation registry 或等价可检查真源 | route decorator、`operation_id`、OpenAPI、接口文档 |
| Error | error registry / application error type | exception handler、error envelope、OpenAPI error responses |
| Request id | 单一入口 middleware 或等价边界 | header、response envelope、error envelope、日志关联字段 |
| Job / Callback | Job / callback schema 与 registry | FastAPI Job route、callback payload、轮询和终态事件文档 |

判断顺序：

1. 这是调用方可依赖的合同，还是内部实现细节？
2. 它属于稳定外壳，还是业务扩展位？
3. 哪个代码事实源拥有它？
4. FastAPI route、OpenAPI、README、SDK 是否只是从事实源投影？
5. 是否有测试或启动校验防止投影漂移？

如果回答不了这些问题，不要直接新增 route 字段、错误字符串、OpenAPI patch 或文档示例。

## 稳定外壳与业务扩展位

稳定外壳只能统一定义一次：

- HTTP success / error envelope。
- 通用错误对象。
- request id / server time 等关联字段。
- 项目已有的分页、资源引用、大产物引用等公共结构。
- 仅在项目已有异步 Job 或 callback 能力时，才包括 JobView / CallbackEnvelope。

业务能力只能扩展自己的位置：

- 同步接口扩展 request schema 和 response `data` schema。
- 异步 Job 扩展 `job_params`、public result、callback data。
- 外部写回或 callback 扩展通道自己的业务 `data`，不扩展 HTTP envelope 顶层。

不要把业务字段提升到 `code`、`msg`、`request_id`、`server_time`、callback 顶层或通用 Job 壳。顶层字段越多，后续每个接口、Job、SDK、OpenAPI patch 和测试都要背负兼容成本。

## Pydantic Schema 事实源

Pydantic schema 应表达合同语义，不只是让 FastAPI 能解析请求。

- 默认拒绝未声明字段；只有明确设计为透传扩展位的对象才允许自由键。
- request schema、response data schema、ORM model、内部 DTO 不要混为同一个类。
- `null`、字段省略、空数组和空字符串的含义必须稳定；改变这些语义是合同变更。
- 响应 schema 描述业务 `data`，不要在每个接口中重复定义 envelope 字段。
- 输入校验失败必须进入统一错误合同，不应泄漏 FastAPI 默认 `detail` 结构，除非项目明确选择裸 FastAPI 合同。
- 隐藏字段、内部字段、供应商原文、调试字段不得因为 `model_dump()` 或 OpenAPI 自动推导意外暴露。

真实项目中严格 schema、统一 envelope 和 OpenAPI contract tests 的价值，不在于复制某个 schema 基类，而在于让未声明字段和投影漂移快速暴露。

## Operation 事实源

当项目只有极少量内部接口时，route + schema + contract tests 可以承担轻量事实源。出现下列复杂度时，应建立 operation registry 或等价可检查真源：

- 公开 HTTP operation 超过一个，并需要稳定 `operation_id`。
- 接口会长期投影到 SDK、CLI help、README 示例、外部文档，或 OpenAPI 需要被调用方稳定依赖。
- 存在业务包、插件式能力、可关闭能力或多入口共享同一操作定义。
- 错误码、幂等、副作用、可观测字段或版本策略需要长期演进。

operation 事实源不必照搬固定字段表，但必须能回答这些维度：

- 谁可以调用，鉴权和调用方身份在哪里进入。
- method / path / channel / `operation_id` 如何保持唯一和稳定。
- 请求 schema、响应 `data` schema、成功状态码和错误集合是什么。
- 幂等键、副作用、状态变化和外部依赖有哪些。
- 需要哪些日志事件、metrics、contract tests 或 OpenAPI 投影。
- 破坏性变更、废弃和版本迁移如何处理。

FastAPI route decorator 可以从 operation 事实源派生，也可以被测试反向校验。不要让 route、OpenAPI、README 示例和 SDK 类型各自维护一份 operation 定义。

## Error Registry 与异常转换

错误合同要让调用方稳定判断失败，而不是让每个接口临时拼错误响应。

错误事实源应至少表达：

- 稳定错误码或 reason。
- HTTP status 或 Job failed 语义。
- 是否可重试。
- 对外 message 和允许暴露的 details 边界。
- public / internal 可见性，以及 internal 错误如何投影到 public 错误。
- 日志中需要保留哪些排障字段。

转换边界：

- 已知业务错误使用 application error 或等价类型，引用已注册错误。
- 请求校验错误转换为统一 error envelope，不泄漏框架默认结构。
- HTTP exception、权限错误、method not allowed、not found 应进入同一错误外壳。
- 未知异常对外投影为内部错误，对内记录堆栈和关联字段。
- 外部依赖错误在 adapter 边界转换，不把供应商错误码、token、签名、完整响应或堆栈暴露给调用方。

不要为了“兼容”添加 silent catch、空结果假成功、默认错误码吞错或动态补造未注册错误。稳定意味着错误结构稳定、可定位、可测试，不是错误消失。

## OpenAPI 与文档投影

OpenAPI、接口文档、README 示例、SDK 类型和 CLI help 都是投影，不是事实源。

FastAPI 项目可以定制 OpenAPI，但定制只能用于反映合同：

- success schema 套统一 envelope，`data` 指向真实 response data schema。
- error responses 指向统一 error envelope。
- request id、鉴权 header、幂等 header 等入口合同与 middleware / dependency 对齐。
- health、metrics、docs、static、stream、download 等非业务接口可以作为明确例外。
- 内部调试、运维或框架生态接口若不承诺给外部调用方，应在项目文档或路由分组中标明边界，不强行套业务合同。

避免两类漂移：

- 实现已经统一 envelope，但 OpenAPI 仍暴露裸 `data` 或默认 `422 detail`。
- OpenAPI patch 手写出一个“理想合同”，但 route、schema、exception handler 没有对应实现。

新增或调整合同后，用 contract tests 或 snapshot 检查 OpenAPI、route metadata、schema 和异常响应是否一致。

## Request ID 合同

`request_id` 同时影响 HTTP 合同、错误响应和日志关联，因此只能有一个 owner。

- 入口接受、校验或生成 request id 的规则必须稳定。
- 成功响应、错误响应和响应 header 中的 request id 应一致。
- 非法 request id 的处理方式必须进入错误合同，而不是让下游随意接受或丢弃。
- 后台 Job 可以保存 `trigger_request_id` 或 trace context，但不得把 `job_id` 冒充为 HTTP `request_id`。
- 日志字段细节按 `observability-logging.md` 执行；本 reference 只约束调用方可见的 request id 合同。

## Job 与 Callback 条件边界

只有项目已经有异步 Job、callback、第三方写回或类似通道时，才引入这部分合同。不要为了“规范完整”给普通 FastAPI CRUD 服务强行设计 Job / CallbackEnvelope。

FastAPI 只负责 HTTP 投影：

- 创建 Job 的 route 校验输入、生成或接收调用方幂等键，并返回统一 HTTP envelope。
- 查询 Job 的 route 返回持久化权威状态视图，不从当前 settings 或执行器状态重新推导历史结果。
- Job 状态机、重试、lease、恢复、投递等不属于 FastAPI route 合同本身。
- callback 是终态事件通知，不替代轮询；callback data 与轮询 public result 如果不同，必须声明二者从同一 canonical result 的映射。
- callback 投递状态应进入轮询视图或内部状态，不塞进 callback 正文本身。

这类合同的重点不是复制某个项目的 Job 字段，而是固定“通用外壳一次定义、业务结果按能力扩展、轮询与 callback 从同一事实源投影”的关系。

## 合同变更审查

以下变更按破坏性合同变更处理，不能当作普通重构：

- 改名、删除字段，或改变字段类型、枚举、格式、单位。
- 改变 `null`、省略、空数组、空字符串的含义。
- 改变 HTTP status、错误码、可重试语义、鉴权边界或幂等键。
- 改变 OpenAPI 暴露形态、SDK 类型或调用方示例中的稳定结构。
- 改变 Job public result、callback data 或二者映射。
- 将内部字段、供应商响应、堆栈、token、签名或调试字段暴露给调用方。

可兼容变更通常是新增可选字段、补充更具体但同类的错误子码、增加新的 operation 或新增日志字段。即使是兼容变更，也应确认旧调用方不会因为默认值、排序、空值或 SDK 生成规则被破坏。

## 验证要求

合同不是只写在文档里。根据项目规模选择最小可执行检查：

- route `operation_id`、method / path、response model、status code 与 operation 事实源一致。
- OpenAPI success / error schema 与实际 response envelope 一致。
- 请求校验错误、业务错误、HTTP exception、未知异常都进入统一 error envelope。
- `X-Request-ID` 或等价 header 的合法、缺失、非法路径有测试。
- 错误 reason / code 不能引用未注册项；public operation 不能暴露 internal error。
- Pydantic schema 不接受未声明字段，不暴露内部字段。
- Job / callback 项目覆盖创建、查询、成功终态、失败终态、callback data 与 public result 映射。

检查失败时应在项目能控制的最早阶段 fail-fast，例如启动、测试、合并或发布前。不要用运行时默认值、动态注册或文档补丁掩盖合同漂移。
