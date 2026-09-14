---
name: fastapi-project-standards
description: "FastAPI 项目规范 skill，当前重点覆盖 Pydantic Settings 配置分层、服务合同事实源与投影、安全访问边界、日志出口、request_id、结构化日志和敏感字段边界。Use when creating, reviewing, or refactoring these FastAPI standards. 不用于通用后端教程、纯业务逻辑小修，或未触及配置/合同/安全/日志规范的普通 API、DB、worker 任务。"
---

# FastAPI Project Standards

使用这个 skill 创建、审查或重构 FastAPI 项目规范。当前重点沉淀四个高价值规范面：Pydantic Settings 配置思想、FastAPI 服务合同思想、安全访问边界，以及 FastAPI 日志观测边界。

不要把本 skill 用作通用 Python / 后端 / FastAPI 教程。普通业务逻辑小修、单函数 bug fix、纯算法实现，或只涉及普通 API、数据库、worker 代码但不触及配置、合同、安全或日志规范时，不需要使用本 skill。

## Workflow

1. 先识别任务触及的规范面。

   当前已有 reference 只覆盖配置、服务合同、安全访问边界与日志。其他 FastAPI 工程问题先读仓库事实和项目文档，不要把本 skill 当作完整项目模板。

2. 先读本仓库事实。

   优先读取 `AGENTS.md` 和当前任务直接触及的代码事实：settings、`.env.example`、Pydantic schema、operation / error registry 或等价事实源、认证/授权 dependency、CORS 与安全 middleware、日志初始化、exception handler、route、OpenAPI 定制、worker、合同测试或启动脚本。代码是真相，文档是快照；不要让规范建议脱离现有实现。

3. 按触及面加载 reference。

   - 创建、审查或重构 Pydantic Settings、`.env.example`、env key 映射、派生配置、启动校验、配置机器检查，或判断某个值是否应该新增为配置 key 时，读取 `references/configuration-settings.md`。
   - 创建、审查或重构 FastAPI 业务 / 对外 / 跨模块 route、Pydantic request / response schema、HTTP envelope、错误 envelope、operation / error registry、OpenAPI 投影、异常转换、request id 合同、Job / Callback HTTP 投影或合同测试时，读取 `references/service-contracts.md`。
   - 创建、审查或重构接口暴露面、调用主体、API Key / JWT / Session / 网关身份、认证/授权 dependency、CORS、CSRF、限流/配额、安全错误暴露或敏感日志边界时，读取 `references/security-access-boundary.md`。
   - 创建、审查或重构日志出口、request id 传播、结构化日志事件、敏感日志字段、业务生命周期日志或日志验证时，读取 `references/observability-logging.md`。

4. 保持边界清晰。

   - API route 负责 HTTP 边界、状态码、依赖注入和请求响应 schema。
   - Service 负责业务用例编排，不直接读取环境变量，不持有 HTTP 细节。
   - Repository 或 DAO 负责持久化边界，不把业务决策散落到查询函数里。
   - Settings 负责应用配置语义和启动校验，不被 route、service、repository 反复构造。
   - Worker/job 入口复用同一套应用配置语义，并明确幂等、重试、超时和失败终态。

5. 不添加隐式兜底。

   不要为了"更稳"添加 silent fallback、吞错默认值、旧配置名自动兼容、空结果假成功或自动降级。配置、输入、依赖、权限或合同错误应快速暴露，并给出可定位的错误。

6. 修改规范相关代码时同步验证。

   根据实际改动运行最小必要验证，例如 targeted tests、type/lint、settings 初始化测试、`.env.example` 对齐检查、migration smoke、route contract test、worker/job 幂等测试或启动脚本 smoke。无法验证时说明具体缺口。

## Design Bias

- 优先沿用当前仓库的模块边界、命名、依赖注入和测试习惯。
- 对外合同优先稳定：API 字段、错误语义、状态码、回调 payload、env key、迁移步骤和脚本入口都不应随实现方便随意改变。
- FastAPI async handler 中不要直接调用同步阻塞 I/O；同步 ORM、阻塞 SDK、CPU 密集任务应显式放到合适边界。
- 密钥、token、数据库 URL、cookie 和内部地址不得写入代码、日志、示例、测试 fixture 或文档真实值。
- 健康检查区分 liveness 与 readiness；readiness 才检查数据库、Redis、队列和外部依赖，并设置短超时。

## References

- `references/configuration-settings.md`：FastAPI / Pydantic Settings 配置分层、派生配置与启动校验规则，用于判断哪些值应该暴露成配置，哪些应保持为常量、派生值、脚本变量或废弃拒绝项。
- `references/service-contracts.md`：FastAPI 服务合同、稳定外壳、业务扩展位、schema / registry 事实源、OpenAPI 投影、异常转换和合同测试规则，用于判断调用方可依赖什么以及这些合同由谁维护。
- `references/security-access-boundary.md`：FastAPI 接口暴露面、调用主体、凭证、授权/限流、浏览器边界和错误暴露规则，用于判断谁能访问接口、凭什么访问、能做什么以及失败时暴露什么。
- `references/observability-logging.md`：FastAPI 日志出口、request id 传播、结构化事件、敏感字段边界与日志验证规则，用于判断哪些信息应进入日志，哪些应留在项目已有事实源、指标或追踪系统中。

## Maintenance

更新本 skill 或吸收真实项目经验前，先读取 `MAINTENANCE.md`。维护时只沉淀可迁移的工程判断，不搬运项目配置清单、业务 key、目录事实或一次性排障细节。
