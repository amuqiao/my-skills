---
name: fastapi-project-standards
description: "FastAPI 项目规范 skill。Use when creating, reviewing, or refactoring FastAPI projects, especially Pydantic Settings 配置分层、派生配置、启动校验、API contracts, SQLAlchemy/Alembic, async boundaries, workers, testing, security, observability, and deployment health checks. 不用于非 FastAPI 项目或纯业务逻辑小修。"
---

# FastAPI Project Standards

使用这个 skill 创建、审查或重构 FastAPI 项目规范。目标是让项目结构、配置、API 合同、数据库、异步边界、任务队列、安全、测试和运维入口保持清晰、可验证、可演进。

不要把本 skill 用作通用 Python 教程。普通业务逻辑小修、单函数 bug fix、纯算法实现，如果不涉及 FastAPI 项目边界或工程规范，不需要使用本 skill。

## Workflow

1. 先识别任务触及的规范面。

   常见规范面包括项目结构、API route 与 schema、dependency injection、配置与 secrets、数据库与迁移、async/sync 边界、worker/job、错误处理、安全、日志与 request id、测试、健康检查和部署入口。

2. 先读本仓库事实。

   优先读取 `AGENTS.md`、已有目录结构、route、schema、settings、数据库 session、migration、worker、测试和启动脚本。代码是真相，文档是快照；不要让规范建议脱离现有实现。

3. 按触及面加载 reference。

   - 创建、审查或重构 Pydantic Settings、`.env.example`、env key 映射、派生配置、启动校验、配置机器检查，或判断某个值是否应该新增为配置 key 时，读取 `references/configuration-settings.md`。

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
