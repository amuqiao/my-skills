# Flutter / 移动 成熟模式

引擎第 4 步识别出 Flutter/Dart/移动 app/离线/本地持久化/app 生命周期信号后加载本包。使用方式：先在下方按问题维度找匹配的"需求信号"行，取出"成熟模式"列；再回到 SKILL.md 第 5-8 步，与真实场景约束对比、比较候选方案、分析失败模式。本包是地图，不是规则——只有场景需要时才采用对应模式。

---

## 状态管理 (State management)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| 团队刚起步，想要低门槛的状态方案 | 状态作用域、依赖注入、副作用管理都缺少约束，容易产生隐式耦合 | Provider — 轻量依赖注入 + ChangeNotifier；适合小/中型 app 或团队过渡期；当异步逻辑和副作用复杂度上升后，样板量不足以约束副作用边界，不要强行沿用 |
| 需要严格可测试的业务逻辑，团队中等规模以上 | 状态变更路径不可追踪，副作用散落在 UI 层 | BLoC / Cubit — 事件驱动单向数据流，Stream 隔离副作用；可测试性强；样板量较高，纯 UI 展示状态不值得用完整 BLoC |
| 想要编译时安全、跨 Widget 访问、代码生成减少样板 | 全局/局部状态管理混乱，依赖注入和缓存失效边界不清晰 | Riverpod 2.0 — Provider 的类型安全继任者，支持 AsyncNotifier / Notifier，内置 keep-alive/dispose 控制；适合中大型 app；学习曲线高于 Provider，团队规模小时引入成本需评估 |
| 想快速出原型，不想写太多模板，副作用轻 | 过早设计状态架构，拖慢开发速度 | GetX Reactive — 极低样板，内置路由/DI/状态；适合原型和小型 app；依赖魔法注入，大型 app 可测试性差，不建议在需要长期维护的代码库中作为主要架构 |
| 页面很多，需要深链、Web URL、状态恢复 | 声明式路由、deep link 解析、恢复上次页面栈位置 | go_router (declarative routing) — 声明式路由表 + deep link + redirect guard；结合 `RestorationMixin` 实现状态恢复；Navigator 2.0 API 直接使用成本过高，不建议绕过 go_router 自行封装 |
| 用户从通知或外部链接进入特定页面 | Deep link 解析、权限检查、未登录时的重定向 | go_router redirect + ShellRoute — 在路由层统一做 auth guard 和 deep link 分发；不要把 deep link 解析逻辑散落在各 Widget 的 initState 中 |
| 用户切后台再回来，页面状态丢失 | App 被系统暂停后内存状态无法恢复 | State Restoration (RestorationMixin + RestorableProperty) — 将关键 UI 状态序列化到系统恢复机制；仅对用户体验敏感的 UI 状态（滚动位置、表单填写）使用，业务数据应由持久化层恢复 |

- Provider 是 Flutter 官方推荐的起点，适合团队学习和中小型 app；状态复杂后应迁移到 Riverpod 或 BLoC，不是"更好的选择"而是"不同场景下的合适选择"。
- BLoC 适合需要严格测试覆盖的金融/医疗类场景，事件流使状态变更可审计；Cubit 是 BLoC 的轻量版，去掉 Event 类，样板减半，适合副作用不复杂的场景。
- Riverpod 2.0 的 AsyncNotifier 将异步状态（loading/data/error）一等建模，避免自行管理三态；`ref.watch` 的自动依赖追踪减少手动 dispose 错误。
- go_router 的 `redirect` 回调是 auth guard 的标准位置，不要在 Widget build 中做页面跳转决策。
- State Restoration 只解决 UI 状态恢复，不替代数据持久化；两者需配合使用。

---

## 离线与持久化 (Offline & persistence)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| 没网也要能用，数据不能丢 | 本地为真相源，后台同步，冲突解决，重连后重放 | Offline-First — 本地数据库为主，网络层为同步通道；需配套 Outbox Pattern 持久化写入意图；不适合强实时一致性场景（如在线支付、实时多人协作），引入前要明确冲突解决策略 |
| 需要存关系型数据，要跨表查询、JOIN | 移动端本地 SQL 查询，类型安全，迁移版本管理 | Drift (SQLite ORM) — 类型安全 SQL，内置 migration API；适合复杂关系型本地数据；比 Hive/Isar 配置成本高，纯 KV 场景过重 |
| 需要高性能本地存储，查询要快，不想写 SQL | NoSQL 文档型本地存储，快速读写 | Isar — 纯 Dart NoSQL 数据库，索引查询快，跨平台；适合中大型对象图；schema 变更需要手动迁移，比 Drift 迁移 API 弱；Web 支持有限 |
| 只需要存简单配置、用户偏好、token | 轻量 KV 持久化，无结构查询需求 | SharedPreferences / flutter_secure_storage — SP 适合非敏感 KV；token/密钥/凭据必须用 flutter_secure_storage（iOS Keychain / Android Keystore）；不要用 SP 存安全敏感数据 |
| 两台设备改了同一条数据，谁的算数 | 多端或离线后同步的冲突解决策略 | Last-Write-Wins (LWW) — 最简单，用服务器时间戳或设备时间戳决定胜者；时钟漂移会导致丢失更新，仅在丢失更新代价低时使用 |
| 丢更新代价高（如表单填写、重要操作记录） | 冲突需要人工裁决或合并，LWW 不够 | 版本向量 / 服务端裁决 — 服务端持有权威版本，客户端提交时携带 base version；冲突时服务端返回 409 + 冲突体，客户端展示合并 UI 或三方合并；运维成本高，只在数据准确性要求高的场景使用 |

- Offline-First 的核心约定：读取永远先查本地，写入永远先写本地（Outbox），网络仅用于同步；背离任何一条都会产生部分失败下的不一致。
- Drift 和 Isar 的选型不是优劣而是场景：有 JOIN 需求选 Drift，无 JOIN 且追求性能选 Isar，纯 KV 选 SharedPreferences。
- flutter_secure_storage 调用系统级安全存储，不是加密的 SharedPreferences——两者底层完全不同，安全敏感数据不要混用。
- LWW 实现简单，但依赖可靠的时间源；设备时间可被用户修改，服务端时间戳通常更可信。

---

## 数据写入与一致性 (Mutations & consistency)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| 点按钮后 UI 要立刻响应，不能等网络 | 弱网下写入延迟导致 UI 卡顿，用户体验差 | 乐观更新 (Optimistic Update) — 先更新本地状态/UI，后台异步提交；必须配套回滚：网络失败后还原本地状态并通知用户；不要在无离线队列的情况下做乐观更新 |
| 没网时操作不能丢，联网后自动同步 | 离线写入意图丢失（app 被杀后无法重放） | Outbox-on-Device — 写入操作先持久化到本地"待发队列"（用 Drift/Isar 存储），重连后按序重放并标记完成；Outbox 记录必须幂等（携带幂等 key），重放时服务端去重；比内存队列多一步持久化，但这是弱网场景下不丢数据的最低要求 |
| 服务端 API 升级了，老版本 app 报错 | 客户端与服务端版本错配，字段缺失或格式变化导致崩溃 | 强制升级闸门 (Force Upgrade Gate) — 服务端在响应头或版本检查接口返回 min_version，客户端启动时检查并弹出强制升级页；适合重大 breaking change；频繁强制升级伤用户体验，应配合向后兼容设计减少触发频率 |
| 旧版 app 还在用，新功能要灰度 | 新旧客户端能力不一致，服务端无法统一响应 | 能力协商 (Capability Negotiation) — 客户端在请求头携带 app version / feature flags，服务端按能力返回不同响应体；需维护服务端多版本适配逻辑，定期清理老版本支持窗口 |
| API 超时或 500，要不要重试？重试会不会重复提交？ | 移动弱网下重试幂等性，避免重复写入 | 幂等键 + 客户端去重 (Idempotency Key) — 写入请求携带客户端生成的 idempotency_key，服务端对相同 key 的重复请求返回缓存结果；重试逻辑见"异步可靠性"维度；服务端幂等实现见 `patterns-python-backend.md` |

- 乐观更新和 Outbox 是互补的：乐观更新解决 UI 响应性，Outbox 解决离线写入持久性；弱网 app 两者都需要。
- Outbox 中的每条记录应包含：操作类型、目标实体 ID、payload、幂等 key、创建时间、重试次数、状态（pending/in-flight/done/failed）。
- 强制升级闸门应设计为可远程配置（如 Remote Config），避免修改阈值时需要发版。
- 能力协商的版本支持窗口应有明确的 EOL 策略，否则服务端会长期维护大量兼容分支。

---

## 异步可靠性 / 并发与生命周期 (Concurrency & lifecycle)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| 解析大 JSON / 图片处理 / 加密运算导致 UI 卡顿 | 重计算占用主 isolate，帧率下降 | compute isolate — 用 `compute()` 或手动 `Isolate.spawn` 将 CPU 密集型任务卸载到独立 isolate；仅用于纯计算（无 Flutter UI 调用）；数据传递有序列化开销，小数据集不值得引入 |
| 需要在后台定期同步数据，app 不在前台也要跑 | 后台任务调度，系统资源限制，跨平台差异 | WorkManager (Android) / BGTaskScheduler / BackgroundFetch (iOS) — Android WorkManager 支持延迟/周期任务，受 Doze 模式限制；iOS 后台执行受系统严格限制，不保证准时触发，不要依赖 iOS 后台任务做时间敏感操作；flutter_background_fetch 插件提供跨平台封装，但 iOS 侧行为需单独验证 |
| App 切后台后再回来，网络连接断了，数据没刷新 | App 生命周期事件处理：resume/pause/detach | AppLifecycleObserver + WidgetsBindingObserver — 在 `resumed` 时触发状态重新加载和网络重连；在 `paused` 时保存关键状态；不要在 `detached` 依赖网络调用完成，进程可能随时被终止 |
| 网络差，请求经常失败，想自动重试 | 弱网重试策略，避免雪崩，处理非幂等请求 | 指数退避 + Jitter (Exponential Backoff with Jitter) — 每次重试等待时间指数增长并加随机抖动，避免多设备同时重试打垮服务端；只对幂等操作重试（GET、带幂等 key 的 POST）；服务端重试策略见 `patterns-python-backend.md` 的 backoff + idempotency |
| 同一个请求用户多次触发，发出了多个重复网络请求 | 请求去重，取消过期请求，避免竞态 | 请求去重 / CancelToken — 用 Dio 的 CancelToken 或 Riverpod 的 `ref.onDispose` 取消过期请求；对同一资源的并发请求合并为一个（request deduplication）；避免在 `build()` 内直接发起网络请求 |
| 网络请求超时没有边界，用户等很久没响应 | 未设置超时导致请求挂起，UI 卡在 loading 状态 | 请求超时 + 超时 UI 反馈 — 在 HTTP client 层（Dio baseOptions）统一设置 connectTimeout / receiveTimeout；超时后显示明确的用户提示并提供重试入口；不要只在底层静默吞掉超时错误 |

- iOS 后台执行有严格系统限制：BGAppRefreshTask 不保证触发时机、时间配额极短（refresh 类任务通常仅数十秒）；需要更长运行时间的任务应改用 BGProcessingTask（可要求联网/充电，可运行数分钟），但仍不保证准时。任何需要可靠后台同步的场景都应同时设计"前台 resume 时补偿同步"逻辑。
- `compute()` 底层基于 `Isolate.spawn` + 端口通信，适合一次性计算（与 Dart 2.19+ 的 `Isolate.run` 行为相近，但并非对它的封装）；持续后台工作（如音频处理）需要长生命周期 isolate 手动管理。
- 指数退避的重试仅限幂等操作；非幂等操作（如创建订单）需先将意图持久化到 Outbox，再在重连后重放，不能盲目重试原始请求。
- 网络重连检测推荐使用 `connectivity_plus`，但注意该包只反映网络接口状态，不代表实际可达性；建议配合一次轻量 ping 或首个真实请求成功来确认真正重连。

---

## 反模式提示

以下是 Flutter/移动特有的反模式，通用推理类反模式见 `anti-patterns.md`；服务端重试/幂等实现见 `patterns-python-backend.md`。

- **长任务只靠内存状态。** 写入操作或长流程只存在内存中，app 被系统回收（OOM/用户划走）后状态永久丢失。修复方向：把写入意图持久化到 Outbox，进程恢复后重放。
- **弱网乐观更新无离线队列无回滚。** 乐观更新后网络失败，没有持久化写入意图，也没有回滚 UI 到旧状态的路径，导致用户看到的本地数据与服务端永久不一致。修复方向：乐观更新必须配套 Outbox + 回滚逻辑。
- **把 server state 当本地真相源但无冲突解决。** 每次从服务端全量拉取并直接覆盖本地，离线期间的本地修改在下次同步时被静默丢弃。修复方向：明确冲突解决策略（LWW / 版本向量 / 服务端裁决），在同步前合并而非覆盖。
- **依赖 iOS 后台任务做可靠定时同步。** iOS BGAppRefreshTask 触发时机由系统决定，不保证准时，后台时限极短。修复方向：将后台同步作为"尽力而为"补充，前台 resume 时必须补偿一次全量/增量同步。
- **未设置 HTTP 超时且静默吞掉超时错误。** 请求无限挂起，UI 永远 loading，用户无感知也无重试路径。修复方向：在 HTTP client 层统一配置超时，超时后向用户暴露错误并提供重试入口。
