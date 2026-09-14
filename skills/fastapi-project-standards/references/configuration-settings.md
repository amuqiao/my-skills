# FastAPI Configuration Settings

本 reference 用于 FastAPI / Pydantic Settings 的配置分层、派生配置与启动校验。它不是配置清单，也不是某个项目的 env key 模板；它的作用是让 Codex 在创建、审查或重构配置时先判断配置语义，避免随手新增 key、暴露派生值、保留废弃配置或把脚本变量混进应用 Settings。

核心问题不是"代码需要一个值，应该叫什么 env key"，而是：

```text
这个值是否真的表达外部部署、安全或业务意图？
它是否应该由现有主控配置和工程余量派生？
它是否只是实现细节、脚本参数、SDK 自动读取变量或旧配置残留？
新增它会不会扩大使用者必须理解和维护的配置面？
```

## 配置心智模型

FastAPI 配置是应用进程的稳定合同，不是变量垃圾桶。Pydantic Settings 负责把少量外部意图读入进程，在 Settings 内集中派生实现参数，并在启动阶段完成统一校验。

配置设计的目标是减少使用者必须理解的变量数量。对外只暴露能表达部署、安全或业务承诺的主控变量；内部实现参数、联动计算结果、历史残留和一次性调试开关不应为了"更灵活"进入 `.env.example`。

当模型准备新增配置时，必须先把这个值放入下列四类之一：

| 类别 | 含义 | 是否进入 `.env.example` |
| --- | --- | --- |
| `env-driven` | 随环境变化，表达部署、安全、租户、外部依赖或业务承诺 | 必需项和稳定可调项可以进入 |
| `tunable constants` | 工程余量，例如比例、倍数、buffer、margin、窗口大小，有代码默认值 | 默认不进入；确需外部调优时才进入 |
| `derived` | 由 `env-driven` 和常量旋钮计算出的最终实现值 | 禁止进入 |
| `non-application env` | 脚本参数、运行形态变量、SDK/平台自动读取变量 | 不进入应用 Settings；按脚本或运行环境规则维护 |

无法分类时，不要新增 env key。先收束需求：这个值到底是外部意图、工程余量、派生结果，还是不该由应用配置负责。

## Settings 入口

FastAPI 服务应使用 Pydantic Settings 作为集中配置入口。Pydantic v2 项目使用 `pydantic-settings` 的 `BaseSettings`；迁移旧版本时先确认字段校验、模型校验和配置源 API 的等价写法。

Settings 入口必须满足：

- 所有应用运行时配置从同一个 settings 对象读取。
- API、Worker、Beat 和脚本任务复用同一套应用配置语义。
- `.env.example` 定义应用配置 key 集合、必需项和语义模板。
- 密钥和环境差异通过进程外配置注入，不写入代码、镜像或前端产物。
- 运行形态变量、SDK 自动读取变量和脚本私有变量与应用 Settings 字段分开维护。

Settings 对象应在进程启动时初始化一次。初始化或校验失败必须阻止 API、Worker、Beat 和脚本任务继续启动；不要让入口先监听、接单或执行副作用，再在业务路径里发现配置错误。

业务代码应通过 FastAPI dependency、应用级单例或显式参数接收 Settings。不要在每个请求、每个任务、Service、Repository 或工具函数中重新读取 `.env`、重新构造 Settings，或直接访问 `os.environ` 取得应用配置。

## 子对象分层

当服务包含数据库、队列、AI 供应商、对象存储、callback 或 Job 配置时，应把 Settings 拆成职责明确的子对象，而不是把所有字段平铺到根 Settings。

推荐结构按职责命名，而不是按文件数量命名：

```text
AppSettings
  -> DatabaseSettings
  -> RedisSettings / BrokerSettings
  -> JobSettings
  -> AIProviderSettings
  -> StorageSettings
  -> CallbackSettings
```

根 Settings 负责聚合子配置、读取统一配置源和校验跨子对象不变量。子对象负责本领域字段的类型、默认值、敏感字段保护和领域内校验。

子对象之间不得互相读取环境变量或重新构造 Settings。跨对象派生关系只能在根 Settings 的统一校验阶段完成。不要为了拆分文件制造第二套配置事实源；`.env.example`、字段到 env key 的映射、废弃键拒绝和机器检查仍应按同一套应用配置语义维护。

## 新增配置决策门

新增任何 Settings 字段、env key 或 `.env.example` 项目前，先回答这些问题：

- 这个值表达的是外部部署、安全或业务意图，还是内部实现细节？
- 使用者是否需要在不同环境中独立改变它？
- 它是否能从已有主控变量加固定 buffer、ratio、margin 或窗口派生？
- 它是否与其他配置有顺序、容量、超时、生命周期或安全不变量？
- 它是否属于脚本入口、Compose/K8s 运行形态、SDK 自动读取变量，而不是应用 Settings？
- 它是否是旧 key、废弃 key、临时调试 key 或一次性迁移 key？
- 新增它后，`.env.example`、Settings 字段映射、启动校验、机器检查和测试是否都有唯一 owner？

只有当答案证明它是稳定外部意图，才把它作为 `env-driven` 配置暴露。否则优先保持为代码常量、派生属性、脚本参数、允许清单项，或直接删除/拒绝。

## 运行时配置

运行时配置只表达环境差异和业务意图，不表达内部实现链路。适合进入 `env-driven` 的通常是：

- 数据库、Redis、对象存储、外部服务 base URL。
- API key、JWT secret、callback secret 等敏感配置。
- 业务意图型主控变量，例如调用方愿意等待的最长时间、系统接收的最大并发/积压、最大输入大小。
- 功能开关、环境类型标志、AB 测试标记等随部署环境变化的开关。

不要让调用方同时配置一组强依赖实现变量。例如不要同时暴露主 timeout、执行 timeout、领取窗口和僵死扫描阈值；应只暴露表达业务承诺的主控变量，其余值由 Settings 按固定关系派生。

敏感运行时配置字段必须避免序列化泄漏。使用 `repr=False`、Secret 类型、序列化 `exclude` 或等价方式，确保密钥、token、数据库 URL 不出现在 settings 对象字符串表示、`model_dump()`、日志输出或错误上报中。`.env.example`、README、部署说明和测试模板只能放不可用占位符。

## 常量旋钮

常量旋钮只用于数值型工程余量，例如比例、倍数、buffer、margin、窗口大小。它们服务派生逻辑，降低最终配置数量，不应被设计成另一组必填运行时配置。

常量旋钮必须满足：

- 有代码默认值。
- 名称体现比例、倍数、buffer、margin 或窗口语义，不命名为最终计算值。
- 注释或文档说明量纲、合理范围和影响对象。
- 默认不进入 `.env.example`；确需外部调优时，必须说明它影响性能、成本、安全还是恢复速度。

功能开关、环境类型标志、AB 测试标记不属于常量旋钮。如果它们需要按环境变化，应归入运行时配置，并显式进入 `.env.example`。

优先使用"主控基数 + 工程余量"的设计，让用户只理解主控决策，让工程余量留在代码默认值或高级可选配置中。

## 派生配置

存在联动关系的最终值必须集中派生，不让用户分别配置。推荐流水线：

```text
读取 env-driven 输入
  -> 合并 tunable constants
  -> 统一派生 derived
  -> 校验最终 settings 不变量
```

典型派生关系：

- 业务主 timeout 派生执行层 soft/hard timeout。
- 单 worker 并发数和接单缓冲倍数派生积压上限。
- callback 单次超时和领取窗口 buffer 派生最终领取窗口。
- stale buffer 派生僵死扫描阈值。

派生配置必须遵守覆盖禁区：

- 派生字段禁止被 env 单独覆盖。
- 派生字段禁止进入 `.env`、`.env.example` 或项目维护的配置模板。
- 如果某个派生值确实需要独立控制，必须把它提升为运行时主控变量，并重新整理其他派生关系。

派生配置优先使用 `@property` 或 Pydantic `@computed_field`，保证每次访问都从当前主控变量重新计算。使用 `@computed_field` 时必须检查 `model_dump()`、schema、日志和调试输出的暴露面；包含内部阈值、供应商参数或敏感派生结果的字段应显式排除。若因性能原因使用初始化后字段，必须将 Settings 整体设为不可变，例如 `ConfigDict(frozen=True)`，避免主控变量修改后派生值失效。

不要出现"先派生、再允许外部覆盖"的混合模型。混合模型会让代码里的派生逻辑和运行时实际值不一致，排查时无法判断哪个事实源有效。

## `.env.example` 边界

`.env.example` 是应用配置 key 集合、必需项和语义模板的单向真源。它不是所有进程环境变量的全集，也不是把每个可调实现参数都展示给用户的地方。

规则：

- `.env.example` 只保存不可用占位符和语义说明，不保存真实密钥、真实连接串或可调用凭证。
- 新增、改名或删除应用配置 key 时，先更新 `.env.example`、Settings 字段映射和配置机器检查，再同步各类环境模板。
- 只出现在 `.env`、`.env.*` 或部署模板中，但没有出现在 `.env.example` 或允许清单中的 key，应被视为未知配置并检查失败。
- 派生字段不进入 `.env.example`；否则维护者会误以为它可以被外部独立覆盖。
- SDK、HTTP 客户端或运行平台自动读取的环境变量，如果不由 Settings 消费，不应强行加入应用 Settings；应进入单独允许清单并说明消费方和影响范围。
- 脚本私有变量不进入应用 Settings；它们只能影响脚本编排、端口映射、容器项目名、smoke 参数或临时调试开关。

如果项目把代理、证书、SDK 行为或平台开关设计成显式产品能力或稳定部署承诺，应改为可选 `env-driven` 应用配置。否则不要把自动读取的环境变量伪装成应用派生配置。

## Env Key 映射

Settings 字段和 env key 的映射必须集中、稳定、可机器检查。不要让字段名、alias、`.env.example`、部署模板和测试 fixture 各自维护一套无法比对的配置事实。

映射规则：

- 每个 `env-driven` 必需字段都有稳定 env key。
- 每个进入 `.env.example` 的应用配置 key 都能映射到 Settings 字段。
- 可选 `tunable constants` 只有需要外部调优时才暴露 env key，并说明调优影响。
- `derived` 字段没有 env key，也不出现在 Secret、ConfigMap、Compose environment 或测试 env 模板中。
- 运行形态变量、SDK 自动读取变量和脚本私有变量不映射到应用 Settings 字段，只能进入允许清单或脚本规则。
- 已废弃或已移除的 env key 必须进入拒绝清单，出现时检查失败。

未知 env key 应报错，或进入明确允许清单。不要静默忽略未知 key，也不要保留旧 key 到新 key 的 silent fallback。确实需要迁移窗口时，应显式记录旧 key、截止版本和失败提示，并通过机器检查推动移除。

## 启动校验

非法配置必须在启动或配置加载阶段快速失败。校验分为输入校验和派生后断言。

输入校验至少覆盖：

- 必需密钥不能为空，生产环境不能使用占位符密钥。
- 生产环境不能启用不安全调试开关或本地绕过开关。
- 数据库、Redis、对象存储等必需依赖配置必须完整。
- 常量旋钮不能保留无效值，例如负数 ratio、过小 buffer、无意义 margin。
- env key 不能同时出现新旧名称、重复语义或已废弃名称。

派生后断言至少覆盖：

- 超时链路单调递增，例如 hard timeout 必须大于 soft timeout。
- 派生积压上限必须大于 worker 并发数。
- 领取窗口、stale 阈值、callback 超时等派生值必须满足业务顺序。
- 派生后的最终值不能超过平台或依赖服务的硬限制。

派生配置完成后，校验统一放在 `model_validator(mode='after')` 或项目等价机制中完成。不要把断言散落到 API route、Service、Worker 或脚本中。不要静默修正非法配置，也不要用 fallback 掩盖配置错误。

## 测试与机器检查

配置规则必须进入验证入口，不能只依赖文档提醒。项目应提供测试或脚本证明配置边界没有漂移。

测试中如需覆盖配置，显式构造新的 Settings 实例，或通过 FastAPI `dependency_overrides` 替换依赖。不要修改全局 settings 对象后复用已初始化实例。

配置相关测试至少覆盖：

- 必需运行时配置缺失时启动失败。
- 敏感字段不会出现在 repr、`model_dump()`、日志样例或错误输出中。
- 派生字段不能通过 env 单独覆盖。
- 非法常量旋钮或非法派生关系会触发 Settings 校验失败。
- 废弃 env key 和未知 env key 会被拒绝，或只在明确允许清单中通过。

配置机器检查应能回答：

- Settings 字段清单是否能区分 `env-driven`、`tunable constants` 和 `derived`。
- `.env.example` 中的应用配置 key 是否能和 Settings 字段对齐。
- `.env`、`.env.*`、部署 env 模板、Secret / ConfigMap 模板中的应用配置 key 是否是 `.env.example` 的子集。
- 派生字段是否没有出现在 `.env`、`.env.example` 或项目维护的配置模板中。
- 已废弃或已移除的配置 key 是否被拒绝。
- `verify.sh check`、测试或 CI 是否至少有一个入口执行配置检查。

检查失败时应给出具体 key、来源文件和违反的规则，便于维护者直接修复。

## 配置变更审查

当任务新增、改名、删除或暴露配置项时，Codex 必须先做配置语义审查，再写代码：

- 这个值是否已被归类为 `env-driven`、`tunable constants`、`derived` 或 `non-application env`？
- 如果它是派生值，是否没有新增 env key，且派生关系集中在 Settings？
- 如果它是运行时配置，`.env.example`、字段映射、启动校验和测试是否同步？
- 如果它是脚本或运行形态变量，是否没有进入应用 Settings？
- 如果它是旧 key，是否被删除或进入拒绝清单，而不是 silent fallback？
- 是否新增了启动校验，证明配置项之间不会互相打架？
- 是否有机器检查或测试防止后续漂移？

结论不确定时，默认不要扩大配置面；先保留为内部常量或派生值，并把需要用户确认的外部意图说清楚。
