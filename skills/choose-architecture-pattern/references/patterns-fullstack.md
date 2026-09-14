# 全栈 / 接缝 成熟模式

**加载条件**：引擎第 4 步识别出项目类型涉及"前后端接缝"时读本包——即需求同时横跨浏览器/移动端与服务端，且关键工程问题出现在两端的边界上。**使用方式**：先在下方各维度小节中，按需求信号找到对应行，确认"真实工程问题"；再回到引擎第 5-8 步，用"成熟模式"列的结论驱动方案选择与生产审查。

**本包只放前后端接缝模式，不重复 `patterns-frontend.md` 和 `patterns-python-backend.md` 的内容。** 纯前端问题（服务端状态缓存、乐观 UI、渲染策略、客户端状态机等）见 `patterns-frontend.md`；纯后端问题（异步可靠性、数据一致性、ASGI 边界、可观测性等）见 `patterns-python-backend.md`。本包的价值是"把两端缝好"，不是复述两端已有的内容。

---

## API 合同 (API Contract)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "前端改了字段名，后端不知道；后端改了字段，前端直接崩" | 两端靠口头约定或文档维护合同，无机器可校验的单一真相源 | **Schema-first contract** — 以 OpenAPI / GraphQL SDL / Protobuf 作为单一真相源，两端均从该 schema 生成代码；边界：schema 本身不自动保证语义正确，须配套契约测试（见"契约测试"维度） |
| "后端接口改了，前端不知道，上线才发现" | 合同变更无通知机制，靠人工沟通同步 | **Breaking-change detection** — CI 中对比新旧 schema diff，阻断含破坏性变更的 PR；边界：仅检测结构变更，语义变更（如枚举新增值的含义变化）需人工审查 |
| "老版本接口要支持多久，什么时候能删" | 缺少 API 生命周期策略，旧版本无限期存活，维护负担不断累积 | **Sunset policy** — 在响应 Header 中返回 `Deprecation` 和 `Sunset` 字段，配套日志统计旧版调用量；达到 sunset 日期后方可删除；边界：Sunset 策略须在首个公开版本发布前确定，事后改造成本极高——响应格式与版本规范见 `patterns-python-backend.md` 的 `## API/合同 (API & contract)` 小节（Stable envelope、Versioned API contract）|
| "前端直接拼 URL 调了内部微服务，绕过了网关" | 前端与后端内部拓扑耦合，网关安全策略被绕过 | **BFF / API gateway as seam** — 前端只与 BFF 或 API Gateway 通信，内部服务拓扑对前端不可见；边界：BFF 适合"为特定前端裁剪数据格式"的场景；若仅做转发无裁剪，引入 BFF 是过度工程 |

- **Schema-first contract**：OpenAPI、GraphQL SDL、Protobuf 等使前后端独立开发时能以机器可验证的方式对齐接口。适合团队规模 ≥ 2 人、接口变更频繁的场景；单人项目且 API 极少变动时，维护 schema 文件本身可能比收益更重。
- **Breaking-change detection**：工具有 `oasdiff`（OpenAPI）、`graphql-inspector`（GraphQL）；在 CI 阶段阻断而非只告警，才能真正保护调用方不被意外破坏。
- **BFF (Backend for Frontend)**：为特定前端（Web、移动端、桌面）提供一层聚合与裁剪，避免前端直接与多个内部服务通信。适合多端差异大、数据聚合逻辑复杂的场景；若只有一个前端且 API 已足够聚合，BFF 是冗余层。

---

## 端到端类型安全 (End-to-end Type Safety)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "前端用了一个字段，结果后端从来没返回过这个字段" | 前端类型是手工维护的，与后端实际实现长期漂移 | **Codegen from single source of truth** — 从 OpenAPI / GraphQL / Protobuf schema 自动生成前端 TypeScript 类型；后端类型由同一 schema 约束；边界：codegen 必须集成进 CI，手动更新生成文件等于放弃自动对齐 |
| "前后端各自维护了一份 Zod schema，经常不一致" | 验证规则双重维护，字段规则变更只改一端 | **Shared validation schema** — 以 Zod（前端）+ 对应生成的 JSON Schema 或 pydantic 模型（后端）为同一份规则的两端实现，通过 schema 文件作为桥接；边界：语义等价需人工验证，工具只能保证结构一致 |
| "想引入 tRPC / GraphQL，但团队只有 2 个人" | 端到端类型同步的收益不足以覆盖工具链引入成本 | **当场景不值得时，不要引入** — 小团队 + 快速迭代期 + 接口变化频繁时，手工维护类型 + 契约测试兜底的成本低于维护 codegen 工具链；边界：当接口数量超过约 15-20 个且前后端由不同人维护时，自动化对齐的收益开始超过成本 |
| "后端换了语言/框架，前端类型全要重写" | 前端类型与后端实现语言耦合，而非与协议耦合 | **Protocol-bound types** — 前端类型与 HTTP/gRPC/GraphQL 协议层对齐，不直接依赖后端语言类型；schema 文件是解耦的接缝；边界：Protocol buffer / OpenAPI 的表达能力有上限，复杂联合类型或递归结构需额外处理 |

- **Codegen from single source of truth**：常用工具链包括 `openapi-typescript`（OpenAPI → TypeScript）、`graphql-codegen`（GraphQL SDL → TypeScript）、`protoc-gen-ts`（Protobuf → TypeScript）。价值在于"类型漂移变成 CI 失败"，而非"类型永远正确"——语义仍需测试覆盖。
- **Shared validation schema**：Zod schema 可通过 `zod-to-json-schema` 导出 JSON Schema，再由 pydantic 的 `model_validate` 加载；或反向由 pydantic 生成 JSON Schema，前端用 `json-schema-to-zod` 消费。两种方向均有工具链，选择时以"哪端是合同权威"为准。
- **何时是过度工程**：团队单人、接口 < 10 个、前后端同一人维护、原型期——这些场景下 codegen 工具链的维护成本超过收益。手工维护 + 契约测试是更轻量的替代。

---

## 认证授权边界 (Auth Boundary)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "前端存了 JWT，怎么防止被 XSS 偷走" | token 存储位置决定攻击面，不同存储策略对应不同威胁模型 | **Token storage strategy** — HttpOnly cookie 防 XSS 读取（但需 CSRF 防护）；`sessionStorage` 防 XSS（但标签关闭即失效）；`localStorage` 最易受 XSS；选择依据威胁模型而非便利性；边界：前端无论如何存储 token，服务端必须独立验证，不能信任前端传来的身份声明 |
| "用户登录后，前端记了权限，但实际权限已经被后台改了" | 前端缓存了权限快照，与服务端真实状态不同步 | **权限裁决在服务端，前端只做 UI 提示** — 前端权限状态仅用于渲染决策（显示/隐藏按钮），不构成安全边界；服务端每次请求独立鉴权；边界：前端权限缓存失效时间须比 token 有效期短，或通过 token 刷新时重取权限清单——前端客户端反模式见 `patterns-frontend.md` 的 `## 反模式提示` 中"在客户端裁决鉴权与权限"；服务端中心化鉴权实现见 `patterns-python-backend.md` 的 `## 安全/访问 (Security & access)` 小节（Central auth boundary） |
| "refresh token 存哪，怎么无感刷新" | refresh token 的存储与轮换策略影响安全性与用户体验 | **Silent refresh with rotation** — access token 短有效期（5-15 min）+ HttpOnly cookie 存 refresh token；access token 过期前静默请求新 token；rotation 策略：每次使用 refresh token 后颁发新 refresh token 并作废旧的（检测重用即吊销）；边界：rotation 需要服务端持久化 token 状态，无状态 JWT 架构需要额外存储层 |
| "多个子域名/微服务都需要鉴权，每个都单独做" | 鉴权逻辑分散，各服务实现不一致，难以统一吊销 | **Centralized auth service + token forwarding** — 独立 auth service 颁发 token，下游服务只做 token 验证（public key 验证签名），不做颁发；边界：token 吊销需要 token 黑名单或极短有效期，纯 JWT 无状态验证无法做到即时吊销 |
| "有状态 session 还是无状态 JWT，怎么选" | 两种方案在可扩展性、吊销能力、实现复杂度上各有取舍 | **Stateful session vs Stateless token** — 有状态 session（server-side session store）：即时吊销简单，适合单体/小规模；无状态 JWT：横向扩展无需共享存储，适合多实例/微服务，但吊销需要额外机制；边界：不要以"JWT 更现代"为由默认选无状态——如果需要即时吊销，无状态 JWT 须配套黑名单，实质上已有状态化 |

- **Token storage strategy**：HttpOnly + Secure + SameSite=Strict cookie 是目前对 XSS 防护最强的 token 存储方式，但需要额外的 CSRF 防护（如 Double Submit Cookie 或 `SameSite=Strict` 本身）。选择时先确定威胁模型，再选存储策略。
- **Silent refresh with rotation**：前端使用 axios/fetch 拦截器捕获 401，自动调用 `/auth/refresh` 后重试原请求；refresh token rotation 使单次泄漏只能被利用一次。
- **Centralized auth service**：服务间 token 验证仅需 public key（可通过 JWKS endpoint 获取），不需要调用 auth service；颁发逻辑集中在一处，减少各服务的鉴权实现差异。

---

## 前后端数据流 (Data Flow)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "前端乐观更新了，但服务端失败了，数据对不上，刷新后又变回去了" | 乐观更新的本地状态与服务端确认结果存在窗口期不一致，用户看到数据闪烁 | **Optimistic update reconciliation** — 前端乐观更新时保留旧快照；`onError` 回滚；`onSuccess` / `onSettled` 时 invalidate 缓存触发重取，以服务端确认结果为最终真相源；边界：乐观 UI 实现见 `patterns-frontend.md` 的 `## 数据写入与 UI 一致性 (Mutations & UI Consistency)` 小节；本接缝关注"服务端确认与前端对账"这一跨端问题 |
| "后端有个后台任务改了数据，前端不知道，一直显示旧数据" | 服务端主动变更无法推送到前端，前端只能靠轮询发现 | **Server-sent events / WebSocket for server-initiated updates** — 只读单向推送用 SSE（实现简单，HTTP/2 原生支持）；双向通信用 WebSocket；边界：实时推送会引入连接管理、重连、扩容（sticky session 或 pub/sub）复杂度，不需要实时性时轮询仍是有效选择 |
| "后端 outbox 里的事件已经发出去了，但前端缓存还没失效" | 服务端事件驱动的状态变更与前端缓存失效时机不同步 | **Cache invalidation via push** — 服务端在事件发出时通过 WebSocket/SSE 推送 `invalidate:resourceType:id` 消息；前端收到后调用 `invalidateQueries`；边界：推送仅触发失效，不携带完整数据，避免推送数据与缓存一致性的双重问题——服务端事件发布可靠性见 `patterns-python-backend.md` 的 `## 异步可靠性 (Async reliability)` 小节（Transactional outbox） |
| "谁是数据的真相源，前端算出来的还是后端存的" | 前后端各自维护派生数据，导致计算结果不一致 | **Single source of truth — 服务端持久化，前端只展示** — 业务关键数据（余额、库存、权限）以服务端存储为准；前端展示的派生值（合计、状态文本）若需强一致，在服务端计算后返回，不在前端重复计算；边界：纯 UI 展示的派生值（格式化、本地化）在前端计算完全合理，不需要往返服务端 |

- **Optimistic update reconciliation**：前端负责即时感知，服务端负责最终确认。`onSettled`（无论成功或失败均执行）触发 invalidation 是最安全的对账策略，确保 UI 最终收敛到服务端真实态。
- **SSE vs WebSocket**：SSE 是 HTTP 协议，天然穿透代理和防火墙，自动重连；适合服务端单向推送（通知、状态变更）。WebSocket 需要额外握手，适合聊天、协同编辑等双向高频通信。两者都有扩容成本，需要 sticky session 或外部 pub/sub（Redis Pub/Sub、Kafka）。
- **Cache invalidation via push**：推送"失效信号"而非"完整数据"，可避免推送时序与缓存写入产生竞争；前端在收到信号后主动重取，以自身 query 时序为准。

---

## 契约测试 (Contract Testing)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "后端改了接口以为没人用，结果前端挂了" | 接口变更无感知，缺少调用方视角的变更检测 | **Consumer-driven contract (CDC)** — 前端（consumer）定义它依赖的接口字段和行为，后端（provider）在 CI 中验证它的实现满足所有 consumer contract；边界：CDC 测试工具（Pact）有学习曲线，小团队 + 单一前端可用 OpenAPI schema snapshot 替代，成本更低 |
| "OpenAPI 文档和实际接口不一致，文档是摆设" | 文档手写与实现漂移，不可信 | **Schema snapshot test** — 在 CI 中对比生成的 OpenAPI schema 与代码库中的快照文件；实现变更若未更新快照则构建失败；边界：schema snapshot 只能检测结构，不检测语义（如状态码含义变化） |
| "路由、OpenAPI 文档、TypeScript 类型三个地方定义，总有一个漂移" | 三方漂移（route / doc / type）是接缝最常见的长期腐化来源 | **Single source, multiple derivations** — 从单一定义（如 FastAPI route 函数签名）自动派生 OpenAPI schema，再从 schema codegen 前端类型；三者同源，任何一处修改自动传播；边界：自动派生不覆盖所有表达能力（如复杂 discriminated union），此时需补充手写 schema 描述 |
| "前端 mock 和真实接口行为不一致，联调才发现" | 开发期 mock 与真实后端行为偏差，集成阶段爆问题 | **Contract-aligned mock** — mock（如 MSW）从同一 OpenAPI schema 或 Pact contract 生成，而非手工编写；边界：schema 本身无法描述后端的有状态行为（分页光标、乐观锁冲突响应），此类行为须额外编写 mock handler |

- **Consumer-driven contract**：Pact 是最成熟的 CDC 工具；前端写 Pact test 定义"我期望后端返回什么"，后端在 provider 验证中确认"我确实返回了这个"。适合多团队、微服务场景；单团队 + monorepo 时 schema snapshot 成本更低。
- **Schema snapshot test**：FastAPI 的 `app.openapi()` 可在测试中调用并序列化；与 git 管理的快照文件 diff，变更须显式确认；是最低成本的"文档即测试"手段。
- **Contract-aligned mock**：`msw`（Mock Service Worker）支持从 OpenAPI 自动生成 handler；开发期 mock 与生产行为同源，减少联调阶段的惊喜。与 `implementation-contract-plans` skill 中"current/contract/plans 文档分离"的理念一致——contract 是持久真相源，plans 是变更意图，current 是已验证实现。

---

## 反模式提示

以下是全栈接缝特有的高频反模式。通用推理类反模式见 `anti-patterns.md`；纯前端反模式见 `patterns-frontend.md` 的 `## 反模式提示`；纯后端反模式见 `patterns-python-backend.md` 的 `## 反模式提示`。

- **在前端裁决业务权限，服务端不验证**：仅靠前端隐藏按钮或路由守卫控制敏感操作，服务端接口不独立鉴权。任何客户端权限判断只是 UI 提示，修改 JS 或直接发请求即可绕过。所有敏感操作须在服务端独立验证，不信任前端传来的身份或权限声明。
- **前后端各自维护一份类型，靠口头约定同步**：手工维护的两份类型定义会在接口演进中逐渐漂移。字段重命名、类型收窄/扩展、枚举新增值——任何一处单侧修改都会在运行时才暴露问题。接缝类型须从单一真相源（schema 文件）自动派生，并在 CI 中验证。
- **乐观更新后不以服务端确认结果为准**：前端乐观更新后，即使成功也不触发 invalidation，长期依赖客户端推算的状态。服务端可能做了裁剪、校准或副作用，前端推算值与服务端持久值长期不一致。`onSuccess` 时仍须 invalidate 相关缓存，以服务端返回值为最终真相。
- **BFF 只做透传不做裁剪**：引入 BFF 层但仅转发请求，不做任何聚合、裁剪或格式适配，增加了一跳延迟和维护负担，却没有带来对应价值。BFF 的存在理由是"为特定端裁剪数据"；若无此需求，去除 BFF 直接对接 API Gateway 或后端服务。
- **token 在前端持久化但服务端从不吊销**：JWT 有效期内即使用户注销或权限被收回，前端仍持有有效 token 且服务端照常接受。无状态 JWT 的吊销须依赖短有效期 + refresh token rotation，或维护服务端黑名单；单纯依赖客户端删除 token 不构成安全吊销。
