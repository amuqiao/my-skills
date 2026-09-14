# Python / 后端 成熟模式

**加载条件**：引擎第 4 步识别出项目类型为 Python 后端（包括 FastAPI / Django / Flask / 异步 worker / 数据管道等）时，读取本包。**使用方式**：先在下方各维度小节的表格中，按用户的原话或需求信号找到对应行，取出"成熟模式"列；再回到引擎第 5-8 步做方案比较、组合分析与失败模式审查。本包是模式地图，不是强制配方——只有场景真正需要时才采用对应模式。命中多个信号时，优先考虑组合使用（例：可靠任务系统 = Transactional outbox + Idempotent consumer + Lease + DLQ + Reconciler + Structured logs）。

---

## 异步可靠性 (Async reliability)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "任务发出去了但不知道有没有执行" / "消息有时候莫名消失" | 写 DB 成功后，消息发布作为独立副作用，进程崩溃则两者不一致 | **Transactional outbox** — 边界：仅适用于写 DB 与发消息须原子的场景；不需要跨服务一致时不要用，增加 polling 开销 |
| "同一条消息/任务被执行了多次" / "重复扣款/重复发邮件" | 网络重试或 at-least-once 队列导致消费多次 | **Idempotent consumer** — 边界：需要持久化去重 key（DB unique constraint 或 Redis SET NX）；纯内存去重重启后失效，不算真幂等 |
| "多个 worker 抢同一个任务" / "任务执行到一半 worker 挂了没人接" | 分布式任务所有权与存活探测 | **Lease + heartbeat** — 边界：lease 时长须大于最长正常执行时间；heartbeat 间隔须能被监控到；不适合毫秒级任务 |
| "失败的任务一直重试把系统打满" / "毒消息卡住整条队列" | 不可恢复消息阻塞消费者 | **Dead letter queue (DLQ)** — 边界：进入 DLQ 必须告警；不要把 DLQ 当成最终坟场，需要人工修复路径或 replay 机制 |
| "下游恢复后大量重试同时打过去把它再打垮" / "重试风暴" | 重试集中在同一时间点形成惊群 | **Exponential backoff + jitter** — 边界：backoff 上限和 max_retries 必须配置；jitter 用 full jitter 而非 equal jitter，仅防惊群用 decorrelated jitter |
| "偶尔会有卡住的任务，但没有人发现" / "任务 pending 了几小时" | 无监督的僵尸任务或状态漂移 | **Sweeper / reconciler** — 边界：扫描周期远大于正常处理时间；必须记录每次扫描动作；不要用于替代主流程，只做兜底对账 |
| "一个大流程里某一步失败了，之前的步骤需要回滚" / "分布式事务" | 跨服务操作缺乏补偿路径 | **Saga / process manager** — 边界：每个步骤必须有补偿操作（且补偿本身也必须幂等）；流程状态持久化到 DB，不依赖内存 |
| "选哪个任务队列" / "Celery 太重 / arq 不支持 cron / Dramatiq 不熟悉" | worker 栈与可靠性需求、调度复杂度、async 原生支持不匹配 | **Worker stack selection** — Celery：成熟、生态丰富、支持复杂调度，但同步优先，async 支持靠 gevent/eventlet 绕路；RQ：极简 Redis 依赖，适合轻量场景，cron/retry 需手动；arq：async 原生（asyncio），适合 async Python 服务，调度功能基础；Dramatiq：强调可靠性（middleware 机制），actor 模型，适合中等复杂度。边界：需要 DB 事务级别的任务幂等，选任何库都必须额外实现 outbox + idempotency key |

- **Transactional outbox**：DB 写入与消息发布原子化；worker 轮询 outbox 表，适用于业务写操作必须带发消息语义的场景。
- **Idempotent consumer**：消费前先检查 idempotency key 是否已处理；使用 DB unique constraint 或 `SET NX` 实现，适用于 at-least-once 队列。
- **Lease + heartbeat**：任务持有者定期续租，超时则释放；适用于需要防止并发执行或 worker 失联后自动接管的长任务。
- **Dead letter queue**：超过重试上限的消息移入 DLQ 并告警；DLQ 条目须包含足够上下文支持 replay 或人工处理。
- **Sweeper / reconciler**：定期扫描"应终态未终态"的记录并触发修复；是状态机可靠性的最后一道防线，必须有执行日志。

---

## 数据/一致性 (Data & consistency)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "两个请求同时修改同一条记录，后写的覆盖了先写的" | 并发写丢失更新（lost update） | **Optimistic concurrency / CAS（Compare-and-swap）** — 边界：冲突率高时退化为大量重试；高争用场景改用悲观锁或队列序列化；version 字段必须在应用层校验，不能只靠 ORM |
| "同一个请求被重复提交，需要保证只执行一次" | 网络重传或用户双击导致重复写入 | **Unique constraint + idempotency key** — 边界：key 生成在客户端，服务端只做校验；key 过期时间须与业务容忍度匹配；不要依赖 try/except 来"兼容"重复而不返回原结果 |
| "需要审计每一笔变更" / "余额/库存不能直接覆盖" | 可变字段覆盖丢失历史，无法重放 | **Append-only ledger** — 边界：当前值需聚合计算，查询成本高；通常与 Materialized summary 组合；不适合频繁读取的低价值字段 |
| "报表查询太慢" / "聚合查询拖垮主库" | 实时聚合在 OLTP 库上性能不可接受 | **Materialized summary** — 边界：必须有失效/刷新策略；数据可能短暂过时，不适合需要强一致性的财务核心数据 |
| "主库写多读多，读操作影响写性能" / "事件驱动的读模型" | 读写混合负载或 CQRS 场景下读模型与写模型职责耦合 | **Read model projection** — 边界：投影必须幂等且可重建；投影滞后须在业务层说明"最终一致性"含义 |
| "连接池设多少" / "DB 连接被打满了" | 池大小与并发 worker 数、DB 连接上限三者不匹配 | **Connection pool sizing** — 上限粗估（非精确公式，还需扣除 superuser 预留、迁移/管理工具及其他服务占用的连接）：pool_size ≲ (DB max_connections × 0.8) / worker_count；async 框架（asyncio）单 worker 并发高，pool_size 可设小（如 5-20），同步 worker 每请求占一连接。边界：SQLAlchemy `pool_size + max_overflow` 必须 ≤ DB 单实例上限；pgbouncer 做连接复用时 pool_size 可进一步缩小 |
| "加字段要停服" / "在线迁移怎么做" | 迁移与代码部署顺序不对导致双版本兼容性问题 | **Expand-contract / online-safe migration** — 先加新列（expand，旧代码兼容），再部署读写新列的代码，最后删旧列（contract）；边界：每一步须可独立回滚；大表加列用 `ALTER TABLE ... ADD COLUMN` + 后台 backfill，不要在迁移脚本里做全表 UPDATE |

- **Optimistic concurrency / CAS**：在 `UPDATE ... WHERE version = ?` 中校验版本号，更新失败返回冲突；适合低争用的并发写场景。
- **Append-only ledger**：只插入不修改，当前值通过 SUM/MAX 聚合；与 Materialized summary 组合解决读性能问题。
- **Expand-contract migration**：三阶段迁移使新旧代码可同时运行；是零停机部署的前提，每阶段单独上线并验证。
- **Connection pool sizing**：async 框架中高并发不等于高 pool_size，反而应压低池大小防止 DB 过载。

---

## API/合同 (API & contract)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "前端不知道这个错误是什么意思" / "各接口响应格式不统一" | 调用方无法机器化处理响应，排查成本高 | **Stable envelope（统一响应结构）** — 固定字段：`request_id / code / message / data`；边界：不要把 HTTP 状态码语义全放进 code 字段，两者分工明确；envelope 一旦对外发布不可随意改字段名 |
| "改了接口老客户端就挂了" / "怎么平滑升级 API" | 接口变更破坏已有调用方 | **Versioned API contract** — URL 路径版本（`/v1/`）或 Header 版本（`Accept: application/vnd.api+json;version=2`）；边界：版本不是补丁更新的借口，破坏性变更才需新版本；老版本必须明确 sunset 策略 |
| "这个接口又查又改，不知道算什么" / "CRUD 还是命令式" | 路由语义不清晰，幂等性和权限边界难以划定 | **Capability-specific route vs Generic command route** — 高价值操作（支付、审批、状态流转）用能力路由（`POST /orders/{id}/confirm`）；CRUD 资源用 REST 资源路由；边界：Generic command route（`POST /actions`）适合工作流引擎或低频操作，不适合需要精细权限控制的场景 |
| "怎么验证 webhook 真的是我们发的" / "webhook 被伪造了" | 第三方回调无法验证来源真实性 | **Signed webhook callback** — HMAC-SHA256 签名加时间戳（防重放），接收方校验签名后再处理；边界：密钥须通过 Secret separation 管理；时间窗口（如 ±5 min）须与业务容忍度匹配 |
| "在 async FastAPI 里调用了一个同步库，性能很差" / "数据库 ORM 阻塞了事件循环" | 同步阻塞调用在 asyncio 事件循环中阻塞整个进程 | **ASGI 边界（sync/async offload）** — 纯 I/O 用 `await`；同步阻塞调用（同步 ORM、CPU 密集、第三方 SDK）用 `loop.run_in_executor` 卸载到线程池；CPU 密集用 `ProcessPoolExecutor`；边界：不要在 async handler 中直接调用同步阻塞函数；`asyncio.to_thread` (Python 3.9+) 是 `run_in_executor` 的语法糖 |

- **Stable envelope**：统一 `request_id / code / message / data` 四字段；`request_id` 贯穿日志和链路追踪，是可观测性的基础。
- **Versioned API contract**：版本策略在首个公开版本发布前确定，事后改造成本极高。
- **Capability-specific route**：显式命名业务操作（`/cancel`、`/approve`）而非用通用 PATCH，使权限和幂等语义清晰可审计。
- **ASGI 边界**：async 框架不自动使同步代码变快；未卸载的阻塞调用会饿死整个事件循环，表现为莫名的高延迟。

---

## 安全/访问 (Security & access)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "鉴权逻辑分散在各个接口里" / "有的接口忘了加权限校验" | 鉴权逻辑碎片化，遗漏导致越权 | **Central auth boundary** — 在框架中间件或依赖注入层统一处理认证与授权；边界：业务逻辑层不做鉴权，只接受已验证的身份上下文；中间件必须覆盖所有路由，包括健康检查以外的内部接口 |
| "用户能做哪些操作不清楚" / "权限太细管不过来" | 访问控制粒度与实施位置不一致 | **Capability allowlist** — 用白名单声明"允许做什么"而非黑名单"禁止做什么"；边界：allowlist 须持久化存储，不能只在内存或配置文件中；RBAC 是 allowlist 的一种实现 |
| "密钥/密码写在代码里了" / "配置文件里有 DB 密码" | 凭据泄露风险 | **Secret separation** — 密钥通过环境变量、Vault 或云 Secret Manager 注入，不进代码仓库；边界：`.env` 文件不得提交；密钥轮换须有自动化流程，不依赖手动重部署 |
| "webhook 被伪造调用" / "回调接口被扫描器刷" | 外部回调来源不可信 | **Signed callbacks / webhooks** — 见 API/合同维度；安全层需同时验证签名有效性和时间戳防重放；边界：校验失败直接返回 400/401，不泄露任何内部错误细节 |
| "接口被刷，DB 被打满" / "某个用户的操作影响了其他用户" | 单用户/单 IP 资源滥用导致整体服务降级 | **Rate limit / bulkhead** — Rate limit 在网关或中间件层按用户/IP/接口限速；Bulkhead 用独立线程池或进程隔离不同服务调用；边界：Rate limit 须区分"软限"（429 重试）和"硬限"（封禁）；Bulkhead 不适合过细粒度，否则资源碎片化反而降低吞吐 |

- **Central auth boundary**：FastAPI 中用 `Depends` 注入统一 auth guard；Django 中用 middleware + permission_classes；不要在 service 层重复鉴权。
- **Secret separation**：`python-dotenv` 仅用于本地开发；生产环境用 Vault / AWS Secrets Manager / GCP Secret Manager 注入；`os.environ` 读取，不传入配置对象后再到处传递。
- **Rate limit / bulkhead**：`slowapi`（FastAPI）或 `django-ratelimit` 实现应用层限速；网关层（nginx / Envoy）做第一道防线；bulkhead 用 `concurrent.futures` 或 `asyncio.Semaphore` 实现。

---

## 可观测/运维 (Observability & ops)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "健康检查通过了但服务还是没法用" / "就绪检查没有" | liveness 与 readiness 不区分，上游在服务未就绪时就打流量 | **Health / readiness split** — `/health`（liveness）只检查进程存活；`/ready`（readiness）检查 DB 连接、依赖服务、队列等外部依赖；边界：readiness 检查须设超时（< 2s），避免慢依赖导致探针超时被误杀 |
| "日志里看不出是哪个请求报的错" / "分布式链路追踪断了" | 日志无关联 ID，无法跨服务串联请求 | **Structured logs with correlation IDs** — 每条日志输出 JSON，携带 `request_id / trace_id / span_id / service / level / timestamp`；边界：`request_id` 从请求入口生成并通过 context var 传递到所有下游调用；Python 用 `structlog` 或 `python-json-logger` |
| "任务队列堆了多少不知道" / "系统慢了但不知道瓶颈在哪" | 缺少饱和度指标，无法提前发现过载 | **Saturation metrics** — 监控：队列深度、活跃 worker 数、重试次数、p95/p99 延迟、失败率、连接池等待数；边界：告警阈值须基于压测基线设定，不要凭感觉；指标用 Prometheus + Grafana 或 Datadog，不要只靠日志聚合 |
| "出事了但看不出系统现在处于什么状态" / "只能看日志猜" | 系统运行状态只存在于内存或日志中，无法直接查询 | **Runbook-visible states** — 关键业务状态（任务状态、Saga 阶段、迁移进度）持久化到 DB 并通过管理端点或工具可查；边界：状态字段须有 `updated_at` 和操作者记录；恢复路径依赖可查询状态，不依赖日志回放 |

- **Health / readiness split**：Kubernetes `livenessProbe` 对应 `/health`，`readinessProbe` 对应 `/ready`；两者混用会导致"服务未就绪就接流量"或"依赖抖动触发重启"。
- **Structured logs with correlation IDs**：`contextvars.ContextVar` 在 asyncio 中安全传递 request context；`structlog` 绑定 context 后所有子调用自动携带。
- **Saturation metrics**：队列深度是异步可靠性最重要的先行指标；连接池等待数是数据库过载的早期信号，优先于慢查询率告警。
- **Runbook-visible states**：Saga 和 process manager 的价值之一正是将流程状态持久化，使运维人员可直接介入修复，而非靠重启恢复。

---

## 反模式提示

以下是 Python 后端特有的高频反模式，通用推理类反模式见 `anti-patterns.md`。

- **在 asyncio 事件循环中直接调用同步阻塞函数**：未用 `run_in_executor` 或 `asyncio.to_thread` 卸载的同步 I/O（包括同步 ORM、requests 库、文件读写）会阻塞整个事件循环，导致所有并发请求延迟飙升，且难以通过 CPU 监控发现。
- **用 try/except 吞掉幂等校验后返回 200**：`INSERT ... ON CONFLICT DO NOTHING` 或 `except IntegrityError: pass` 在重复请求时静默成功，但未返回原操作结果，调用方无法区分"首次成功"与"重复忽略"，破坏幂等语义。
- **连接池大小与 worker 并发数不匹配**：async 框架（如 FastAPI + asyncpg）单进程并发极高，pool_size 不压低会导致 DB 连接耗尽；相反，同步 Django + gunicorn 每请求一个连接，pool_size 过小会导致请求排队。两种场景默认值不同，不能互用。
- **迁移脚本中做全表 UPDATE 或 NOT NULL 加列**：在大表上 `ALTER TABLE ADD COLUMN NOT NULL DEFAULT x` 会锁表（PostgreSQL 11 之前）或导致长时间写阻塞；在迁移脚本中对百万行做 `UPDATE` 会持有事务锁，应改用后台 backfill + 分批提交。
- **Celery task 里持有不可序列化的对象**：将 SQLAlchemy model 实例、文件句柄或 asyncio 协程直接作为 task 参数，序列化时静默截断或抛出难以追踪的错误；task 参数只传 ID 或原始类型，在 task 内部重新查询。
