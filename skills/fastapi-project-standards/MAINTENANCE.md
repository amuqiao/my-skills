# FastAPI Project Standards Maintenance

本文是 `fastapi-project-standards` 的维护准则。它负责约束以后如何把真实项目经验沉淀进 skill，避免把 skill 写成配置清单、项目模板或排障备忘录。

## 维护目标

这个 skill 要沉淀的是 FastAPI 项目规范中的高维工程判断，而不是某个项目的实现快照。

维护时优先回答：

```text
这条经验能否帮助后续使用者在新项目里做更好的判断？
它描述的是稳定原则，还是某个项目的配置事实？
它会不会扩大 skill 的触发范围、上下文负担或误导未来实现？
```

如果一条经验只能说明某个项目当前怎么配、有哪些 key、用了什么端口、接了哪个业务供应商，它不应该进入本 skill。它最多可以作为校准样本，帮助提炼更高一层的判断规则。

## 可吸收内容

适合吸收的是会改变工程判断的规则、边界和问题：

- 项目结构、API 合同、Settings、数据库、worker、测试、可观测性、安全和部署健康检查的职责边界。
- 判断某个值是否应该暴露为配置、保持为派生值、留作内部常量或归入脚本变量的方法。
- 防止 silent fallback、旧 key 兼容、空结果假成功、隐式降级的约束。
- 启动校验、机器检查、合同测试、漂移检查这类能防止规范失效的验证要求。
- 真实项目反复证明有价值的抽象模式，例如"先分类再暴露配置"、"派生值禁止进入 env"、"脚本变量不进入应用 Settings"。

好的沉淀方式是把经验改写成判断句、边界规则或审查问题，而不是复制项目实现。

## 不吸收内容

以下内容默认不进入本 skill：

- 某个项目的完整 `.env.example`、配置 key 列表、默认端口、模型名、bucket、业务包名或供应商账号结构。
- 只服务单个项目的一次性排障记录、迁移历史、临时兼容方案或历史遗留解释。
- 具体业务 schema、业务 job_type、业务目录事实、业务阈值和业务开关。
- 已由更具体 skill 负责的通用能力，例如脚本入口合同、文档写作规则、pipeline stage contract。
- 大模型已经能自然完成的通用 FastAPI 教程内容。

如果确实需要保留项目事实，应放回对应项目文档，而不是放进这个可复用 skill。

## 抽象过滤

从真实项目吸收经验时，先做四步过滤：

1. 去项目化。

   删除项目名、业务名、具体 key、端口、bucket、模型、供应商实例和目录快照。

2. 找判断变量。

   识别真正影响决策的变量，例如外部意图、派生关系、安全边界、生命周期顺序、失败代价、调用方合同、维护 owner。

3. 改写成规则。

   用"什么时候应该"、"什么时候不应该"、"必须先判断什么"、"违反后会产生什么风险"表达，不用"某项目这样做"表达。

4. 放到正确层级。

   短的全局判断放 `SKILL.md`；某个规范面的大段规则放 `references/`；只在维护 skill 时需要的信息放本文件。

## Reference 维护边界

`SKILL.md` 是触发入口和路由器，只保留适用场景、基本边界和按需读取 reference 的规则。不要把完整规范塞进 `SKILL.md`。

`references/` 是运行时按需读取的详细规则。每个 reference 应围绕一个稳定规范面，例如配置、服务合同、数据库迁移、worker/job、测试或可观测性。reference 应表达判断框架，而不是实现清单。

当前 skill 只实际维护配置、服务合同、安全访问边界和日志四个 reference。不要因为名称是 `fastapi-project-standards`，就提前补齐一套完整后端规范库；新增 reference 必须先证明该规范面有可复用的高维判断，而不是普通 FastAPI 教程。

维护者文档只放在本文件。不要把维护说明写进普通 reference，避免普通任务加载与当前实现无关的维护过程。

## 配置规则的标尺

`references/configuration-settings.md` 是本 skill 的标尺案例。它要表达的是：

```text
FastAPI Pydantic Settings 配置分层、派生配置与启动校验规则。
```

维护它时，重点防止：

- 把"代码需要一个值"直接变成"新增一个 env key"。
- 把派生值、内部 buffer、实现阈值暴露给 `.env.example`。
- 把脚本参数、运行形态变量、SDK 自动读取变量混进应用 Settings。
- 为旧 key 添加 silent fallback，而不是删除或拒绝。
- 只改配置代码，不补启动校验、机器检查或测试。

不要把真实项目配置清单搬进这个 reference。真实项目只能提供判断校准，最终进入 reference 的应是可迁移的配置思想。

## 服务合同规则的标尺

`references/service-contracts.md` 要表达的是：

```text
FastAPI 服务合同、稳定外壳、业务扩展位、注册事实源、OpenAPI 投影与合同测试规则。
```

维护它时，重点防止：

- 把"新增一个接口"直接变成"route 里临时拼一份响应"。
- 在每个接口、Job、callback 或 OpenAPI patch 中复制 envelope 顶层字段。
- 把 OpenAPI、README 示例或 SDK 类型当作事实源，而不是 schema、registry 或等价可检查真源的投影。
- 使用自由字符串错误码、未注册异常 reason、临时 HTTP status 或泄漏内部错误。
- 把业务字段提升到服务顶层、Job 通用壳或 callback 顶层。
- 混用 HTTP、Job、callback、CLI 或外部写回合同，导致调用方无法稳定判断。
- 只改 route 或文档，不补 schema、registry / 等价真源、异常转换和 contract tests。

不要把真实项目的 `OperationSpec` 字段清单、错误码表、业务 operation id、endpoint 列表、schema catalog、测试命令清单或业务包目录事实搬进这个 reference。真实项目只能用来校准哪些合同维度容易漂移，最终进入 reference 的应是可迁移的合同思想。

## 安全访问边界规则的标尺

`references/security-access-boundary.md` 要表达的是：

```text
FastAPI 接口暴露面、调用主体、凭证、授权/限流、浏览器边界与错误暴露规则。
```

维护它时，重点防止：

- 把"给 route 加鉴权"直接变成"随手加一个 Depends"。
- 新增业务 / 对外 / 跨模块 route 时默认放行，而不是要求访问级别声明或公开例外 allowlist。
- 把 CORS、HTTPS、网关、API Key、JWT、Session、mTLS、限流和授权混成同一个安全概念。
- 把 API Key、JWT secret、callback secret、Cookie、signed URL 或内部地址写进代码、前端产物、日志、示例或测试真实值。
- 认证通过后跳过资源归属、租户、角色、配额、成本或副作用授权。
- 把租户、角色或资源归属授权塞进 middleware，靠调用方自报字段判断，而不是读取服务端事实源。
- 混淆 401、403、429、404 和 5xx，导致调用方无法判断认证、授权、容量和系统失败。
- 把完整 token、request body、供应商错误、数据库状态、权限规则或内部拓扑暴露到错误响应或日志。
- 只改安全代码，不补失败路径测试、OpenAPI security 投影、日志脱敏或网关绕过检查。

不要把真实项目的密钥名称、JWT claim 表、权限角色表、租户模型、CORS origin 列表、网关部署拓扑、限流阈值或安全测试命令搬进这个 reference。真实项目只能用来校准哪些访问边界容易被模型混淆，最终进入 reference 的应是可迁移的安全判断框架。

## 日志规则的标尺

`references/observability-logging.md` 要表达的是：

```text
FastAPI 日志出口、request id 传播、结构化事件、敏感字段边界与日志验证规则。
```

维护它时，重点防止：

- 把"想看更多信息"直接变成"打印完整 payload"。
- 在应用代码里默认新增文件日志，而不是保持 stdout/stderr 出口边界。
- 在 route、service、worker 或 callback 中生成多套 request id。
- 随手写自由字符串 event，而不是使用 registry、枚举、白名单或项目等价机制。
- 把日志当成业务状态、恢复、成本、callback 或审计的事实源。
- 只改日志代码，不补事件白名单、敏感字段边界、request id 或合同测试。

不要把真实项目的日志事件枚举、业务字段表、本地日志路径或排障命令搬进这个 reference。真实项目只能提供判断校准，最终进入 reference 的应是可迁移的日志思想。它也不应扩展成通用 observability skill；除非另有明确目标，metrics、trace、dashboard 和告警只作为日志边界的相邻概念出现。

## 更新流程

更新本 skill 时按最小闭环执行：

1. 读本文件、`SKILL.md` 和目标 reference。
2. 判断新增经验属于高维规则、运行时 reference，还是项目事实。
3. 只更新最小必要文件。
4. 检查新增内容是否会导致误触发、上下文膨胀或项目清单化。
5. 运行 skill validator。
6. 在回复中说明吸收了什么判断、没有吸收什么项目事实，以及验证结果。

## 审查问题

提交或结束前用这些问题自检：

- 新增内容是否仍然面向 FastAPI 项目规范，而不是单个项目？
- 是否保留了用户真正想沉淀的高维概念？
- 是否避免了配置清单、业务 key、目录快照和临时排障记录？
- 是否给后续使用者增加了新的判断能力，而不是重复常识？
- 是否保留渐进式披露：入口短，细节按需读？
- 是否明确了不该做什么，尤其是不要 silent fallback 和不要扩大配置面？
- validator 是否通过？
